# Day 6 · `get_model_overview` 與 Mock transport

## 今日目標

做出第一個真正讀取模型的工具 `get_model_overview`：回傳樓層、連結模型（含載入狀態）、各類別元素數量、目前視圖。同時實作 Mock transport，用 JSON fixture 模擬 add-in 的回應，讓 Sidecar 可以在 Mac 上執行；最後用 Mock 寫第一批 Sidecar 單元測試。

## 你會學到

- `UIApplication`、`UIDocument`、`Document` 的關係與取得方式
- `FilteredElementCollector` 的基本用法，以及 quick filter 與 slow filter 的效能差異
- `Level`、`View`、`RevitLinkInstance`、`RevitLinkType` 的關係；連結載入狀態要從哪裡讀
- `BuiltInCategory` 與 `Category` 的對應
- 用介面抽象 transport，讓 Mock 和真實實作可以互換

---

## Step 1. 在 Core 定義回傳格式

`src/RevitController.Core/Models/ModelOverview.cs`：

```csharp
public sealed class ModelOverview
{
    public string Title { get; set; } = "";
    public string RevitVersion { get; set; } = "";
    public List<LevelInfo> Levels { get; set; } = new();          // Name, ElevationMm
    public List<LinkInfo> Links { get; set; } = new();            // Name, InstanceUid, Status
    public Dictionary<string, int> CategoryCounts { get; set; } = new();  // "OST_Walls" → 123
    public string? ActiveView { get; set; }
}
```

- 對外單位一律 mm（SPEC §5.2），`ElevationMm` 在 add-in 內從英尺換算。
- 類別用 `BuiltInCategory` 的名稱（`OST_Walls`），與 `ClashRule` 的 `categories` 欄位一致。
- 這個 DTO 同時給 add-in、Sidecar、Mock 使用，所以放在 Core。

## Step 2. Revit 物件的關係

| 物件 | 取得方式 | 代表 |
| --- | --- | --- |
| `UIApplication` | ExternalEvent handler 的參數 | 整個 Revit 程式（含 UI） |
| `UIDocument` | `uiapp.ActiveUIDocument`（可能為 null） | 目前作用中的文件（含選取、視圖） |
| `Document` | `uidoc.Document` | 模型資料本身 |
| `View` | `doc.ActiveView` | 目前視圖 |

`ActiveUIDocument` 為 null 時回傳 `NoActiveDocument`（Day 3 已實作）。

## Step 3. FilteredElementCollector

```csharp
var levels = new FilteredElementCollector(doc)
    .OfClass(typeof(Level))
    .Cast<Level>()
    .OrderBy(l => l.Elevation)
    .Select(l => new LevelInfo { Name = l.Name, ElevationMm = FeetToMm(l.Elevation) });
```

| 方法 | 類型 | 說明 |
| --- | --- | --- |
| `OfClass(typeof(Level))` | quick filter | 依 .NET 類別篩選 |
| `OfCategory(BuiltInCategory.OST_Walls)` | quick filter | 依類別篩選 |
| `WhereElementIsNotElementType()` | quick filter | 排除類型（Type），只留實例 |
| `ElementParameterFilter` | slow filter | 要展開元素讀參數，較慢（Week 2 會用到） |
| LINQ `.Where(...)` | 在 .NET 端 | 最慢；每個元素都要建立物件 |

原則：**先用 quick filter 把數量縮小，再用 slow filter 或 LINQ。** Quick filter 不需要展開元素，可以在 Revit 內部的索引上完成（細節待查證）。

## Step 4. 各類別元素數量

不要對每個 `BuiltInCategory` 各跑一次 collector（數量上千）。改成一次收集所有實例再分組：

```csharp
var counts = new FilteredElementCollector(doc)
    .WhereElementIsNotElementType()
    .Where(e => e.Category != null && e.Category.CategoryType == CategoryType.Model)
    .GroupBy(e => CategoryName(e.Category))
    .ToDictionary(g => g.Key, g => g.Count());
```

- `CategoryName` 把 `Category` 轉成 `OST_xxx`。`Category.BuiltInCategory` 屬性在哪一版新增待查證；舊版可以用 `(BuiltInCategory)category.Id` 的整數值轉換（`ElementId` 型別差異走 Day 2 的 `Compat/`）。
- 只算 `CategoryType.Model`，排除註解、分析類別，結果對碰撞檢討才有意義。
- 大模型這一步可能要幾秒，先量測耗時並寫進 log。如果太慢，改成只統計常用類別清單。

## Step 5. 連結模型與載入狀態

| 類別 | 代表 | 數量 |
| --- | --- | --- |
| `RevitLinkType` | 連結的「檔案」 | 每個連結檔一個 |
| `RevitLinkInstance` | 連結在主模型中的一次「放置」 | 同一個檔案可以放多次 |

```csharp
foreach (var inst in new FilteredElementCollector(doc).OfClass(typeof(RevitLinkInstance)).Cast<RevitLinkInstance>())
{
    var type = (RevitLinkType)doc.GetElement(inst.GetTypeId());
    var loaded = inst.GetLinkDocument() != null;
    // 狀態細分：type.GetLinkedFileStatus() → LinkedFileStatus（待查證）
}
```

- `GetLinkDocument()` 回傳 null 代表未載入或無法存取，**一定要回報**（SPEC §6：不得略過）。
- `InstanceUid` 用 `inst.UniqueId`，之後碰撞紀錄的 `link_instance_uid` 就是它。

## Step 6. 註冊 RPC 方法與 MCP 工具

Add-in：在 `Handlers/ModelOverviewHandler.cs` 實作，dispatcher 註冊方法名 `model.overview`，透過 `App.Queue.EnqueueAsync` 執行。

Sidecar：在 `Mcp/RevitTools.cs` 加工具：

```csharp
[McpServerTool(Name = "get_model_overview")]
[Description("回傳目前開啟模型的概要：樓層（標高 mm）、連結模型與載入狀態、各模型類別的元素數量、目前視圖。唯讀。開始任何分析前先呼叫。")]
public Task<string> GetModelOverview(CancellationToken ct)
    => audit.RunAsync("get_model_overview", null, () => revit.CallAsync("model.overview", null, ct));
```

## Step 7. Mock transport

`src/RevitController.Mock/MockTransport.cs` 實作 Day 4 的 `IRevitTransport`：

```csharp
public sealed class MockTransport(string fixtureDir) : IRevitTransport
{
    public Task<RpcResponse> CallAsync(string method, object? @params, CancellationToken ct)
    {
        var file = Path.Combine(fixtureDir, $"{method}.json");    // fixtures/mock/model.overview.json
        return File.Exists(file)
            ? Task.FromResult(ReadFixture(file))      // Mock 內的私有方法，用 System.Text.Json
            : Task.FromResult(MethodNotFound(method)); // 回傳 Error.Code = -32601
    }
}
```

- Fixture 放在 repo 根目錄的 `fixtures/mock/`（與 [README](../README.md) 的目錄結構一致），一個方法一個檔案。
- `IRevitTransport` 在 Day 4 已放在 Core，所以 Mock 只引用 Core，不會和 Sidecar 循環引用。
- 最好的 fixture 來源：在真實 Revit 上呼叫一次 `model.overview`，把回應存下來，再刪減成小檔。
- 也可以加一個 fixture 回傳錯誤（例如 `NoActiveDocument`），用來測試錯誤路徑。

Sidecar 依參數選擇 transport：

```bash
dotnet run --project src/RevitController.Sidecar -- --transport mock
```

這樣在 Mac 上就能啟動 Sidecar，用 MCP Inspector 呼叫 `get_model_overview` 拿到 fixture 的內容。

## Step 8. 第一批單元測試

`tests/RevitController.Sidecar.Tests/`（xUnit，待決定，見 [02-tech-stack.md](../02-tech-stack.md)）：

| 測試 | 驗證 |
| --- | --- |
| `GetModelOverview_ReturnsFixtureContent` | 工具回傳的內容與 fixture 一致 |
| `GetModelOverview_WritesAuditLog` | 呼叫後 audit log 多一行，`tool` 欄位正確 |
| `UnknownMethod_ReturnsMethodNotFound` | Mock 對不存在的方法回 `-32601` |
| `NoActiveDocument_IsReadableMessage` | 錯誤以 AI 看得懂的文字回傳 |
| `ModelOverview_RoundTripsJson` | （Core.Tests）DTO 序列化再反序列化結果相同 |

這些測試不需要 Revit，在 Mac 和 CI 上都能跑：

```bash
dotnet test tests/RevitController.Sidecar.Tests
dotnet test tests/RevitController.Core.Tests
```

---

## 驗收

- [ ] 在真實 Revit 上呼叫 `get_model_overview`，樓層、連結數、目前視圖與 Revit 畫面一致（兩個版本都測）
- [ ] 有未載入的連結時，結果中標示為未載入，沒有被略過
- [ ] 樓層標高以 mm 回傳
- [ ] Mac 上 `--transport mock` 可以啟動 Sidecar，並透過 MCP 取得 fixture 的內容
- [ ] Sidecar 與 Core 的測試在 Mac 上全部通過，不需要 Revit
- [ ] 能說明 quick filter 與 slow filter 的差異

## 常見問題

**Q：類別數量和 Revit 的「明細表」對不起來？**
檢查是否排除了 ElementType、是否只算 Model 類別、是否包含群組內的元素與巢狀族群。先在小模型上手算幾個類別比對。

**Q：連結模型的元素要不要算進數量？**
M0 只算主模型，連結模型只回報名稱與狀態。Week 2 再處理連結內的元素。

**Q：Mock 回應和真實 add-in 的格式不一致？**
這是 Mock 最大的風險。fixture 應該從真實回應擷取；DTO 修改時，同步更新 fixture，並在 Day 7 驗收時再比對一次。

## 學習筆記

（寫下今天的心得、卡關點、想問的問題）
