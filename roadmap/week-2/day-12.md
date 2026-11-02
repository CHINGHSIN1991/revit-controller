# Day 12 · 查詢與檢視工具（`show_clash`）

## 今日目標

讓 AI 能「看到」碰撞：`list_clashes` 篩選與分頁、`get_clash_detail` 查看單筆細節，以及 `show_clash` 在 Revit 中建立或更新專用的 3D 視圖、套用 section box、替兩個元素上色並選取，最後**回傳截圖**。碰撞紀錄也要補上樓層與最近的軸線位置（`grid_ref`）。

`list_clashes`、`get_clash_detail` 只查 SQLite，可以在 Mac 上完成；`show_clash` 要在 Windows 上驗證。

## 你會學到

- 查詢工具的設計：篩選欄位、分頁、精簡回傳
- `View3D.CreateIsometric`、`SetSectionBox`、`ViewOrientation3D`
- `OverrideGraphicSettings` 與 `View.SetElementOverrides`，以及連結元素的上色限制
- `Selection.SetElementIds`、`ShowElements`，以及選取連結元素需要的 `CreateLinkReference`
- `Document.ExportImage` 與 `ImageExportOptions`
- 哪些動作需要 `Transaction`
- 用 `Grid.Curve` 找最近的軸線交點

---

## Step 1. `list_clashes`

參數（JSON schema 要嚴格，單位 mm）：

| 參數 | 型別 | 說明 |
| --- | --- | --- |
| `rule_id` | string? | |
| `status` | string[]? | `new`、`active`… 本週只會有 `new` |
| `level` | string? | 例如 `3F` |
| `category` | string? | 任一元素的類別符合即可 |
| `run_id` | string? | 預設最新一次執行 |
| `page` / `page_size` | int | `page_size` 上限 50（待調整） |

回傳只放摘要，細節用 `get_clash_detail` 查：

```jsonc
{ "total": 37, "page": 1, "items": [
  { "clash_id": "a1b2…", "rule_id": "mep-vs-struct-beam", "level": "3F", "grid_ref": "C-4",
    "categories": ["Ducts", "Structural Framing"], "volume_mm3": 12345, "status": "new" } ] }
```

篩選用 Day 8 開的欄位（`rule_id`、`status`、`level`）做 SQL `WHERE`；`category` 這類不常用的條件可以先撈出來再在記憶體中過濾。

## Step 2. `get_clash_detail`

回傳 SPEC §5.4 的完整紀錄，加上兩個元素的關鍵參數（族群、類型、系統、尺寸）。

| 做法 | 優點 | 缺點 |
| --- | --- | --- |
| 偵測時就存下參數快照 | 不需要 Revit 開著；可以在 Mac 上測 | 模型修改後會過時 |
| 查詢時即時讀 Revit | 永遠最新 | Revit 要開著，元素可能已被刪除 |

建議：偵測時存快照（Day 11 的 `clash.runBatch` 一起回傳），細節中標明快照時間。是否加上即時查詢（`element.params` RPC），等 Week 4 AI 判讀時看需求再決定。

參數值可能含有使用者輸入的文字，回傳時當作資料，不要拼進工具描述或系統提示（見 [01-architecture.md](../01-architecture.md) 安全邊界）。

## Step 3. 樓層與 `grid_ref`

**樓層**：取碰撞中心點的高程，找「高程小於等於該點的最高樓層」。用元素自己的樓層參數不可靠（管線的參考樓層可能和實際位置不同；而且兩個元素可能不同樓層）。

**軸線**：

```csharp
// src/RevitController.Addin/Clash/GridLocator.cs
internal static string? Locate(Document host, XYZ p)
{
    var grids = new FilteredElementCollector(host).OfClass(typeof(Grid)).Cast<Grid>();
    // 1. 只處理直線軸線：grid.Curve is Line
    // 2. 依方向分成兩組（例如接近 X 方向、接近 Y 方向）
    // 3. 各組中找出到 p 的水平距離最近的一條（curve.Project(p) 或自己算點到線距離）
    // 4. 回傳 "<組 1 名稱>-<組 2 名稱>"，例如 "C-4"
    return null;
}
```

- 弧形軸線、多段軸線先不處理，回傳 null。
- 軸線只從主模型取。如果軸線畫在連結的建築模型中，要另外處理（待決定）。
- 這兩個欄位在 Day 11 寫入時就算好，不要每次查詢時才算。

## Step 4. `show_clash`：建立專用 3D 視圖

一律使用名為 `RC_Clash` 的專用視圖，不要改使用者自己的視圖。

```csharp
// src/RevitController.Addin/Handlers/ShowClashHandler.cs（在 ExternalEvent handler 內）
using (var t = new Transaction(doc, "RC show clash"))
{
    t.Start();
    var view = FindView(doc, "RC_Clash");
    if (view == null)
    {
        view = View3D.CreateIsometric(doc, Get3DViewFamilyTypeId(doc));
        view.Name = "RC_Clash";                           // 只在新建時設定；名稱重複會丟例外
    }
    view.SetSectionBox(BoxAround(clashPoint, marginFt));  // 碰撞中心外擴一段距離
    view.IsSectionBoxActive = true;                       // 屬性名稱待查證
    ApplyOverrides(view, record);                         // Step 5
    t.Commit();
}
```

| API | 說明 |
| --- | --- |
| `View3D.CreateIsometric(doc, typeId)` | 需要一個 `ViewFamily.ThreeDimensional` 的 `ViewFamilyType` id（用 collector 找） |
| `SetSectionBox(BoundingBoxXYZ)` | 座標是內部座標；這就是 Day 10 決定同時存內部座標的原因 |
| `SetOrientation(ViewOrientation3D)` | 設定相機的 eye、up、forward。可以先用預設等角視角，之後再調 |
| 切換到該視圖 | `uidoc.ActiveView = view`，**不能**在 Transaction 中切換，且有使用時機限制（待查證）；或用 `RequestViewChange`（待查證） |

## Step 5. 上色與選取

```csharp
var ogs = new OverrideGraphicSettings();
// 設定投影線與表面填滿顏色；方法名稱在 2019 前後有變動（待查證）
// ogs.SetProjectionLineColor(new Color(255, 0, 0));
// ogs.SetSurfaceForegroundPatternColor(...); ogs.SetSurfaceForegroundPatternId(solidFillId);
view.SetElementOverrides(hostElementId, ogs);
```

| 情況 | 上色 | 選取 |
| --- | --- | --- |
| 主模型元素 | `SetElementOverrides` 可以直接用 | `uidoc.Selection.SetElementIds(ids)` |
| 連結元素 | **不能**對連結中的單一元素做覆寫，只能覆寫整個 link instance（替代方案待查證，例如視圖篩選器） | `new Reference(linkedElem).CreateLinkReference(linkInstance)`，再用 `Selection.SetReferences`（版本可用性待查證） |

連結元素無法單獨上色時，至少把其他元素調成半透明或灰色，讓兩個碰撞元素凸顯出來（作法待驗證）。

`uidoc.ShowElements(ids)` 會把畫面縮放到元素。截圖的範圍預期由 section box 與 `ZoomType` 決定，目前畫面的縮放是否會影響匯出結果（待查證）。

## Step 6. 匯出截圖

```csharp
var opt = new ImageExportOptions
{
    ExportRange = ExportRange.SetOfViews,
    FilePath = Path.Combine(snapshotDir, record.ClashId),   // 副檔名由 Revit 加上（待查證）
    HLRandWFViewsFileType = ImageFileType.PNG,
    ShadowViewsFileType = ImageFileType.PNG,
    ZoomType = ZoomFitType.FitToPage,
    PixelSize = 1200,
};
opt.SetViewsAndSheets(new List<ElementId> { view.Id });
doc.ExportImage(opt);   // 不需要 Transaction
```

- 圖片寫在 Sidecar 可以讀到的資料夾（例如 `%LocalAppData%\RevitController\snapshots\`），路徑存進 `snapshot_path`。
- 實際輸出檔名的規則（是否加上視圖名稱）待查證，匯出後用 `Directory.GetFiles` 找最新檔案比較保險。
- Sidecar 把 PNG 以 MCP 的 image content 回傳給 AI（C# SDK 的寫法待查證），同時回傳文字摘要。

## Step 7. Mock 與測試

- `view.showClash` 的 Mock 回應：回傳 `fixtures/mock/clash/sample.png` 的路徑。
- `list_clashes` 的篩選與分頁、`get_clash_detail` 的組裝，都在 Mac 上用 SQLite 寫單元測試。

---

## 驗收

- [ ] `list_clashes` 的每個篩選條件與分頁都有測試
- [ ] `get_clash_detail` 回傳兩個元素的參數快照、`level`、`grid_ref`
- [ ] 從 Mac 呼叫 `show_clash`，Revit 切換到 `RC_Clash` 視圖，section box 包住碰撞位置
- [ ] 主模型元素上色、選取正常；連結元素至少能被選取，上色限制已記錄
- [ ] AI 端拿到截圖，畫面中看得到兩個碰撞元素
- [ ] 連續對不同碰撞呼叫 `show_clash`，重用同一個視圖，不會產生大量視圖

## 常見問題

**Q：`ExportImage` 輸出的是空白或全黑的圖？**
可能是遠端桌面或 GPU 的問題（見 Day 1），或 section box 太小、位置錯誤。先在 Revit 中手動打開 `RC_Clash` 確認畫面。

**Q：切換 `ActiveView` 時丟例外？**
切換視圖有使用時機限制，常見原因是在 Transaction 中切換。先 Commit，再切換（詳細限制待查證）。

**Q：每次 `show_clash` 都會改模型，會不會讓使用者的「復原」清單變亂？**
會。修改視圖屬於模型修改，會出現在復原清單。把 Transaction 命名清楚（`RC show clash`），讓使用者看得懂。

## 學習筆記

（寫下今天的心得、卡關點、想問的問題）
