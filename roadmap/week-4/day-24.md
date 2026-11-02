# Day 24 · Dockable Pane 基礎

> 本文件依計畫推估，開始前請依 Day 21 回顧結果修訂。

## 今日目標

在 Revit 中建立一個 WPF Dockable Pane，顯示碰撞清單，可以依狀態、群組、規則、樓層篩選。資料從 sidecar 取得：Pane 和 AI harness 一樣，都是 sidecar 的 client。今天決定 Pane 與 sidecar 之間的通訊方式並記錄下來。

今天只做「看」，點選、改判、匯出留到 Day 25。

## 你會學到

- `IDockablePaneProvider`、`DockablePaneId`、`RegisterDockablePane`，以及為什麼必須在啟動時註冊
- WPF 基礎：XAML、資料繫結、MVVM
- WPF 在 net48 與 net8 版 Revit 中的差異，以及與其他 add-in 的資源衝突
- UI 事件中不能直接呼叫 Revit API 修改模型，要透過 ExternalEvent

---

## Step 1. 決定 Pane 與 sidecar 的通訊方式

Day 4 的 named pipe 是「add-in 當 server、sidecar 當 client」，方向相反，Pane 不能直接拿來用。選項：

| 選項 | 做法 | 優點 | 缺點 |
| --- | --- | --- | --- |
| A. Sidecar 開本機 HTTP API | Sidecar 在 `127.0.0.1` 開一組 `/api/...` endpoint，與 MCP 工具共用同一個 service 層 | Add-in 端只需要 `HttpClient`（BCL 內建）；和 MCP 走同一套邏輯與 audit log | 多一個 port 要管理；要帶 token |
| B. 第二條 named pipe（sidecar 當 server） | 反向再開一條 pipe | 不經過網路堆疊 | 要多寫一套 server；兩條 pipe 的重連邏輯 |
| C. Pane 直接讀 SQLite | Add-in 引用 SQLite 套件 | 最快 | 違反 SPEC §3.2（add-in 相依越少越好）；繞過 service 層與 audit log |

建議預設採用 **A**，理由是和 MCP 共用 service 層，行為才會一致（Day 25 的重點）。C 不建議。

決定後寫進 [01-architecture.md](../01-architecture.md) 的架構圖與安全邊界表，並在 [PROGRESS.md](../PROGRESS.md) 卡關紀錄或技術債中記錄。

選 A 時要注意：

- 只綁定 `127.0.0.1`，不對外開放。MCP endpoint 可能對區網開放，這組不要跟著開。
- 一樣要求 token。Token 怎麼交給 add-in（設定檔、環境變數）沿用 Day 5 的做法。
- JSON 序列化沿用 Day 4 選定的、net48 也能用的函式庫，不要為了 Pane 新增套件。

## Step 2. Sidecar 端的 service 層

如果 Week 2、3 的 MCP 工具把邏輯直接寫在工具方法裡，今天先抽出來：

```csharp
// src/RevitController.Sidecar/Services/ClashService.cs（骨架）
public sealed class ClashService
{
    public Task<Page<ClashSummary>> ListAsync(ClashFilter filter, PageRequest page, CancellationToken ct) { /* SQLite */ }
    public Task<ClashDetail> GetDetailAsync(string clashId, CancellationToken ct) { /* … */ }
}

// src/RevitController.Sidecar/Mcp/ClashTools.cs      → 呼叫 ClashService
// src/RevitController.Sidecar/LocalApi/ClashEndpoints.cs → 也呼叫 ClashService
```

```csharp
// LocalApi（依 Day 5 實際採用的 HTTP 框架；以下以 minimal API 示意，骨架）
app.MapGet("/api/clashes", (ClashService svc, [AsParameters] ClashQuery q, CancellationToken ct)
    => svc.ListAsync(q.ToFilter(), q.ToPage(), ct));
```

Local API 也要寫 audit log，來源標記為 `pane`，和 AI 的呼叫區分。

## Step 3. 註冊 Dockable Pane

Dockable Pane 必須在 `IExternalApplication.OnStartup` 中註冊，不能等到使用者開文件之後。

```csharp
// src/RevitController.Addin/UI/ClashPaneProvider.cs（骨架）
public sealed class ClashPaneProvider : IDockablePaneProvider
{
    public static readonly DockablePaneId PaneId =
        new DockablePaneId(new Guid("請產生一個固定的 GUID"));

    private readonly ClashPane _pane;
    public ClashPaneProvider(ClashPane pane) => _pane = pane;

    // SetupDockablePane 的參數型別與屬性名稱待查證，以實際版本的 RevitAPI 為準
    public void SetupDockablePane(DockablePaneProviderData data)
    {
        data.FrameworkElement = _pane;          // WPF UserControl
        // 初始停靠位置等設定：屬性名稱與可用值待查證
    }
}
```

```csharp
// App.cs 的 OnStartup 中
var pane = new ClashPane(new ClashPaneViewModel(sidecarClient));
application.RegisterDockablePane(ClashPaneProvider.PaneId, "Clash Review", new ClashPaneProvider(pane));
```

- `DockablePaneId` 的 GUID 要固定，不要每次啟動都產生新的。
- 再加一個 ribbon 按鈕，用來顯示或隱藏 Pane（取得 pane 物件與顯示的 API 待查證）。
- 註冊時機、各版本行為差異，以實際安裝版本的 RevitAPI 文件為準（待查證）。

## Step 4. WPF 專案設定

Add-in 專案要啟用 WPF：

```xml
<PropertyGroup>
  <UseWPF>true</UseWPF>
</PropertyGroup>
```

| 版本 | 注意事項 |
| --- | --- |
| net48 版 Revit | WPF 是 .NET Framework 內建的 |
| net8 版 Revit | 可能需要 `net8.0-windows` 之類的 TFM 才能用 WPF（待查證，以 Day 2 的設定為準） |

資源衝突：

- 不要在 `Application.Current.Resources` 中加全域樣式，會影響 Revit 本身與其他 add-in。樣式只放在 UserControl 自己的 `Resources`。
- 不要引用第三方 UI 套件（例如 MVVM 框架、控制項庫），理由同 SPEC §3.2。簡單的 `INotifyPropertyChanged` 與 `ICommand` 自己寫。

## Step 5. 碰撞清單畫面

```xml
<!-- src/RevitController.Addin/UI/ClashPane.xaml（骨架） -->
<UserControl x:Class="RevitController.Addin.UI.ClashPane" ...>
  <DockPanel>
    <StackPanel DockPanel.Dock="Top" Orientation="Horizontal">
      <ComboBox ItemsSource="{Binding Statuses}" SelectedItem="{Binding StatusFilter}"/>
      <ComboBox ItemsSource="{Binding Levels}"   SelectedItem="{Binding LevelFilter}"/>
      <ComboBox ItemsSource="{Binding Rules}"    SelectedItem="{Binding RuleFilter}"/>
      <Button Content="重新整理" Command="{Binding RefreshCommand}"/>
    </StackPanel>
    <TextBlock DockPanel.Dock="Bottom" Text="{Binding StatusMessage}"/>
    <ListView ItemsSource="{Binding Groups}"> <!-- 依群組分組顯示 --> </ListView>
  </DockPanel>
</UserControl>
```

```csharp
// ClashPaneViewModel.cs（骨架）
private async Task RefreshAsync()
{
    StatusMessage = "載入中…";
    try { Groups = await _client.ListClashesAsync(BuildFilter(), _cts.Token); StatusMessage = ""; }
    catch (HttpRequestException) { StatusMessage = "無法連到 sidecar，請確認已啟動"; }
}
```

MVVM 的分工：XAML 只描述畫面；ViewModel 持有狀態與命令；`SidecarApiClient` 負責 HTTP。ViewModel 不引用 Revit API，可以在 Mac 上寫單元測試（但 WPF 型別本身只能在 Windows 上跑，測試範圍要切清楚）。

## Step 6. 執行緒規則

| 動作 | 在哪裡執行 |
| --- | --- |
| 呼叫 sidecar HTTP API | `async`/`await`，不要 `.Result`，否則會卡住 Revit UI |
| 更新繫結的屬性 | WPF UI thread（await 後通常會回到原本的 context） |
| 呼叫 Revit API | **不能在按鈕事件中直接呼叫**，要排入 ExternalEvent 佇列（Day 3） |

今天的功能只讀 sidecar，不碰 Revit API。Day 25 跳視圖時才需要 ExternalEvent。

---

## 驗收

- [ ] Pane 與 sidecar 的通訊方式已決定，並更新 01-architecture.md
- [ ] Local API 只綁定 `127.0.0.1`、需要 token、寫入 audit log
- [ ] MCP 工具與 Local API 呼叫同一個 `ClashService`
- [ ] 兩個 Revit 版本都能顯示 Pane，Revit 重開後位置與顯示狀態正常
- [ ] 篩選（狀態、群組、規則、樓層）結果與 `list_clashes` 一致
- [ ] Sidecar 未啟動時 Pane 顯示錯誤訊息，Revit 不會卡住
- [ ] Add-in 沒有新增第三方套件

## 常見問題

**Q：Pane 沒有出現？**
先確認註冊是在 `OnStartup` 中執行，且沒有丟例外（看 add-in log）。再確認是否有 ribbon 按鈕或其他方式讓它顯示。參考 [troubleshooting/addin-not-loaded.md](../troubleshooting/addin-not-loaded.md)。

**Q：在 Pane 裡按鈕呼叫 Revit API 為什麼會出錯？**
WPF 事件雖然在 UI thread 上，但不在 Revit 給的 API context 中，修改模型會失敗。一律透過 ExternalEvent。

**Q：清單很長時很慢？**
Pane 也要分頁，或先只顯示群組摘要、展開時再載入。WPF 的 `ListView` 虛擬化是否生效也要檢查。

## 學習筆記

（寫下今天的心得、卡關點、想問的問題）
