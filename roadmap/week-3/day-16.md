# Day 16 · 忽略規則

## 今日目標

實作 SPEC §5.2 中 `ignore` 欄位的四種規則：

| 名稱 | 意思 | 例子 |
| --- | --- | --- |
| `hosted` | 被寄宿元素與它的主體 | 門窗與牆 |
| `joined` | 已經用「接合幾何」接合的元素 | 柱與牆、梁與樓板 |
| `same_system` | 同一個 MEP 系統中的元素 | 同一條風管系統的管段與管件 |
| `{ "custom": … }` | 依參數條件忽略 | 「保溫層」類型的元素、標記為「暫不處理」的元素 |

今天結束時，在基準模型上跑同一組規則，開啟忽略規則後誤報明顯減少，**且沒有漏掉任何標註的真碰撞**。

## 你會學到

- `FamilyInstance.Host`、`JoinGeometryUtils.AreElementsJoined`、MEP `Connector` 的用法
- 這些 API 都只在單一文件內有效，跨連結時該怎麼處理
- 誤報（false positive）與漏報（false negative）的取捨
- 忽略規則放在流程中的哪一步、被忽略的碰撞要不要記錄

---

## Step 1. 忽略規則在流程中的位置

SPEC §5.3 的流程中，忽略規則在精算**之後**、分群**之前**。但為了效能，可以把便宜的判斷提前：

| 規則 | 成本 | 建議位置 |
| --- | --- | --- |
| `hosted` | 很低（讀一個屬性） | 初篩之後、精算之前 |
| `joined` | 低 | 初篩之後、精算之前 |
| `same_system` | 中（要走 connector） | 精算之後 |
| `custom` | 依條件 | 收集元素時能用參數篩選就先篩；否則精算之後 |

被忽略的組合**不要直接丟掉**。至少在 job 摘要中記錄「各規則忽略了幾筆」，方便 Day 21 驗收和使用者質疑時追查。是否寫成 `status = ignored` 的紀錄，要先決定：

- 寫入：可追溯，但資料量變大，而且會和使用者手動設為 `ignored` 混在一起。
- 不寫入：只記數量。建議 v1 用這個，加一個 `ignored_by_rule` 計數表。

## Step 2. 規則的資料結構（Core）

```csharp
// Core/Clash/IgnoreRules.cs
public abstract record IgnoreRule;
public sealed record HostedIgnore     : IgnoreRule;
public sealed record JoinedIgnore     : IgnoreRule;
public sealed record SameSystemIgnore : IgnoreRule;
public sealed record CustomIgnore(string Side, string Parameter, string Op, string Value) : IgnoreRule;
// Side: "a" | "b" | "either"；Op: "equals" | "contains" | "not_equals" ...

// JSON 解析放在 Sidecar（System.Text.Json），Core 只接收已解析的值，避免 add-in 多一個相依套件
public static class IgnoreRuleFactory
{
    public static IgnoreRule FromName(string name) => name switch
    {
        "hosted" => new HostedIgnore(), "joined" => new JoinedIgnore(), "same_system" => new SameSystemIgnore(),
        _ => throw new ArgumentException($"unknown ignore rule: {name}")
    };
}
```

- `record` 在 netstandard2.0 / net48 上需要設定 `LangVersion` 並補 `IsExternalInit` 型別（待查證 Day 2 的設定是否已處理）。

- 自訂規則的格式（SPEC 中只寫了 `{ "custom": "..." }`）今天要定下來。用結構化欄位，**不要讓使用者或 AI 寫任意運算式**，避免注入和難以驗證。
- 未知的規則名稱要回傳錯誤，不要默默略過。
- 這段在 Core，可以在 Mac 上測試解析。

## Step 3. `hosted`

```csharp
// Addin/Clash/IgnoreEvaluator.cs（骨架）
bool IsHosted(Element a, Element b) =>
    (a is FamilyInstance fa && fa.Host?.Id == b.Id) ||
    (b is FamilyInstance fb && fb.Host?.Id == a.Id);
```

- `Host` 只能指向同一個文件中的元素。門在建築模型、牆也在建築模型時有效；若兩者分屬主模型與連結，`Host` 可能指向連結本身或為 null（待查證）。
- 還有 `FamilyInstance.SuperComponent`（巢狀族群）與 `HostFace` 等相關屬性，是否要一起處理，以基準模型的實際誤報決定。

## Step 4. `joined`

```csharp
bool IsJoined(Document doc, Element a, Element b) =>
    JoinGeometryUtils.AreElementsJoined(doc, a, b);
```

- 只適用於**同一個文件**中的兩個元素。跨連結時，接合幾何本來就不存在，規則直接回傳 `false`（不忽略）。
- 已接合的元素，Revit 會把幾何切開，hard clash 交集可能本來就很小。實測看看這條規則實際影響多少筆。

## Step 5. `same_system`

判斷兩個 MEP 元素是否屬於同一個系統：

```csharp
bool InSameSystem(Element a, Element b)
{
    var sa = GetSystemIds(a);   // 從 ConnectorManager 取每個 Connector 的 MEPSystem
    var sb = GetSystemIds(b);
    return sa.Overlaps(sb);
}

HashSet<ElementId> GetSystemIds(Element e)
{
    ConnectorManager cm = e switch
    {
        MEPCurve c        => c.ConnectorManager,
        FamilyInstance fi => fi.MEPModel?.ConnectorManager,
        _ => null
    };
    // foreach (Connector c in cm.Connectors) if (c.MEPSystem != null) add c.MEPSystem.Id
}
```

| 情況 | 處理 |
| --- | --- |
| 管段（`MEPCurve`） | 直接有 `ConnectorManager`；也可以讀 `MEPSystem` 屬性（待查證各版本差異） |
| 管件、設備（`FamilyInstance`） | 從 `MEPModel.ConnectorManager` 取 |
| 未連接的 connector | `MEPSystem` 可能為 null，視為「不在系統中」 |
| 跨連結 | 兩邊的 `ElementId` 不在同一個文件，不能比較。直接回傳 `false` |

- 「同一個系統」不代表不能碰撞（例如同系統的兩條平行管線互相穿插仍是問題）。這條規則主要用來排除管段與相鄰管件的接觸，必要時加上「直接相連」的限制（`Connector.AllRefs`）。

## Step 6. 自訂規則

```csharp
bool MatchCustom(Element e, CustomIgnore r)
{
    Parameter p = e.LookupParameter(r.Parameter);   // 名稱可能重複或依語系不同（待查證）
    string v = p?.AsValueString() ?? p?.AsString();
    return r.Op switch { "equals" => v == r.Value, "contains" => v?.Contains(r.Value) == true, _ => false };
}
```

- `LookupParameter` 用名稱查詢，遇到同名參數只回傳第一個；共用參數最好用 GUID（`get_Parameter(Guid)`）。
- `AsValueString()` 會依顯示單位格式化，不適合比較數值。數值條件要用 `AsDouble()` 並轉換單位。

## Step 7. 在基準模型上量測

用 Day 8 的標註檔，比較開啟忽略規則前後：

| 指標 | 關閉忽略 | 開啟忽略 |
| --- | --- | --- |
| 回報總數 | | |
| 真碰撞（標註）被報出數 | | |
| 誤報數 | | |
| **漏報數** | | （必須是 0） |
| 各規則忽略的數量 | — | hosted / joined / same_system / custom |

只要有一筆標註的真碰撞被忽略，就要回頭檢查那條規則，**寧可保守**：讓使用者多看幾筆，比漏掉一個真問題好。

---

## 驗收

- [ ] 四種忽略規則都能從 `ClashRule.ignore` 解析；未知名稱回傳錯誤
- [ ] 跨連結的組合中，`hosted` / `joined` / `same_system` 不會誤判為忽略
- [ ] 基準模型上誤報明顯減少，Step 7 的表格已填寫
- [ ] 漏報數為 0
- [ ] Job 摘要中可以看到各規則忽略的數量
- [ ] 自訂規則的格式已寫進 SPEC §5.2（或在 PROGRESS.md 記錄待決）

## 常見問題

**Q：門窗和牆的碰撞還是被報出來？**
檢查元素是否在不同文件（建築模型是連結）；也可能門是巢狀族群，`Host` 在上層元素上。

**Q：`same_system` 把真的碰撞也忽略了？**
系統範圍太大（例如整棟的送風系統）。加上「直接相連」的條件，或只忽略管段與管件之間的組合。

**Q：AI 可以自己新增忽略規則嗎？**
可以透過 `create_clash_rule` 設定，但每一次都會寫入 audit log。Day 22 設計 AI 流程時，要決定 AI 是否需要人工確認才能加忽略規則。

## 學習筆記

（寫下今天的心得、卡關點、想問的問題）
