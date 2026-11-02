# Day 13 · 連接器介面與 Excel / HTML 連接器

## 今日目標

在 Core 定義 `IClashConnector` 與 `ConnectorCapabilities`（SPEC §8.1），在 Sidecar 實作第一批連接器：Excel（一筆碰撞一列，附截圖）與 HTML 報告（單一檔案、可獨立開啟、內含截圖與篩選）。最後加上 `list_connectors` 與 `export_clashes` 兩個 MCP 工具。

今天全部是 Sidecar 與 Core 的工作，**可以完全在 Mac 上用 Mock 與 SQLite 中的資料完成**。

## 你會學到

- 連接器模式（strategy / plugin）：新增一個工具 = 新增一個實作
- 能力旗標（`[Flags] enum`）的設計與檢查方式
- 為什麼連接器只能「從中立格式轉出」，不能反過來修改核心資料模型
- Excel 函式庫的選擇與授權差異
- 單一檔案 HTML 報告：base64 內嵌圖片與檔案大小

---

## Step 1. Core：介面與能力旗標

直接照抄 SPEC §8.1，不要自行增減。放在 `src/RevitController.Core/Connectors/`：

```csharp
[Flags]
public enum ConnectorCapabilities
{
    None = 0, Export = 1 << 0, Import = 1 << 1, SyncStatus = 1 << 2,
    Snapshots = 1 << 3, Viewpoints = 1 << 4,
}

public interface IClashConnector
{
    string Id { get; }
    string DisplayName { get; }
    ConnectorCapabilities Capabilities { get; }
    Task<ExportResult> ExportAsync(IReadOnlyList<ClashRecord> clashes, ExportOptions options, CancellationToken ct);
    Task<ImportResult> ImportAsync(ImportSource source, CancellationToken ct);
    Task<SyncResult>   SyncStatusAsync(IReadOnlyList<ClashRecord> clashes, CancellationToken ct);
}
```

`ExportOptions`、`ExportResult` 等型別 SPEC 沒有定義，今天先定最小版本：

| 型別 | 先放的欄位 |
| --- | --- |
| `ExportOptions` | `OutputPath`、`IncludeSnapshots`、`Title` |
| `ExportResult` | `FilePath`、`ExportedCount`、`Warnings`、`ExternalRefs`（clash_id → 外部 ID） |
| `ImportSource` / `ImportResult` / `SyncResult` | 空殼，Week 3 以後再補 |

| 設計問題 | 說明 |
| --- | --- |
| 為什麼用 `[Flags]`？ | 一個連接器可以有多種能力，用 `Capabilities.HasFlag(ConnectorCapabilities.Export)` 檢查 |
| 不支援 Import 的連接器怎麼辦？ | `ImportAsync` 丟 `NotSupportedException`；呼叫前先檢查旗標 |
| 為什麼放 Core？ | Week 4 的 Dockable Pane（在 add-in 中）要列出可匯出的連接器 |

注意：Core 是 netstandard2.0，`Task`、`CancellationToken`、`IReadOnlyList` 都可以用。

## Step 2. 註冊與選擇

```csharp
// src/RevitController.Sidecar/Connectors/ConnectorRegistry.cs
public sealed class ConnectorRegistry
{
    readonly Dictionary<string, IClashConnector> _byId;
    public ConnectorRegistry(IEnumerable<IClashConnector> connectors) => _byId = connectors.ToDictionary(c => c.Id);
    public IClashConnector Get(string id) => _byId.TryGetValue(id, out var c) ? c : throw new KeyNotFoundException(id);
    public IEnumerable<IClashConnector> All => _byId.Values;
}
```

在 `Program.cs` 用 DI 註冊：`services.AddSingleton<IClashConnector, ExcelConnector>()`、`HtmlConnector` 同理。Week 3 加 BCF 時只要多一行。

## Step 3. Excel 連接器

函式庫的選擇：

| 函式庫 | 授權（待確認，以官方最新說明為準） | 備註 |
| --- | --- | --- |
| ClosedXML | MIT（待確認） | [02-tech-stack.md](../02-tech-stack.md) 預設選這個 |
| EPPlus | 新版改為商用授權（待確認從哪一版開始） | 商業產品要特別注意 |
| NPOI | Apache 2.0（待確認） | API 比較底層 |

決定後在 02-tech-stack.md 記錄套件名稱、版本與授權。

```csharp
// src/RevitController.Sidecar/Connectors/ExcelConnector.cs
public sealed class ExcelConnector : IClashConnector
{
    public string Id => "excel";
    public string DisplayName => "Excel (.xlsx)";
    public ConnectorCapabilities Capabilities => ConnectorCapabilities.Export | ConnectorCapabilities.Snapshots;

    public Task<ExportResult> ExportAsync(IReadOnlyList<ClashRecord> clashes, ExportOptions o, CancellationToken ct)
    {
        // using var wb = new XLWorkbook();
        // var ws = wb.Worksheets.Add("Clashes");
        // 標題列：clash_id、rule、level、grid、元素 A、元素 B、volume_mm3、status、截圖
        // 每筆一列；截圖：ws.AddPicture(path).MoveTo(ws.Cell(row, col))（API 待查證）
        // wb.SaveAs(o.OutputPath);
        throw new NotImplementedException();
    }
    // ImportAsync / SyncStatusAsync → throw new NotSupportedException();
}
```

- 欄位名稱用中文或英文要和使用者確認；建議第一列放英文欄位 key，方便之後匯入時對應。
- 圖片太多時 Excel 會變得很大，`IncludeSnapshots = false` 時只放 `snapshot_path`。
- `volume_mm3` 顯示時可以換成 cm³，但只改顯示格式，不改資料。

## Step 4. HTML 報告

單一 `.html` 檔，寄給沒有任何軟體的人也能開：

```
report.html
├── <style>        內嵌 CSS
├── <table>        一筆碰撞一列，含 data-level、data-rule 屬性
├── <img src="data:image/png;base64,…">   內嵌截圖
└── <script>       內嵌的篩選邏輯（依樓層、規則、狀態），不引用外部 CDN
```

```csharp
public sealed class HtmlConnector : IClashConnector
{
    public string Id => "html";
    public string DisplayName => "HTML report";
    public ConnectorCapabilities Capabilities => ConnectorCapabilities.Export | ConnectorCapabilities.Snapshots;
    // ExportAsync：用 StringBuilder 或簡單的範本組出 HTML
    // 所有來自模型的文字都要 HTML escape（System.Net.WebUtility.HtmlEncode）
}
```

| 問題 | 處理 |
| --- | --- |
| 檔案大小 | base64 會讓圖片大約變大三分之一。碰撞數百筆時檔案可能很大，設定上限，或把圖片縮小、改成 JPEG |
| 安全性 | 元素名稱、參數值可能含 `<script>`，一定要 escape（模型內容是資料，不是指令） |
| 離線開啟 | 不引用外部 CDN、字型或圖片 |

## Step 5. MCP 工具

| 工具 | 參數 | 行為 |
| --- | --- | --- |
| `list_connectors` | 無 | 回傳 `id`、`display_name`、`capabilities`（字串陣列，例如 `["export","snapshots"]`） |
| `export_clashes` | `connector_id`、`filter`（與 `list_clashes` 相同）、`options` | 檢查連接器有 `Export` 能力 → 從 Store 取資料 → 呼叫 `ExportAsync` → 回傳檔案路徑與筆數 |

- `filter` 直接重用 Day 12 的查詢邏輯，不要寫第二份。
- 匯出前如果某些碰撞還沒有截圖，先決定行為：略過圖片並在 warnings 說明（建議），或自動呼叫 `show_clash`（會操作 Revit，較慢）。
- 輸出路徑由 Sidecar 決定（例如 `%LocalAppData%\RevitController\exports\`），不要讓 AI 指定任意路徑，避免寫到不該寫的地方。
- `ExternalRefs` 寫回 Store 的 `external_refs[connector_id]`；Excel / HTML 沒有外部 ID，可以留空。

## Step 6. 測試

- Core：`ConnectorCapabilities` 的旗標組合。
- Sidecar：用 Day 11 的 Mock 資料產生 10 筆碰撞，匯出 Excel 與 HTML；Excel 用函式庫讀回來檢查列數；HTML 檢查含有 10 列、escape 正確（放一筆名稱含 `<b>` 的元素）。
- 手動：在 Mac 上用 Excel / Numbers 與瀏覽器開啟。

---

## 驗收

- [ ] `IClashConnector`、`ConnectorCapabilities` 與 SPEC §8.1 一致
- [ ] `list_connectors` 列出 `excel`、`html` 與各自能力
- [ ] `export_clashes` 匯出的 Excel 一筆碰撞一列，有截圖
- [ ] HTML 可以離線開啟、篩選可用、含截圖；元素名稱有 escape
- [ ] 對沒有 `Export` 能力的連接器呼叫 `export_clashes`，回傳清楚的錯誤
- [ ] Excel 函式庫的名稱、版本、授權已記錄在 02-tech-stack.md

## 常見問題

**Q：為什麼不讓連接器直接讀 SQLite？**
連接器只認識 SPEC §5.4 的中立格式，不知道資料存在哪裡。這樣換掉儲存方式或加新連接器都不會互相影響。

**Q：HTML 報告要用前端框架嗎？**
不需要。一個表格加幾十行原生 JavaScript 就夠，也比較容易保持單一檔案。

**Q：Excel 的截圖位置跑掉？**
圖片在 Excel 中是浮動物件，要設定列高與定位方式。細節依函式庫而定（待查證）。

## 學習筆記

（寫下今天的心得、卡關點、想問的問題）
