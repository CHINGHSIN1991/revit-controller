# Week 1 · 環境、骨架、通訊管線（M0）

第一週不做任何碰撞功能。目標是打通整條管線：**Mac 上的 AI harness → MCP → Sidecar → named pipe → Add-in → ExternalEvent → Revit API**，並在兩個 Revit 版本上都跑通。唯一的業務工具是唯讀的 `get_model_overview`。

管線打通後，第二週起每加一個功能，都只是「在 add-in 加一個 handler、在 Sidecar 加一個工具」。

**本週文件中的程式碼都是骨架，撰寫時沒有編譯或執行過**（見 [../README.md](../README.md#文件的可信度)）。標示「（待查證）」的 API、套件名稱與版本差異，請對照實際安裝的版本確認，並回填到當天文件。

## 每日日程

| Day | 主題 | 文件 |
| --- | --- | --- |
| Day 1 | 環境建置與版本查證 | [day-01.md](day-01.md) |
| Day 2 | Solution 結構與多版本 build | [day-02.md](day-02.md) |
| Day 3 | Add-in 載入與 ExternalEvent 佇列 | [day-03.md](day-03.md) |
| Day 4 | Add-in ↔ Sidecar 的本機 RPC | [day-04.md](day-04.md) |
| Day 5 | Sidecar 的 MCP Server | [day-05.md](day-05.md) |
| Day 6 | `get_model_overview` 與 Mock transport | [day-06.md](day-06.md) |
| Day 7 | 安裝程式、雙版本驗證、M0 驗收 | [day-07.md](day-07.md) |

## Week 1 Definition of Done

- [ ] 從 Mac 透過 MCP 呼叫 `get_model_overview`，在至少兩個 Revit 版本上都能取得結果
- [ ] Mock 讓 sidecar 測試可以在沒有 Revit 的環境中執行

完整的驗收清單見 [day-07.md](day-07.md) 的 M0 Checklist。

## 一次呼叫經過的路徑

| 層 | 本週哪一天做 | 關鍵類別 |
| --- | --- | --- |
| AI harness → Sidecar（MCP over HTTP + Bearer Token） | Day 5 | `RevitTools`、`BearerTokenMiddleware`、`AuditLog` |
| Sidecar → Add-in（JSON-RPC 2.0 over named pipe） | Day 4 | `IRevitTransport`、`PipeTransport`、`PipeServer` |
| Pipe 執行緒 → UI thread | Day 3 | `RequestQueue`、`RevitEventHandler` |
| UI thread → Revit API | Day 6 | `ModelOverviewHandler` |
| Sidecar → Mock（取代上面三層） | Day 6 | `MockTransport` |

## 本週程式碼的落點

```
revit-controller/
├── RevitController.sln                     ← Day 2
├── RevitController.Mac.slnf                ← 只含可在 Mac build 的專案（Day 2）
├── Directory.Build.props                   ← 共用設定（Day 2）
├── src/
│   ├── RevitController.Core/               netstandard2.0（Day 2）
│   │   ├── Rpc/RpcRequest.cs · RpcResponse.cs · RpcError.cs   ← Day 4
│   │   ├── Rpc/ErrorCodes.cs              ← RevitBusy / NoActiveDocument / Timeout…（Day 3，Day 4 對應錯誤碼）
│   │   ├── Rpc/IRevitTransport.cs         ← Day 4
│   │   └── Models/ModelOverview.cs        ← Day 6
│   ├── RevitController.Addin/              依 Revit 版本分別 build（Day 2）
│   │   ├── App.cs                         ← IExternalApplication（Day 3）
│   │   ├── AddinLog.cs                    ← 最小檔案 log（Day 3）
│   │   ├── Compat/                        ← 唯一允許 #if REVIT… 的地方（Day 2 起）
│   │   ├── Execution/RequestQueue.cs · RevitEventHandler.cs   ← Day 3
│   │   ├── Rpc/PipeServer.cs · RpcSerializer.cs               ← Day 4
│   │   ├── Handlers/ModelOverviewHandler.cs                   ← Day 6
│   │   └── RevitController.addin          ← manifest（Day 3）
│   ├── RevitController.Sidecar/            net8.0（Day 2，Day 5 改為 ASP.NET Core）
│   │   ├── Program.cs                     ← Day 4 暫時的 --ping，Day 5 MCP server
│   │   ├── Mcp/RevitTools.cs              ← ping（Day 5）、get_model_overview（Day 6）
│   │   ├── Mcp/BearerTokenMiddleware.cs   ← Day 5
│   │   ├── Revit/PipeTransport.cs         ← Day 4
│   │   └── Audit/AuditLog.cs              ← Day 5
│   └── RevitController.Mock/
│       └── MockTransport.cs               ← Day 6
├── tests/
│   ├── RevitController.Core.Tests/         ← Day 2 建立，Day 6 加測試
│   └── RevitController.Sidecar.Tests/      ← Day 6
├── fixtures/mock/model.overview.json       ← Day 6
└── scripts/
    ├── build-all.ps1                       ← Day 2
    └── deploy-addin.ps1                    ← Day 7
```

與 [02-tech-stack.md](../02-tech-stack.md) 目錄樹的差異：`IRevitTransport` 放在 Core（避免 Sidecar 與 Mock 循環引用），Mock 的 fixture 放在根目錄 `fixtures/mock/`。Day 7 回填時一併更新 02-tech-stack.md。

## 本週的學習重點

| 主題 | 內容 |
| --- | --- |
| 多版本 build | SDK-style csproj、自訂 configuration（`Release.R24`、`Release.R25`）、`REVIT20XX_OR_GREATER`、RevitAPI `Private=false` |
| netstandard2.0 | 為什麼 Core 能同時被 net48 與 net8 引用；它少了哪些 API |
| Revit 執行限制 | API 只能在 API context 中呼叫；pipe 執行緒絕不碰 Revit API |
| ExternalEvent | `Raise()` 延後執行、重複 Raise 合併、handler 要清空佇列、modal 對話框會讓請求卡住 |
| async 橋接 | `TaskCompletionSource` 把 callback 轉成 `await`；timeout 用 `Task.WhenAny` |
| 本機 RPC | Named pipe、JSON-RPC 2.0、換行分隔訊息、斷線重連、限定目前使用者 |
| MCP | tools / schema / description、Streamable HTTP、Bearer Token、audit log（JSON lines） |
| Revit 模型讀取 | `FilteredElementCollector`、quick vs slow filter、`Level`、`RevitLinkInstance` / `RevitLinkType` |
| 可測試性 | `IRevitTransport` 抽象，Mock 讓 Sidecar 在 Mac 與 CI 上不需要 Revit |

## 本週最需要先查證的項目

這些會直接影響能不能 build 或連線，遇到時優先確認並回填：

| 項目 | 出現在 |
| --- | --- |
| 各 Revit 版本對應的 .NET 版本、`ElementId.Value` 的版本界線 | Day 1、Day 2 |
| RevitAPI NuGet 套件名稱與授權（例如 Nice3point.Revit.Api） | Day 2 |
| net48 add-in 在 Revit 中使用 System.Text.Json 是否有相依衝突 | Day 4 |
| `PipeOptions.CurrentUserOnly` 在 net48 是否可用，或改用 `PipeSecurity` | Day 4 |
| MCP C# SDK 的套件名稱與 Streamable HTTP 的註冊 API | Day 5 |
| 是否有可靠的方式偵測 modal 對話框（目前一律回傳 `Timeout`） | Day 3 |
| `RevitLinkType.GetLinkedFileStatus()` 等連結狀態 API | Day 6 |
