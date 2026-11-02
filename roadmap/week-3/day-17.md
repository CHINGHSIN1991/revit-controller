# Day 17 · 分群

## 今日目標

把大量碰撞收斂成少量、可以一次處理的群組（SPEC §5.3 第 5 步）：

1. **依共同元素**：同一根管穿過五根梁，是一個問題，不是五個。
2. **依空間鄰近、樓層與軸線**：同一個區域內的一堆小碰撞放在一起看。
3. `list_clashes` 支援依群組查詢，並提供群組摘要（數量、代表碰撞、代表截圖）。

今天的演算法全部放在 Core，**可以在 Mac 上用 Mock 資料寫測試**，不需要 Revit。

## 你會學到

- 圖的連通元件與 union-find（disjoint set）
- 空間分群：網格雜湊（grid hashing）與 DBSCAN 的概念
- 群組 ID 怎麼在重新執行後保持穩定，以及它的極限

---

## Step 1. 先定義「群組」

| 欄位 | 說明 |
| --- | --- |
| `group_id` | 穩定 ID（見 Step 4） |
| `method` | `shared_element` 或 `spatial` |
| `clash_ids` | 成員 |
| `anchor` | 代表元素（共同元素分群）或代表位置（空間分群） |
| `level` / `grid_ref` | 樓層與軸線範圍 |
| `representative_clash_id` | 用來截圖的那一筆 |

一筆碰撞只屬於一個群組（`ClashRecord.group_id` 是單一值）。兩種方法的順序：**先用共同元素分群，剩下沒有成群的（單獨一筆）再做空間分群。**

## Step 2. 共同元素分群：union-find

把每個元素當作圖的節點，每一筆碰撞是一條邊，連通元件就是群組。

```csharp
// Core/Clash/Grouping/UnionFind.cs
public sealed class UnionFind<T> where T : notnull
{
    private readonly Dictionary<T, T> _parent = new();
    public T Find(T x)
    {
        if (!_parent.TryGetValue(x, out var p)) { _parent[x] = x; return x; }
        if (!p.Equals(x)) _parent[x] = Find(p);      // 路徑壓縮
        return _parent[x];
    }
    public void Union(T a, T b) => _parent[Find(a)] = Find(b);
}

// 使用
foreach (var c in clashes) uf.Union(c.Elements[0].Key, c.Elements[1].Key);
var groups = clashes.GroupBy(c => uf.Find(c.Elements[0].Key));
```

要注意「連通過頭」：一根很長的主管碰到整層的梁，再透過這些梁連到其他管線，最後整層變成一個群組。對策：

| 對策 | 說明 |
| --- | --- |
| 只用一側的元素做連結 | 例如只以 A 組（MEP）元素為中心：同一根管的所有碰撞成一群，梁不當作橋樑 |
| 限制群組大小 | 超過上限就拆回空間分群 |
| 限制在同一樓層 | 不同樓層的碰撞不併在一起 |

建議 v1 先用「只以 A 組元素為中心」，最直觀，也最容易對 AI 和使用者解釋（「這根管有 5 處碰撞」）。

## Step 3. 空間分群：網格雜湊

把碰撞點（`point`，mm）放進固定大小的立方格，同一格與相鄰格的點併成一群：

```csharp
// Core/Clash/Grouping/SpatialGrouper.cs（骨架）
(long, long, long) Cell(double[] p, double size) =>
    ((long)Math.Floor(p[0] / size), (long)Math.Floor(p[1] / size), (long)Math.Floor(p[2] / size));

// 1. 每個點放進 Dictionary<Cell, List<Clash>>
// 2. 對每個點，檢查自己與 26 個相鄰格中距離 < radius 的點，用 union-find 合併
// 3. 合併時加上條件：同一樓層（level 相同）
```

| 方法 | 優點 | 缺點 |
| --- | --- | --- |
| 網格雜湊 + 距離閾值 | 實作簡單、O(n) 左右 | 結果受格子大小影響 |
| DBSCAN | 能處理形狀不規則的群集、會標出雜訊點 | 要調兩個參數（eps、minPts），解釋起來比較難 |

v1 用網格雜湊，半徑預設例如 1000 mm（可設定）。DBSCAN 的概念了解即可：它其實就是「距離 < eps 的點互相連通，且核心點至少有 minPts 個鄰居」，網格雜湊是它常用的加速結構。

## Step 4. 群組 ID 的穩定性

Day 18 的重跑比對靠 `clash_id`，群組也需要一個重跑後還能對應的 ID，否則使用者對群組的指派會消失。

| 方法 | group_id 的來源 | 失效情況 |
| --- | --- | --- |
| 共同元素 | `hash(rule_id, "shared", anchor_element_key)` | 錨點元素被刪除重建（UniqueId 改變） |
| 空間 | `hash(rule_id, "spatial", level, 最小的成員 clash_id)` | 最小成員消失時，ID 會換 |
| 空間（替代） | `hash(rule_id, "spatial", level, 網格座標)` | 碰撞點跨格移動時會換 |

選一種並寫清楚**它在什麼情況下會失效**。建議：

- 共同元素群組用錨點元素，這是最直觀也最穩定的。
- 空間群組用「最小成員 clash_id」，並在 Day 18 的重跑比對中，若新舊群組的成員重疊超過一半，就沿用舊的 `group_id`（待 Day 18 實作時決定是否需要）。

```csharp
// 單元測試（Mac 可跑）
[Fact] public void SameInput_DifferentOrder_SameGroupIds() { /* 打亂輸入順序，group_id 不變 */ }
[Fact] public void LongPipe_ThroughFiveBeams_IsOneGroup() { }
[Fact] public void DifferentLevels_NotMerged() { }
```

## Step 5. 寫入 Result Store 與 `list_clashes`

SQLite 新增群組表（Day 18 會一起加狀態歷史表，可以合併在同一個 migration）：

```sql
CREATE TABLE clash_groups (
  group_id TEXT PRIMARY KEY,
  rule_id  TEXT NOT NULL,
  method   TEXT NOT NULL,          -- shared_element | spatial
  anchor   TEXT,
  level    TEXT,
  representative_clash_id TEXT
);
-- clashes.group_id 已在 Day 8 的 schema 中（若沒有就新增欄位）
```

`list_clashes` 新增兩種用法：

| 參數 | 回傳 |
| --- | --- |
| `group_by: "group"` | 群組摘要清單：`group_id`、數量、樓層、代表碰撞、狀態分布 |
| `filter.group_id` | 該群組的碰撞清單（分頁） |

- 群組摘要要精簡（SPEC §7 設計原則）。AI 先看群組，再挑群組看細節，可以大幅減少 token。
- 代表截圖：先用 `representative_clash_id` 呼叫 `show_clash` 的同一段邏輯。是否要一張「涵蓋整群」的截圖（section box 包住所有成員），可以留到 M3 視 AI 判讀需要再做。

---

## 驗收

- [ ] Union-find 與空間分群的單元測試在 Mac 上通過（含輸入順序打亂後 ID 不變）
- [ ] 基準模型上，群組數明顯少於碰撞數（記錄兩個數字）
- [ ] 人工抽查 3 個群組，分在一起的碰撞確實是同一個問題
- [ ] 沒有「整層變成一個群組」的情況
- [ ] `list_clashes` 可以回傳群組摘要，也可以依 `group_id` 篩選
- [ ] `group_id` 的產生規則與失效情況寫在文件中

## 常見問題

**Q：群組太大，AI 看一張截圖判斷不了？**
限制群組大小，或讓 AI 對大群組逐筆查看。群組是幫助瀏覽的工具，不是判讀的單位。

**Q：一筆碰撞可以同時屬於「共同元素群組」和「空間群組」嗎？**
v1 不行，`group_id` 是單一值。若之後需要多種檢視方式，再把群組改成多對多關係表。

**Q：使用者手動調整的群組要保留嗎？**
SPEC §7 的 `update_clash` 可以更新群組。手動指定的群組要標記為 `manual`，重跑時不要被自動分群覆蓋。

## 學習筆記

（寫下今天的心得、卡關點、想問的問題）
