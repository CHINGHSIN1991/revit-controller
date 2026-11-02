# Day 19 · BCF 2.1 結構與視角轉換

## 今日目標

研讀 BCF 2.1 規格，在 Sidecar 中寫好 `ClashRecord` → BCF 的資料轉換：

- 每筆碰撞（或每個群組，見 Step 1）→ 一個 **Topic**（`markup.bcf`）
- 視角 → **Viewpoint**（`viewpoint.bcfv`），含相機與元素選取
- 截圖 → `snapshot.png`

今天只做「轉換邏輯 + 單元測試」，打包成 `.bcfzip` 與外部工具驗證留到 Day 20。轉換全部在 Sidecar，**不呼叫 Revit**：`ifc_guid`、`viewpoint`、`snapshot_path` 都已經在 `ClashRecord` 中（SPEC §5.4）。

> 本文件中 BCF 的檔名、XML 元素名稱、欄位與單位，是依照對 BCF 2.1 的一般認識撰寫的。**所有細節（待查證，以 buildingSMART BCF-XML 規格為準）**，開始實作前請先下載官方 2.1 的 XSD 與範例檔。

## 你會學到

- BCF 是什麼：用 ZIP 包裝的一組 XML，用來在不同 BIM 工具之間交換「問題」
- BCF 2.1 的檔案結構與主要 XML 元素
- 相機的表示方式（透視 / 正交），以及 Revit `ViewOrientation3D` 怎麼轉過去
- 座標系與單位：Revit 內部座標、共用座標、IFC 座標，以及為什麼這是最容易出錯的地方
- BCF 2.1 與 3.0 的主要差異

---

## Step 1. 先決定：一個 Topic 對應一筆碰撞還是一個群組

| 選擇 | 優點 | 缺點 |
| --- | --- | --- |
| 一筆碰撞 = 一個 Topic | 對應簡單，`external_refs` 一對一 | 數量多，外部工具清單很長 |
| 一個群組 = 一個 Topic | 數量少，符合「一個問題」的概念 | 重跑後群組成員變動，Topic 的意義會跟著變 |

建議：**預設一筆一個 Topic**，提供 `options.per_group = true` 讓使用者選擇。`external_refs["bcf21"]` 存 Topic GUID（Day 20 用來重複匯出時沿用）。

## Step 2. BCF 2.1 的檔案結構

以下為一般認識的結構（待查證，以 buildingSMART BCF-XML 規格為準）：

```
export.bcfzip                 ← 其實就是 ZIP
├── bcf.version               ← 宣告版本，例如 VersionId="2.1"
├── project.bcfp              ← 選用：專案資訊
└── <Topic GUID>/             ← 每個 Topic 一個資料夾，以 GUID 命名
    ├── markup.bcf            ← Topic 資訊、留言、viewpoint 清單
    ├── viewpoint.bcfv        ← 相機、元素選取、可見性
    └── snapshot.png          ← 截圖
```

一個 Topic 可以有多個 viewpoint（多個 `.bcfv` 與對應截圖）。v1 每個 Topic 一個就好。

## Step 3. 對應表：ClashRecord → markup.bcf

以下元素名稱均待查證（以 buildingSMART BCF-XML 規格為準）：

| ClashRecord | BCF（markup.bcf） | 備註 |
| --- | --- | --- |
| （新產生或 `external_refs["bcf21"]`） | `Topic/@Guid` | 重複匯出時沿用 |
| `rule_id` + 兩個元素類別 | `Topic/Title` | 例如「風管 vs 結構梁 @ 3F C-4」 |
| `type` | `Topic/@TopicType` | 值可能需要對應外部工具的 extensions（待查證） |
| `status` | `Topic/@TopicStatus` | 五個狀態要對應到對方接受的值（待查證） |
| `assignee` | `Topic/AssignedTo` | |
| `ai_assessment.severity` | `Topic/Priority` | |
| `level` | `Topic/Labels` 或 `Description` | |
| `comments[]` | `Comment`（含 `Date`、`Author`、`Comment`） | 留言要指向 viewpoint（待查證） |
| `point`、`volume_mm3`、`grid_ref` | `Topic/Description` | BCF 沒有對應欄位，放進描述文字 |

```csharp
// Sidecar/Connectors/Bcf21/BcfModel.cs（骨架）
public sealed record BcfTopic(Guid Guid, string Title, string? Type, string? Status,
    string? AssignedTo, string? Priority, string? Description,
    IReadOnlyList<BcfComment> Comments, IReadOnlyList<BcfViewpoint> Viewpoints);

public sealed record BcfViewpoint(Guid Guid, BcfCamera Camera,
    IReadOnlyList<string> SelectedIfcGuids, IReadOnlyList<BcfClippingPlane> Clipping, byte[]? Snapshot);
```

## Step 4. 相機轉換

Revit 端：`View3D.GetOrientation()` 回傳 `ViewOrientation3D`，有 `EyePosition`、`ForwardDirection`、`UpDirection`。Day 12 的 `show_clash` 已把這些存進 `ClashRecord.viewpoint.camera`（eye、target、up、fov）。

BCF 端（待查證，以 buildingSMART BCF-XML 規格為準）：

| BCF 相機 | 欄位 | 對應 |
| --- | --- | --- |
| `PerspectiveCamera` | `CameraViewPoint`、`CameraDirection`、`CameraUpVector`、`FieldOfView`（度） | 透視視圖 |
| `OrthogonalCamera` | 前三者相同，另有 `ViewToWorldScale` | 正交（等角）視圖 |

- Day 12 用 `View3D.CreateIsometric` 建立的是**正交**視圖，所以輸出 `OrthogonalCamera`。`ViewToWorldScale` 的意義（可視範圍的高度？）與計算方式待查證；先用 section box 的大小估算。
- Revit API 是否能直接讀到透視視圖的視角（FOV）待查證。v1 不輸出 `PerspectiveCamera`。
- Section box 可以轉成 BCF 的 `ClippingPlanes`（六個平面，每個由位置與法向量表示，待查證法向量方向的約定）。外部工具是否支援 clipping planes，Day 20 驗證。

```csharp
// Sidecar/Connectors/Bcf21/BcfMapper.cs（骨架）
BcfCamera ToCamera(Viewpoint vp, ICoordinateMapper map)
{
    var eye = map.ToBcf(vp.Camera.Eye);                      // mm → m，並轉換座標系
    var dir = map.ToBcfDirection(vp.Camera.Target - vp.Camera.Eye).Normalize();
    var up  = map.ToBcfDirection(vp.Camera.Up).Normalize();
    return new OrthogonalCamera(eye, dir, up, viewToWorldScale: EstimateScale(vp.SectionBox));
}
```

## Step 5. 座標系與單位：最容易出錯的地方

外部工具會把 BCF 的相機放在**它載入的 IFC 模型座標系**中。所以 BCF 的座標必須和 IFC 匯出時使用的座標基準一致。

| 項目 | 本平台 | BCF（待查證） |
| --- | --- | --- |
| 長度單位 | mm（對外） | 公尺 |
| 座標原點 | `point` 為「專案座標」（Day 10 決定的基準） | 與 IFC 匯出的座標基準一致 |
| 方向向量 | 單位向量 | 單位向量 |

要確認的事：

1. Revit 匯出 IFC 時的座標基準選項（內部原點、專案基準點、測量點 / 共用座標，待查證選項名稱）會改變 IFC 的座標原點。
2. BCF 匯出要**與使用者匯出 IFC 的設定一致**，所以 `ExportOptions` 要讓使用者指定座標基準，預設與公司的 IFC 匯出慣例相同。
3. Day 10 存的 `point` 是哪個座標系，要寫清楚；需要的轉換（平移 + 繞 Z 軸旋轉，即正北角）要在 add-in 端用 `ProjectLocation` 相關 API 取得後存起來（待查證 API）。Sidecar 不能呼叫 Revit，所以轉換參數要在偵測時或匯出前由 add-in 提供。

```csharp
public interface ICoordinateMapper
{
    double[] ToBcf(double[] pointMm);        // 平移、旋轉、mm → m
    double[] ToBcfDirection(double[] dir);   // 只旋轉
}
```

這部分寫完後，用一個已知的點手算一次，和 Revit 中「點位座標」標註的值比對。

## Step 6. 元素選取

`viewpoint.bcfv` 的 `Components/Selection/Component` 以 `IfcGuid` 屬性指定元素（待查證，以 buildingSMART BCF-XML 規格為準）。

- 直接用 `ClashRecord.elements[].ifc_guid`，兩個元素都放進選取。
- `ifc_guid` 的計算是否與 Revit IFC exporter 一致**還沒驗證**（SPEC §11）。Day 20 在外部工具中能不能選到元素，就是第一次驗證。
- 連結模型中的元素：IFC GUID 是在該連結模型自己匯出 IFC 時產生的，外部工具要載入對應的 IFC 才找得到。
- 可見性（`Visibility`、`DefaultVisibility`）：v1 不隱藏任何元素。

## Step 7. 寫 XML 與單元測試

用 `System.Xml.Linq` 輸出，不加第三方套件：

```csharp
// Sidecar/Connectors/Bcf21/BcfXmlWriter.cs（骨架，元素名稱待查證）
XDocument WriteMarkup(BcfTopic t) => new(
    new XElement("Markup",
        new XElement("Topic", new XAttribute("Guid", t.Guid),
            new XElement("Title", t.Title) /* … */),
        t.Comments.Select(WriteComment),
        t.Viewpoints.Select(v => new XElement("Viewpoints", new XAttribute("Guid", v.Guid),
            new XElement("Viewpoint", "viewpoint.bcfv"),
            new XElement("Snapshot", "snapshot.png")))));
```

測試（Mac 可跑）：

- 用官方 XSD 驗證輸出的 `markup.bcf` 與 `viewpoint.bcfv`（`XmlSchemaSet` + `XDocument.Validate`）。把 XSD 放進 `tests/fixtures/bcf21/`，並記錄來源與授權。
- 座標轉換：已知點、已知正北角的手算結果。
- 中文標題、留言中的特殊字元能正確跳脫。

## BCF 2.1 與 3.0 的主要差異（概略，待查證）

| 項目 | 2.1 | 3.0 |
| --- | --- | --- |
| 支援程度 | 目前普遍認為支援的工具較多（Day 20 驗證） | 較新，支援度待查 |
| 檔案結構 | 如 Step 2 | 有調整（例如 extensions 的定義方式） |
| 決定 | v1 實作 | SPEC §8.2 列為之後 |

---

## 驗收

- [ ] 已下載 BCF 2.1 官方 XSD 與範例，本文件中標示待查證的元素名稱已逐一核對並修正
- [ ] `BcfMapper` 能把 `ClashRecord` 轉成 Topic + Viewpoint
- [ ] 輸出的 XML 通過 XSD 驗證（單元測試）
- [ ] 座標轉換有手算比對的測試案例
- [ ] 座標基準選項寫進 `ExportOptions`，並說明要與 IFC 匯出設定一致
- [ ] Topic 對應單位（每筆或每群）已決定並記錄

## 常見問題

**Q：可以用現成的 BCF 函式庫嗎？**
可以評估（.NET 生態系中有一些開源的 BCF 函式庫，名稱與授權待查證）。但 BCF 2.1 的 XML 不大，自己寫可以避免相依問題，也更了解格式。若選用套件，放在 Sidecar，不要放進 Add-in。

**Q：截圖要多大？**
BCF 規格本身對截圖大小可能有建議（待查證）。先沿用 Day 12 的截圖，若外部工具顯示異常再調整。

**Q：Revit 的 `point` 和 IFC 的座標差了一個固定值？**
幾乎可以確定是座標基準不一致（Step 5）。先確認 IFC 匯出設定，再調整 `ICoordinateMapper`。

## 學習筆記

（寫下今天的心得、卡關點、想問的問題）
