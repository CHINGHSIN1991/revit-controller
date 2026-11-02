# Day 29 · 修正策略設計

> 本文件依計畫推估，開始前請依 Day 21 與 Day 28 回顧結果修訂。

## 今日目標

決定 v1 要讓 AI 自動修正哪些碰撞、怎麼修，並定義 `propose_fix` 的輸出格式（`FixProposal`）。今天以設計為主，只寫 Core 裡的 DTO 與驗證邏輯，不呼叫任何會修改模型的 Revit API。

產出：`docs/fix-strategies.md`（修正策略文件）與 `FixProposal` 的 JSON schema。

## 你會學到

- MEP 管線在 Revit 中的幾何表示：`LocationCurve`、`MEPCurve`、接頭（fitting）
- `ElementTransformUtils.MoveElement` 與修改 `Offset` 類參數的差異
- 修改一個元素為什麼會牽動其他元素（接頭、hosted 元素、連接的管段）
- 怎麼把「無法在主模型修改」的情況設計成正式的輸出，而不是錯誤

---

## Step 1. 列出可以自動修正的類型

先從基準模型（Day 8）與 Day 23 的評估集中，挑出 `verdict = true_clash` 的碰撞，看它們最常見的修法。

| 類型 | 做法 | 典型情境 | 難度 | v1 |
| --- | --- | --- | --- | --- |
| `offset` | 整段管線或風管水平平移 | 風管貼著柱子 | 中 | ✅ |
| `elevation` | 調整高程（改 offset 參數或垂直移動） | 風管撞梁，往下降 | 中 | ✅ |
| `delete_duplicate` | 刪除重複元素中的一個 | Duplicate 類碰撞 | 低 | ✅ |
| `reroute` | 加彎頭繞過障礙物 | 管線撞梁但不能整段降 | 高 | 評估後決定 |
| `assign` | 不改模型，指派給負責專業 | 牽涉連結模型的元素 | 低 | ✅ |

`reroute` 要新增接頭並重新連接管段，牽涉的 API 與失敗情況最多。建議今天只寫設計，實作放到 Day 30 試做後再決定是否納入 v1。

## Step 2. 理解 MEP 幾何

| 概念 | 說明 |
| --- | --- |
| `MEPCurve` | 管、風管、電纜架等線性 MEP 元素的基底類別 |
| `LocationCurve` | 線性元素的定位線，`element.Location as LocationCurve` 取得 |
| Connector | 管段兩端的連接點，透過 `ConnectorManager` 取得（待查證各類別的取法） |
| Fitting | 彎頭、三通等接頭，是 `FamilyInstance`，透過 connector 與管段相連 |

移動一段管線時，兩端接的 fitting 可能跟著調整、可能斷開、也可能產生錯誤。實際行為依「移動多少」「是否只移動一段」而不同（待查證，Day 30 用 dry-run 實測）。

```csharp
// 讀取管段的定位線與兩端連接的元素（骨架）
if (elem is MEPCurve curve && curve.Location is LocationCurve lc)
{
    var line = lc.Curve;                                 // 內部單位：英尺
    var connected = curve.ConnectorManager.Connectors    // 待查證：列舉與 AllRefs 的用法
        .Cast<Connector>()
        .SelectMany(c => c.AllRefs.Cast<Connector>())
        .Select(r => r.Owner.Id)
        .Where(id => id != curve.Id);
}
```

## Step 3. 移動 vs 改參數

| 方式 | 寫法 | 適合 | 注意 |
| --- | --- | --- | --- |
| 幾何移動 | `ElementTransformUtils.MoveElement(doc, id, XYZ)` | 水平平移 | 會不會帶動相連元素需實測（待查證） |
| 改參數 | 設定高程類參數（例如 `Offset`） | 調整高程 | 參數名稱與 `BuiltInParameter` 依類別與版本不同（待查證） |

原則：**高程用參數、水平用移動**。參數修改比較符合使用者在 Revit 裡的操作習慣，也比較容易在 audit log 中記錄「改前 / 改後」的值。

## Step 4. 影響範圍

修改一個元素前，先算出它的「影響範圍」，寫進 `FixProposal`：

1. 直接修改的元素。
2. 透過 connector 相連的元素（fitting、其他管段）。
3. Hosted 在它上面的元素（例如吊架，待查證取得方式）。
4. 範圍內其他規則可能產生的新碰撞（Day 30 dry-run 時重算）。

如果影響範圍超過門檻（例如相連元素超過 N 個），方案標成 `needs_manual`，不自動執行。門檻先寫在設定檔，Day 31 依試用結果調整。

## Step 5. 連結模型的元素

SPEC §6：連結中的元素在主模型裡是唯讀的。碰撞的兩個元素：

| A | B | 方案 |
| --- | --- | --- |
| 主模型 | 連結 | 只能改 A；若 A 不適合改，改為 `assign` 給 B 的專業 |
| 連結 | 連結 | 一律 `assign`，註明應由哪個專業的模型負責人修改 |
| 主模型 | 主模型 | 依策略選擇要改哪一個 |

`assign` 不是失敗，而是正式的方案類型。它會寫入 `update_clash` 的 `assignee`，之後可以透過 BCF 或 ACC 連接器送給對方。

## Step 6. `FixProposal` 的資料模型

放在 `src/RevitController.Core/Models/FixProposal.cs`，單位一律 mm（SPEC §5.2）：

```csharp
public enum FixKind { Offset, Elevation, DeleteDuplicate, Reroute, Assign }

public sealed class FixTarget
{
    public string? LinkInstanceUid { get; set; }   // 非 null 代表連結元素，不可修改
    public string UniqueId { get; set; } = "";
    public string Action { get; set; } = "";       // "move" | "set_param" | "delete"
    public double[]? MoveMm { get; set; }          // [dx, dy, dz]
    public string? ParamName { get; set; }
    public double? OldValueMm { get; set; }
    public double? NewValueMm { get; set; }
}

public sealed class FixProposal
{
    public string ProposalId { get; set; } = "";
    public string ClashId { get; set; } = "";
    public FixKind Kind { get; set; }
    public List<FixTarget> Targets { get; set; } = new();
    public List<string> AffectedUniqueIds { get; set; } = new();  // 影響範圍
    public string ExpectedEffect { get; set; } = "";               // 給人看的說明
    public string? AssignTo { get; set; }                          // Kind == Assign 時
    public string Status { get; set; } = "draft";
    // draft | previewed | approved | applying | applied | failed | rejected | expired
}
```

加一個 Core 層的驗證函式，並寫單元測試（Mac 上可跑）：

```csharp
public static IEnumerable<string> Validate(FixProposal p)
{
    if (p.Kind != FixKind.Assign && p.Targets.Count == 0) yield return "no_targets";
    if (p.Targets.Any(t => t.LinkInstanceUid != null)) yield return "linked_element_readonly";
    if (p.Kind == FixKind.Assign && string.IsNullOrEmpty(p.AssignTo)) yield return "assign_to_required";
}
```

## Step 7. 寫修正策略文件

`docs/fix-strategies.md` 至少包含：

- 每種 `FixKind` 的適用條件與不適用條件（例如：有坡度的排水管不自動調高程）
- 預設的移動量規則（例如：至少讓出規則 `tolerance_mm` 再加一段餘裕）
- 方案的排序：優先動 MEP、不動結構；優先動小管、不動主幹（依專案慣例，待與使用者確認）
- 什麼情況一律 `needs_manual`

這份文件也是 AI 的判斷依據，Day 30 會把重點摘進 `propose_fix` 的工具描述。

---

## 驗收

- [ ] 修正類型表已定案，`reroute` 是否納入 v1 有明確結論或待 Day 30 實測
- [ ] `FixProposal` 已放進 Core，JSON schema 可由 sidecar 輸出
- [ ] `Validate` 有單元測試，涵蓋連結元素、空目標、`assign` 缺對象
- [ ] `docs/fix-strategies.md` 已寫好適用條件與 `needs_manual` 規則
- [ ] 能說明 `MoveElement` 與改 `Offset` 參數的差異與各自適用情境
- [ ] 能說明為什麼牽涉連結元素時，方案只能是 `assign`

## 常見問題

**Q：AI 能不能直接決定移動量？**
可以提出，但移動量要由確定性的程式驗證（SPEC §1：幾何一律由確定性的程式計算）。AI 負責選策略與說明理由，數值由 add-in 依幾何計算或檢查。

**Q：有坡度的管線（例如排水管）可以整段改高程嗎？**
不建議自動處理。改高程可能破壞坡度與上下游連接。v1 一律標 `needs_manual`。

**Q：`Offset` 參數在各版本叫什麼？**
不同類別、不同版本的參數名稱與 `BuiltInParameter` 可能不同（待查證）。在 add-in 內集中管理對照表，用 `#if REVITxxxx_OR_GREATER` 處理差異。

## 學習筆記

（寫下今天的心得、卡關點、想問的問題）
