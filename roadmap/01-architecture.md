# 01 · 系統架構設計

這個專案最重要的學習目標是：**理解 Revit 的執行限制，以及如何在這個限制下讓外部的 AI 安全地操作模型。**

完整規格見 [SPEC.md](../SPEC.md)，這份文件著重在「為什麼這樣分」。

## 架構總覽

```
┌──────────────────────────────────────────────────────────────┐
│  AI Harness（Mac）                                            │
│  任何支援 MCP 的 client：Claude Code、Agent SDK、MCP Inspector │
└──────────────────────────────┬───────────────────────────────┘
                               │  MCP over Streamable HTTP + Bearer Token
┌──────────────────────────────▼───────────────────────────────┐
│  Sidecar Process（Windows，.NET 8）                           │
│  MCP Server · Job 管理 · Connectors · SQLite · Audit log      │
│  不碰 Revit API；可以自由使用第三方套件                         │
└──────────────────────────────┬───────────────────────────────┘
                               │  JSON-RPC 2.0 over Named Pipe（本機）
┌──────────────────────────────▼───────────────────────────────┐
│  Revit Add-in（在 Revit.exe 內，依版本分別 build）              │
│  Pipe Server（背景執行緒）→ 請求佇列 → ExternalEvent            │
│  Handler 在 UI thread 上呼叫 Revit API                         │
└──────────────────────────────┬───────────────────────────────┘
                               │
┌──────────────────────────────▼───────────────────────────────┐
│  Revit Document（主模型 + 連結模型）                            │
└──────────────────────────────────────────────────────────────┘
```

## 為什麼分成這幾層？

### Add-in：只做 Revit 才能做的事

Revit API 只能在 Revit 程序裡、在 Revit 給你的 API context 中呼叫（通常就是 UI thread）。從背景執行緒直接呼叫 API，輕則丟例外，重則讓 Revit 當掉。

所以 add-in 的設計原則是：

- **少**：只放 Revit API 呼叫，業務邏輯放 Core，其他功能放 Sidecar。
- **不擋 UI**：長時間的工作要分批，讓使用者還能操作 Revit。
- **相依套件越少越好**：同一個 Revit 裡的所有 add-in 共用同一個程序。如果別的 add-in 載入了同一個 DLL 的不同版本，就可能衝突。

### Sidecar：AI 與外部世界的出入口

- Revit 重開或當掉時，sidecar 不受影響，AI 端的連線也不用重建。
- MCP、SQLite、Excel、BCF、APS 這些需要第三方套件的功能都放這裡。
- 它在 Mac 上也能搭配 Mock 執行，所以大部分開發和測試不需要 Windows。

### Core：兩邊共用的資料與邏輯

- 目標框架 netstandard2.0，所以 .NET Framework 4.8 版的 add-in 和 .NET 8 版的 sidecar 都能引用。
- 不引用 RevitAPI，可以在任何平台上做單元測試。
- 放資料模型（`ClashRule`、`ClashRecord`）、RPC 訊息格式、分群與狀態邏輯、連接器介面。

## 一次完整呼叫的流程（以 `get_model_overview` 為例）

| 步驟 | 執行環境 | 動作 |
| --- | --- | --- |
| 1 | AI Harness | 呼叫 MCP tool `get_model_overview` |
| 2 | Sidecar（MCP） | 驗證 Bearer Token，寫入 audit log |
| 3 | Sidecar（RPC client） | 透過 named pipe 送出 `{"jsonrpc":"2.0","id":1,"method":"model.overview"}` |
| 4 | Add-in（pipe 執行緒） | 收到請求，放進佇列，呼叫 `ExternalEvent.Raise()`，等待 `TaskCompletionSource` |
| 5 | Add-in（UI thread） | Revit 閒置時執行 handler，用 `FilteredElementCollector` 讀取資料 |
| 6 | Add-in（pipe 執行緒） | `TaskCompletionSource` 完成，把結果寫回 pipe |
| 7 | Sidecar | 把結果轉成 MCP tool result 回傳，寫入 audit log |

## 執行緒模型

| 執行緒 | 可以做 | 不可以做 |
| --- | --- | --- |
| Revit UI thread（ExternalEvent handler 內） | 呼叫 Revit API、開 Transaction | 長時間阻塞（會凍結 Revit） |
| Add-in 的 pipe 執行緒 | 讀寫 pipe、序列化、排入佇列 | **呼叫任何 Revit API** |
| Sidecar 的任何執行緒 | 任何事 | — |

ExternalEvent 的重點：

- `Raise()` 不會立刻執行，而是請 Revit 在下次閒置時執行。
- Revit 有 modal 對話框開著時，handler 不會執行，請求會一直卡住。所以每個請求都要有 timeout，並回傳 `ModalDialogOpen` 或 `Timeout`。
- 同一個 `ExternalEvent` 在執行前重複 `Raise()` 只會執行一次，所以 handler 每次都要把佇列裡的請求全部處理完。

## 長時間工作（Job）

碰撞測試可能需要數十秒到數分鐘，不能在一個 MCP 呼叫中同步等待。

```
AI ──run_clash_test──▶ Sidecar 建立 job，立即回傳 job_id
                        │
                        └─▶ 分批送 RPC 到 Add-in（每批在 UI thread 上執行一小段）
AI ──get_job_status──▶ Sidecar 回傳進度（已處理 / 總數）
AI ──list_clashes────▶ 完成後查詢結果
```

分批的好處：每批之間 Revit 可以處理使用者操作；取消時只要不再送下一批。

## 安全邊界

| 邊界 | 措施 | 何時完成 |
| --- | --- | --- |
| AI ↔ Sidecar（跨機器） | Bearer Token；只開放區網或 VPN；不對外網開放 | Day 5 |
| Sidecar ↔ Add-in（本機） | Named pipe 限定目前使用者存取 | Day 4 |
| 修改模型 | 預設唯讀；修改類工具要 dry-run 與人工核准 | Day 30 – 31 |
| 模型內容當作輸入 | 元素名稱、參數值可能含有誘導 AI 的文字（prompt injection），工具回傳時要當作資料而不是指令 | Day 22、Day 34 |
| 外部服務憑證（APS） | 不進版本控制；存在 OS 的安全儲存 | Day 32 |

## 開發版 vs 正式版

| 項目 | 開發版（Week 1） | 正式版（Week 5 前） |
| --- | --- | --- |
| Sidecar 啟動 | 手動在 Terminal 啟動 | Add-in 啟動時自動拉起，或安裝成 Windows 服務（待決定） |
| Token | 固定寫在設定檔 | 安裝時隨機產生 |
| Revit 版本 | 兩個版本 | 依客戶調查結果 |
| Pipe 名稱 | 固定 | 含 Revit 版本與程序 ID，避免同時開多個 Revit 時衝突 |
