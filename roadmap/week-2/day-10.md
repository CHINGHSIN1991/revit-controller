# Day 10 · 連結模型與座標轉換

## 今日目標

讓主模型與連結模型的元素可以在**同一個座標系統**中比較：對連結中的 solid 套用 `RevitLinkInstance.GetTotalTransform()`，處理同一個連結放置多次的情況，未載入或隱藏的連結要回報給使用者，不得略過。最後在偵測時就算好每個元素的 IFC GUID。

## 你會學到

- `Transform` 的組成（`Origin`、`BasisX/Y/Z`）與 `SolidUtils.CreateTransformed`
- 內部座標、專案基準點、測量點、共用座標的差異，以及回報時用哪一個
- `RevitLinkInstance.GetLinkDocument()` 何時回傳 null
- 連結中的元素在主模型裡是唯讀的
- Revit `UniqueId` 與 IFC GUID 的關係

---

## Step 1. 列出連結 instance

```csharp
// src/RevitController.Addin/Clash/LinkResolver.cs
internal static IEnumerable<SourceScope> ResolveLinks(Document host, string? nameFilter, List<LinkIssue> issues)
{
    var instances = new FilteredElementCollector(host)
        .OfClass(typeof(RevitLinkInstance))
        .Cast<RevitLinkInstance>();

    foreach (var inst in instances)
    {
        var linkDoc = inst.GetLinkDocument();
        if (linkDoc == null)
        {
            issues.Add(new LinkIssue(inst.UniqueId, inst.Name, "LinkNotLoaded"));
            continue;    // 不計算，但一定要回報
        }
        // nameFilter 比對（規則見 Day 9 Step 2）
        yield return new SourceScope { Doc = linkDoc, LinkInstance = inst, ToHost = inst.GetTotalTransform() };
    }
}
```

`GetLinkDocument()` 回傳 null 的常見情況（待查證完整清單）：

| 情況 | 處理 |
| --- | --- |
| 連結已卸載（Unloaded） | 回報 `LinkNotLoaded` |
| 找不到連結檔案 | 回報 `LinkNotLoaded`，附上路徑 |
| 巢狀連結設為 Overlay，從上層模型看不到 | 回報，說明是巢狀連結（行為待查證） |

連結狀態也可以從 `RevitLinkType` 讀（例如 `GetLinkedFileStatus()`，待查證），Day 6 的 `get_model_overview` 應該已經有讀，沿用同一段。

**隱藏的連結**：連結 instance 在某個視圖中被隱藏，不影響模型幾何，碰撞仍然可以計算。但使用者可能以為它「不在」，所以要在結果摘要中列出「在目前視圖中被隱藏的連結」。用 `Element.IsHidden(View)` 判斷（待查證是否涵蓋類別隱藏、篩選器隱藏等情況）。

## Step 2. `LinkNotLoaded` 怎麼回報

SPEC §3.3 把 `LinkNotLoaded` 列為正式錯誤。但一條 `source: any` 的規則，不應該因為一個連結沒載入就整個失敗。區分兩種情況：

| 規則 | 連結狀態 | 結果 |
| --- | --- | --- |
| `source: link:STR` | STR 沒載入 | 整條規則失敗，錯誤碼 `LinkNotLoaded` |
| `source: any` | 其中一個沒載入 | 規則照跑，job 結果的 `warnings` 列出沒載入的連結 |

AI 拿到 warnings 時，應該告訴使用者「結果不完整」。把這句寫進 `run_clash_test` 的工具描述。

## Step 3. 套用 Transform

```csharp
internal static Solid ToHost(Solid s, SourceScope scope)
    => scope.LinkInstance == null ? s : SolidUtils.CreateTransformed(s, scope.ToHost);
```

| `Transform` 成員 | 意義 |
| --- | --- |
| `Origin` | 連結原點在主模型中的位置 |
| `BasisX/Y/Z` | 連結的三個軸在主模型中的方向（旋轉） |
| `OfPoint(XYZ)` | 把點轉到主模型座標 |
| `Inverse` | 反向轉換，例如把主模型的點轉回連結座標 |

重點：

- `GetTotalTransform()` 已包含連結本身的放置與基準點設定；另一個 `GetTransform()` 的差異請查 API 文件（待查證）。
- 轉換後的 solid 是新的物件，可以快取，key 要包含 link instance（同一個元素在兩個 instance 中是兩份）。
- Bounding box 也要轉換：`BoundingBoxXYZ` 本身有 `Transform` 屬性，直接用 `Min`/`Max` 會錯。最穩的做法是把八個角點都 `OfPoint` 後重算 min/max（Day 11 用）。

## Step 4. 同一個連結放置多次

例如同一棟標準層的結構模型放了兩次。SPEC §6：每個 instance 分開計算。

- `ResolveLinks` 已經是依 instance 迭代，所以同一份 `linkDoc` 會出現兩次，各帶不同的 `ToHost`。
- 元素參照用 `(link_instance_uid, unique_id)`，`link_instance_uid` 就是 `RevitLinkInstance.UniqueId`。
- 元素收集（Day 9）可以依 `linkDoc` 快取，solid 的轉換則依 instance 各做一次。

## Step 5. 三種組合

| 組合 | A 的來源 | B 的來源 | 精算方式（Day 11） |
| --- | --- | --- | --- |
| 主模型對連結 | host | link | 轉換後用 `BooleanOperationsUtils` |
| 連結對連結 | link 1 | link 2 | 兩邊都轉到主模型座標 |
| 同一個連結內部 | link 1 | link 1（同 instance） | 可以在連結文件內用 `ElementIntersectsSolidFilter`，不用轉換（待驗證） |

## Step 6. 座標系統：回報時用哪一個

| 座標系統 | 說明 | 用途 |
| --- | --- | --- |
| 內部座標（Internal Origin） | Revit API 回傳的座標都是這個 | 計算 |
| 專案基準點（Project Base Point） | 專案的參考原點 | 使用者在 Revit 中看到的座標 |
| 測量點（Survey Point） | 對應實際地理位置 | 對外交換 |
| 共用座標（Shared Coordinates） | 多個模型共同的座標系 | 與 Navisworks、BCF 對齊 |

SPEC §5.4 說 `point` 是「mm，專案座標」。**今天要決定「專案座標」到底指哪一個**，並更新 SPEC。建議：

- Result Store 內部同時存「內部座標」（用來回到 Revit 定位，Day 12 用）。
- 對外（報告、BCF）用共用座標，才能和 Navisworks 對齊。共用座標的 transform 可以從 `doc.ActiveProjectLocation` 取得（API 名稱待查證）。

## Step 7. IFC GUID

`ifc_guid` 在偵測時就算好（SPEC §5.4），之後匯出 BCF 不需要再查 Revit。

```csharp
// src/RevitController.Addin/Clash/IfcGuid.cs
internal static string Get(Document doc, ElementId id)
{
    // 方式 1：ExportUtils.GetExportId(doc, id) 取得 Guid，再壓縮成 IFC 的 22 字元 base64（待查證）
    // 方式 2：讀元素的 IFC GUID 參數（若模型曾匯出過 IFC 並存回參數，待查證參數名稱）
    throw new NotImplementedException();
}
```

這是 SPEC §11 的待驗證項目之一。驗證方式：用 Revit 的 IFC exporter 匯出基準模型，挑 5 個元素，比對 IFC 檔中的 `GlobalId` 與你算出來的值。

連結中的元素要用**連結文件**的 `doc` 計算，不是主模型。

## Step 8. 連結元素是唯讀的

連結中的元素不能在主模型中修改，也不能直接用 `ElementId` 選取（Day 12 會用 `Reference.CreateLinkReference`）。這件事現在先記在 `ClashElementRef` 上：`link_instance_uid != null` 代表唯讀，Week 5 的修正建議要註明應由哪個專業的模型負責人修改（SPEC §6）。

---

## 驗收

- [ ] 基準模型中每個連結 instance 都被列出，放置兩次的連結出現兩次
- [ ] 卸載一個連結後執行，回報 `LinkNotLoaded` 或 warning（依 Step 2 的規則）
- [ ] 挑一個連結元素，轉換後的 bounding box 中心與 Revit 畫面中的位置一致
- [ ] SPEC §5.4 的「專案座標」定義已更新
- [ ] 至少 5 個元素的 IFC GUID 與 IFC exporter 的結果一致（SPEC §11 打勾或記錄差異）

## 常見問題

**Q：連結的 Transform 看起來都是 Identity？**
如果連結是用「原點對原點」放置且沒有移動，Transform 就會是 Identity。刻意把一個連結移動或旋轉，才測得出轉換是否正確。

**Q：轉換後的 solid 位置還是錯的？**
檢查是否重複套用：`GetInstanceGeometry()` 已套用族群 instance 的 transform，連結 transform 只要再套一次。也確認沒有誤用 `GetTransform()`。

**Q：IFC GUID 和 exporter 對不上？**
先記錄差異，不要卡住。BCF 是 Week 3 的事，這裡的結果會決定 Day 19 的做法。

## 學習筆記

（寫下今天的心得、卡關點、想問的問題）
