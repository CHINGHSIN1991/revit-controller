# Day 3 · Add-in 載入與 ExternalEvent 佇列

## 今日目標

讓 add-in 在 Revit 啟動時自動載入，並建好「從任何執行緒排入請求 → 在 Revit UI thread 上執行 → 把結果還給呼叫端」的佇列。請求要有 timeout，並能回傳 `RevitBusy`、`NoActiveDocument`、`Timeout` 這幾種錯誤狀態。

今天還沒有 named pipe（Day 4 才做），所以用一個暫時的 Ribbon 按鈕從背景執行緒排入請求，驗證佇列可以運作。

## 你會學到

- 為什麼 Revit API 只能在 API context 中呼叫
- `IExternalApplication` 與 `IExternalCommand` 的差異；`.addin` manifest 的欄位
- `ExternalEvent.Raise()` 的語意：延後執行、重複 Raise 會合併
- 用 `TaskCompletionSource` 把 callback 轉成 `async/await`
- `%AppData%` 與 `%ProgramData%` 兩個 Addins 資料夾的差異

---

## Step 1. Revit 的執行限制

| 情境 | 能不能呼叫 Revit API |
| --- | --- |
| `IExternalCommand.Execute` 內（使用者按按鈕） | 可以 |
| `IExternalApplication.OnStartup` 內 | 可以（但還沒有開啟的文件） |
| `IExternalEventHandler.Execute` 內 | 可以 |
| 自己開的 `Thread` / `Task.Run` / pipe 執行緒 | **不可以**，會丟例外或讓 Revit 當掉 |

我們的請求來自 pipe 執行緒（Day 4），所以必須「排隊等 Revit 叫我們」。這就是 `ExternalEvent` 的用途。完整的執行緒模型見 [01-architecture.md](../01-architecture.md#執行緒模型)。

## Step 2. Manifest 與 `IExternalApplication`

`src/RevitController.Addin/RevitController.addin`：

```xml
<?xml version="1.0" encoding="utf-8"?>
<RevitAddIns>
  <AddIn Type="Application">
    <Name>RevitController</Name>
    <Assembly>RevitController.Addin.dll</Assembly>
    <AddInId>（產生一個固定的 GUID，之後不要改）</AddInId>
    <FullClassName>RevitController.Addin.App</FullClassName>
    <VendorId>（待決定，例如公司代號）</VendorId>
  </AddIn>
</RevitAddIns>
```

| 欄位 | 說明 |
| --- | --- |
| `Type` | `Application`（啟動時載入）或 `Command`（單一按鈕） |
| `Assembly` | DLL 路徑；相對路徑以 manifest 所在資料夾為準 |
| `AddInId` | 唯一識別；Revit 用它記住使用者是否信任這個 add-in |
| `FullClassName` | 實作 `IExternalApplication` 的完整類別名稱 |
| `VendorId` | 開發者代號（格式要求待查證） |

`src/RevitController.Addin/App.cs` 骨架：

```csharp
public class App : IExternalApplication
{
    internal static RequestQueue Queue { get; private set; } = null!;

    public Result OnStartup(UIControlledApplication app)
    {
        AddinLog.Write("OnStartup");
        var handler = new RevitEventHandler();
        var ev = ExternalEvent.Create(handler);   // 必須在 API context 中建立
        Queue = new RequestQueue(ev, handler);
        // Day 3 暫時：建立一個 Ribbon 按鈕觸發 SelfTestCommand
        return Result.Succeeded;
    }

    public Result OnShutdown(UIControlledApplication app) => Result.Succeeded;
}
```

## Step 3. 簡單的檔案 log

Add-in 盡量不引用第三方套件，所以自己寫一個最小的 log：

```csharp
internal static class AddinLog
{
    static readonly string Path = System.IO.Path.Combine(
        Environment.GetFolderPath(Environment.SpecialFolder.LocalApplicationData),
        "RevitController", "addin.log");
    static readonly object Gate = new();
    public static void Write(string msg) { lock (Gate) File.AppendAllText(Path, $"{DateTime.Now:O} {msg}\n"); }
}
```

（記得先建立資料夾。）Day 7 的 checklist 會檢查這個 log 裡有沒有 `OnStartup`。

## Step 4. 請求佇列與 handler

設計：

```
任何執行緒 ──Enqueue(work)──▶ ConcurrentQueue ──Raise()──▶ Revit 閒置時
                    ▲                                      │
                    └──── TaskCompletionSource ◀── handler 在 UI thread 執行 work
```

`src/RevitController.Addin/Execution/RequestQueue.cs`：

```csharp
internal sealed class RequestQueue
{
    public Task<object?> EnqueueAsync(Func<UIApplication, object?> work, TimeSpan timeout)
    {
        var item = new PendingRequest(work, new TaskCompletionSource<object?>(
            TaskCreationOptions.RunContinuationsAsynchronously));
        _items.Enqueue(item);
        _event.Raise();                         // 回傳值寫進 log（列舉值待查證）
        return WithTimeout(item, timeout);      // 逾時回傳 Timeout 錯誤
    }
}
```

`src/RevitController.Addin/Execution/RevitEventHandler.cs`：

```csharp
internal sealed class RevitEventHandler : IExternalEventHandler
{
    public void Execute(UIApplication app)
    {
        while (_queue.TryDequeue(out var req))          // 一次處理完所有請求
        {
            if (req.IsCancelled) continue;               // 已經逾時的就跳過
            try   { req.Tcs.TrySetResult(req.Work(app)); }
            catch (Exception ex) { req.Tcs.TrySetException(ex); }
        }
    }
    public string GetName() => "RevitController";
}
```

重點：

| 細節 | 為什麼 |
| --- | --- |
| handler 用 `while` 把佇列清空 | 重複 `Raise()` 會合併，若一次只處理一筆，後面的請求會卡住直到下一次 Raise |
| `RunContinuationsAsynchronously` | 避免 `TrySetResult` 時在 UI thread 上同步執行呼叫端的後續程式碼 |
| 逾時的請求標記為 cancelled | handler 之後才執行時直接跳過，不要再動模型 |
| 例外要接住並轉給 `Tcs` | 未處理的例外會讓 Revit 顯示錯誤對話框，甚至影響其他 add-in |

## Step 5. 錯誤狀態

錯誤碼定義在 Core（`src/RevitController.Core/Rpc/ErrorCodes.cs`），Day 4 會對應到 JSON-RPC error code：

| 狀態 | 何時回傳 | 今天是否實作 |
| --- | --- | --- |
| `Timeout` | 等待超過設定時間（預設值待決定，例如 30 秒） | 是 |
| `NoActiveDocument` | `app.ActiveUIDocument` 為 null | 是，在 handler 裡檢查 |
| `RevitBusy` | 佇列長度超過上限，直接拒絕新請求 | 是（判斷方式先用佇列長度） |
| `ModalDialogOpen` | Revit 有對話框開著 | **否**。目前不確定有可靠的 API 能偵測（待查證），先統一回傳 `Timeout` |
| `LinkNotLoaded` | 連結模型未載入 | Week 2 |

`Timeout` 的實作骨架：`Task.WhenAny(item.Tcs.Task, Task.Delay(timeout))`，逾時就把 item 標記為 cancelled，並丟出帶有 `Timeout` 錯誤碼的例外。

## Step 6. 暫時的自我測試指令

在 `OnStartup` 加一個 Ribbon 按鈕，對應 `SelfTestCommand : IExternalCommand`。按下後**從背景執行緒**排入請求：

```csharp
Task.Run(async () =>
{
    var title = await App.Queue.EnqueueAsync(
        app => app.ActiveUIDocument?.Document.Title, TimeSpan.FromSeconds(10));
    AddinLog.Write($"SelfTest: {title}");
});
```

驗證三件事：

1. 開著專案時按下 → log 出現專案名稱。
2. 沒有開專案時按下 → log 出現 `NoActiveDocument`。
3. 先開「選項」對話框，再從另一個觸發來源排入請求（例如 Step 6 的 Task 中先 `Delay` 幾秒，趁這段時間打開對話框）→ 約 10 秒後 log 出現 `Timeout`，Revit 沒有卡死。

Day 4 有了 pipe 之後，這個按鈕可以移除或保留成診斷工具。

## Step 7. 部署到 Addins 資料夾

| 路徑 | 適用 |
| --- | --- |
| `%AppData%\Autodesk\Revit\Addins\20XX\` | 目前使用者；開發時用這個，不需要管理員權限 |
| `%ProgramData%\Autodesk\Revit\Addins\20XX\` | 所有使用者；正式安裝時考慮 |

今天手動複製：`RevitController.addin` 放在 `Addins\20XX\`。原始碼中的 manifest 只寫 DLL 檔名，所以今天可以把 DLL 直接放在 manifest 旁邊；或放進 `Addins\20XX\RevitController\` 子資料夾，再手動把 `<Assembly>` 改成 `RevitController\RevitController.Addin.dll`。Day 7 的 `deploy-addin.ps1` 會自動做這個改寫。載入失敗時看 [troubleshooting/addin-not-loaded.md](../troubleshooting/addin-not-loaded.md)。

---

## 驗收

- [ ] 兩個 Revit 版本啟動時都會載入 add-in，`addin.log` 出現 `OnStartup`
- [ ] 自我測試按鈕可以從背景執行緒排入請求，並取得目前文件名稱
- [ ] 沒有開啟文件時回傳 `NoActiveDocument`
- [ ] 有 modal 對話框時，請求在 timeout 後回傳 `Timeout`，Revit 不會卡死
- [ ] 連續快速排入多個請求，全部都能完成（驗證 handler 有清空佇列）
- [ ] 能說明為什麼不能在 pipe 執行緒上直接呼叫 Revit API

## 常見問題

**Q：Revit 啟動時跳出「無法驗證發行者」之類的安全提示？**
未簽章的 add-in 會出現這個提示，選「總是載入」即可。數位簽章在 Day 7 與正式安裝程式時處理。

**Q：`ExternalEvent.Create` 丟例外？**
多半是在非 API context（例如背景執行緒）呼叫。放在 `OnStartup` 裡建立。見 [troubleshooting/external-event-not-firing.md](../troubleshooting/external-event-not-firing.md)。

**Q：handler 一直沒有被執行？**
確認 Revit 沒有在編輯模式（按 Esc）、沒有對話框開著；並檢查 `Raise()` 的回傳值。

**Q：DLL 被鎖住，無法覆蓋？**
Revit 開著時會鎖住已載入的 DLL。重新部署前先關閉 Revit。有些工具（例如 Add-in Manager 類工具，待查證）可以熱重載，但先不要依賴。

## 學習筆記

（寫下今天的心得、卡關點、想問的問題）
