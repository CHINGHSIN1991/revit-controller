# Day 15 · Clearance 與 Duplicate

## 今日目標

在 hard clash 之外，加上 SPEC §5.1 的另外兩種碰撞類型：

- **Clearance**：兩個元素的間距小於規則設定的淨空（`tolerance_mm`）。
- **Duplicate**：同類別、同族群類型、幾何幾乎重疊的元素。

今天結束時，`run_clash_test` 可以執行 `type` 為 `hard`、`clearance`、`duplicate` 的規則，結果都寫進 Result Store。

## 你會學到

- Revit 內部單位（英尺）與對外 mm 的轉換點應該放在哪裡
- Clearance 為什麼是「擴大 bbox 初篩 + 最短距離精算」兩段式
- Revit API 沒有直接的 solid 對 solid 最短距離，要自己近似，以及精度與效能的取捨
- Duplicate 的判斷標準，以及 A 組與 B 組相同時如何避免重複計算

---

## Step 1. 單位：只在邊界轉換一次

規則從 MCP 進來時是 mm（SPEC §5.2），Revit 內部是英尺。原則：**Core 與 Sidecar 全部用 mm，只有 Add-in 在呼叫 Revit API 前後轉換。**

Core 不能引用 RevitAPI，所以在 Core 放一個純常數；Add-in 端優先用 Revit 的 API：

```csharp
// Core/Units/Mm.cs（Day 9 建立，此處擴充）
public static class Units
{
    public const double MmPerFoot = 304.8;
    public static double MmToFeet(double mm) => mm / MmPerFoot;
    public static double FeetToMm(double ft) => ft * MmPerFoot;
}

// Addin 端
double tolFt = UnitUtils.ConvertToInternalUnits(rule.ToleranceMm, UnitTypeId.Millimeters);
```

- `UnitTypeId` 是 `ForgeTypeId` 世代的 API。Revit 2022 是否已可用、舊的 `DisplayUnitType` 何時移除（待查證，見 SPEC §11）。若 2022 不支援，就在 adapter 內用 `#if` 分流，或直接用 Core 的常數。
- 回傳給 AI 的距離、體積、座標全部換回 mm（`volume_mm3`、`point`）。

## Step 2. 擴充 ClashRule 與計算流程

`ClashRule.type` 已在 Day 8 定義成字串或 enum，今天讓計算流程依 type 分派：

```csharp
// Addin/Clash/ClashEngine.cs（骨架）
IClashDetector detector = rule.Type switch
{
    ClashType.Hard      => new HardDetector(),       // Day 11
    ClashType.Clearance => new ClearanceDetector(),
    ClashType.Duplicate => new DuplicateDetector(),
    _ => throw new NotSupportedException()
};
foreach (var batch in Batch(candidatePairs, batchSize)) // 分批，沿用 Day 11 的 job 機制；net48 沒有 LINQ Chunk，自己寫
    results.AddRange(detector.Detect(batch, ctx));
```

`ClashRecord` 對不同類型的欄位意義：

| type | `volume_mm3` | 額外資訊 |
| --- | --- | --- |
| hard | 交集體積 | — |
| clearance | null | 最短距離 `distance_mm`（新增欄位） |
| duplicate | null 或重疊體積 | 位置差、方向差 |

新增 `distance_mm` 要同步改 Day 8 的 SQLite migration 與 Mock fixture。

## Step 3. Clearance：擴大 bbox 初篩

先把 A 組每個元素的 bounding box（已套用連結 transform）**每個方向擴大淨空距離**，再和 B 組做 bbox 相交：

```csharp
Outline Expand(BoundingBoxXYZ bb, double d) =>
    new Outline(bb.Min - new XYZ(d, d, d), bb.Max + new XYZ(d, d, d));

var filter = new BoundingBoxIntersectsFilter(Expand(bbA, tolFt));
```

- 連結元素不能直接用 `BoundingBoxIntersectsFilter` 在主模型裡查，沿用 Day 11 自己算 bbox 相交的做法。
- 初篩只排除「一定不夠近」的組合，不能決定結果。

## Step 4. Clearance：計算兩個 solid 的最短距離

這是今天最難的部分。Revit API 有 `Face.Project(XYZ)`、`Curve.Project(XYZ)`、`Face.Triangulate()` 等基礎工具，但**沒有直接回傳兩個 solid 最短距離的方法**（待查證是否有新版 API 可用）。

| 方法 | 做法 | 精度 | 效能 |
| --- | --- | --- | --- |
| A. 頂點投影 | 取 A 的所有邊端點與 tessellate 點，投影到 B 的每個 face，取最小值；再反向一次 | 平面對平面時可能高估（漏掉面中央最近點） | 快 |
| B. Mesh 距離 | 兩邊 `Face.Triangulate()` 成三角網格，計算三角形對三角形最短距離 | 取決於 tessellation 細緻度 | 慢，但可以搬到背景執行緒 |
| C. 先判斷 hard | 若兩者已相交，距離為 0，直接當作 clearance 違規 | 精確 | 快 |

建議：**先做 C + A**，把 A 的結果當作上界；若上界與淨空的差距在一個小範圍內，再用 B 精算。

```csharp
// Addin/Clash/SolidDistance.cs（骨架）
public static double ApproxMinDistance(Solid a, Solid b)
{
    double best = double.MaxValue;
    foreach (XYZ p in SamplePoints(a))           // 邊端點 + edge.Tessellate()
        foreach (Face f in b.Faces)
        {
            IntersectionResult r = f.Project(p); // 投影不到 face 範圍內時回傳 null（待查證）
            if (r != null) best = Math.Min(best, r.Distance);
        }
    // 反向再算一次 b → a，並考慮 edge 對 edge
    return best;
}
```

- 精度要求先定為：**寧可多報，不可漏報**。把 A 方法的誤差方向寫清楚，必要時把容差放寬一點。
- `Face.Project` 對曲面（圓管）的行為與 `Distance` 的單位（應為英尺）要實測（待查證）。
- 每一筆的距離計算都在 UI thread 上執行，要控制批次大小，避免 Revit 凍結。

## Step 5. Duplicate

判斷條件（全部成立才算 duplicate）：

| 條件 | 判斷方式 |
| --- | --- |
| 同類別 | `Category.Id` 相同 |
| 同族群類型 | `GetTypeId()` 相同 |
| 位置接近 | `LocationPoint.Point` 或 `LocationCurve.Curve` 端點距離小於容差（預設例如 1 mm，可設定） |
| 方向相同 | `LocationPoint.Rotation` 或曲線方向差小於角度容差 |
| 幾何重疊 | 選用：bbox 幾乎相同，或交集體積 / 較小體積 > 閾值 |

Duplicate 規則的 `set_a` 和 `set_b` 通常是同一組元素，要注意：

```csharp
// 只比對 i < j，並且排除自己跟自己
for (int i = 0; i < items.Count; i++)
    for (int j = i + 1; j < items.Count; j++) { /* 先用空間雜湊縮小範圍，再比對 */ }
```

- 兩兩比對是 O(n²)，數萬個元素時一定要先用 bbox 或網格雜湊（Day 17 會再用到）縮小範圍。
- 跨連結的 duplicate（同一根柱子在建築與結構模型各畫一次）也很常見，但兩邊的族群類型不同。是否列入 v1 要先決定，建議先只做同一文件內，跨連結的寫進 PROGRESS.md 技術債。

## Step 6. 穩定 ID 要先排序

`clash_id = hash(rule_id, ref_a, ref_b)`（SPEC §5.4），元素以 `(link_instance_uid, unique_id)` 識別。Duplicate 與「A、B 組有重疊」的規則中，同一對元素可能以 `(a, b)` 或 `(b, a)` 出現，**要先排序再雜湊**：

```csharp
// Core/Models/ClashId.cs（Day 8 建立，此處調整）
public static string Compute(string ruleId, ElementRef a, ElementRef b)
{
    var (first, second) = string.CompareOrdinal(a.Key, b.Key) <= 0 ? (a, b) : (b, a);
    return Sha256Hex($"{ruleId}|{first.Key}|{second.Key}");   // Key = $"{link_instance_uid ?? ""}/{unique_id}"
}
```

這段在 Core，可以在 Mac 上寫單元測試：`Compute(r, a, b) == Compute(r, b, a)`。

---

## 驗收

- [ ] 基準模型上至少一條 clearance 規則能執行，`distance_mm` 合理（人工抽查 3 筆）
- [ ] 已知相交的元素在 clearance 規則中被報出（距離為 0）
- [ ] 基準模型中標註的重複元素都被 duplicate 規則找到
- [ ] Duplicate 結果中沒有「自己跟自己」或同一對元素出現兩次
- [ ] `ClashId.Compute` 對交換順序的輸入回傳相同結果（Core 單元測試）
- [ ] 所有對外欄位都是 mm；程式中只有 Add-in 邊界做單位轉換
- [ ] 最短距離方法的誤差方向（高估或低估）寫在文件中

## 常見問題

**Q：clearance 規則跑起來比 hard 慢很多？**
擴大 bbox 後候選組合會變多，而且距離計算比相交判斷貴。先檢查初篩是否有效（記錄候選數），再考慮把 mesh 距離搬到背景執行緒（SPEC §5.3）。

**Q：圓管對梁的距離明顯不對？**
頂點投影對曲面的取樣點太少。增加 edge tessellation 的點數，或改用 mesh 距離。

**Q：Duplicate 找到大量「重複」的牆？**
可能是不同樓層的相同牆、或疊在一起的裝修層。加上樓層與高度的條件，並和基準模型的標註比對。

**Q：tolerance_mm 是 0 的 clearance 規則和 hard 有什麼差別？**
距離為 0（接觸）在 clearance 會報出，在 hard 可能因交集體積為 0 不報。兩者的定義寫進規則說明，避免使用者混淆。

## 學習筆記

（寫下今天的心得、卡關點、想問的問題）
