# Day 4 · Add-in ↔ Sidecar 的本機 RPC

## 今日目標

讓 Sidecar 能透過 named pipe 呼叫 add-in 的方法。訊息格式用 JSON-RPC 2.0，定義在 Core；add-in 端是 pipe server，收到請求後丟進 Day 3 的 ExternalEvent 佇列；Sidecar 端是 pipe client，Revit 重開後要能自動重連。第一個方法是 `ping`。

## 你會學到

- Named pipe 的原理，以及 `NamedPipeServerStream` / `NamedPipeClientStream` 的用法
- JSON-RPC 2.0 的 request、response、notification 與錯誤碼
- 在 stream 上切訊息邊界：換行分隔 vs length-prefix
- Add-in 端選 JSON 函式庫時要注意的相依衝突
- 怎麼把 pipe 限制為只有目前使用者可以連線

---

## Step 1. 為什麼用 named pipe

| 選項 | 適合 | 不選的原因 |
| --- | --- | --- |
| Named pipe | 同一台 Windows 上的兩個程序 | — |
| 本機 HTTP（localhost） | 也可以 | 要在 Revit 程序內開 HTTP server，相依較多；同機其他程式也能連 |
| 共用檔案 / 資料庫輪詢 | 簡單 | 延遲高、要處理鎖定 |

Named pipe 有作業系統層級的存取控制，可以限定只有目前登入的使用者能連線（見 [01-architecture.md](../01-architecture.md#安全邊界)）。

## Step 2. 在 Core 定義訊息格式

`src/RevitController.Core/Rpc/`：

```csharp
public sealed class RpcRequest
{
    public string Jsonrpc { get; set; } = "2.0";
    public long? Id { get; set; }              // null 代表 notification（不需要回應）
    public string Method { get; set; } = "";
    public object? Params { get; set; }        // 實際型別依 JSON 函式庫決定，見 Step 4
}

public sealed class RpcResponse
{
    public string Jsonrpc { get; set; } = "2.0";
    public long? Id { get; set; }
    public object? Result { get; set; }
    public RpcError? Error { get; set; }       // Result 與 Error 只會有一個
}

public sealed class RpcError { public int Code { get; set; } public string Message { get; set; } = ""; public object? Data { get; set; } }
```

錯誤碼：JSON-RPC 保留 `-32768` 到 `-32000`（例如 `-32601` Method not found、`-32602` Invalid params）。我們自己的錯誤用另一段範圍：

| Code | 名稱 | 來源 |
| --- | --- | --- |
| -32601 | MethodNotFound | JSON-RPC 標準 |
| -32602 | InvalidParams | JSON-RPC 標準 |
| 1001 | `RevitBusy` | Day 3 |
| 1002 | `ModalDialogOpen` | 保留（目前不會回傳） |
| 1003 | `NoActiveDocument` | Day 3 |
| 1004 | `Timeout` | Day 3 |
| 1005 | `LinkNotLoaded` | Week 2 |

（數字是本專案自訂的，可以調整，但定好後就不要改。）

方法名稱用 `領域.動作` 的格式：`ping`、`model.overview`（Day 6）。

## Step 3. 訊息邊界

Pipe 是 byte stream，一次 `Read` 不保證剛好讀到一則完整訊息。兩種切法：

| 方式 | 寫法 | 優缺點 |
| --- | --- | --- |
| 換行分隔（NDJSON） | 每則訊息序列化成一行 JSON，以 `\n` 結尾 | 簡單、好除錯（可以直接看）；JSON 內不可有未跳脫的換行（序列化器預設會跳脫） |
| Length-prefix | 先寫 4 bytes 長度，再寫內容 | 適合二進位與很大的訊息；除錯時看不懂 |

**本專案選換行分隔**：訊息都是 JSON，好除錯。之後若要傳大量幾何資料再評估。讀寫用 `StreamReader.ReadLineAsync` / `StreamWriter.WriteLineAsync`，並設 `AutoFlush`。編碼統一 UTF-8 無 BOM。

## Step 4. Add-in 端的 JSON 函式庫

Add-in 跑在 Revit 程序裡，和其他 add-in 共用同一個程序：

| 選項 | 風險 |
| --- | --- |
| Newtonsoft.Json | Revit 本身或其他 add-in 可能已載入另一個版本，造成版本衝突（Revit 是否自帶此 DLL 及版本待查證） |
| System.Text.Json | net8 版內建；net48 版要另外帶 NuGet 套件及其相依 DLL（待查證相依清單） |
| 自己寫簡單的序列化 | 沒有相依，但只適合很小的訊息 |

建議：net8 版直接用內建 System.Text.Json；net48 版先試 System.Text.Json NuGet 套件，若在 Revit 中出現載入衝突再改。Sidecar 與 Mock（net8）直接用內建 System.Text.Json，各自在自己的專案裡做序列化；Core 只放型別，不放序列化程式碼。把 add-in 的序列化包在 Addin 的一個類別（例如 `Rpc/RpcSerializer.cs`）裡，換函式庫時只改一處。Revit 2025 以後是否有 add-in 相依隔離機制，見 SPEC §11（待查證）。

## Step 5. Add-in 端：PipeServer

`src/RevitController.Addin/Rpc/PipeServer.cs`，在 `OnStartup` 時啟動，跑在背景執行緒：

```csharp
while (!ct.IsCancellationRequested)
{
    using var pipe = new NamedPipeServerStream(PipeName, PipeDirection.InOut, 1,
        PipeTransmissionMode.Byte, PipeOptions.Asynchronous /* | CurrentUserOnly，待查證 */);
    await pipe.WaitForConnectionAsync(ct);
    var reader = new StreamReader(pipe); var writer = new StreamWriter(pipe) { AutoFlush = true };
    string? line;
    while ((line = await reader.ReadLineAsync()) != null)
        await writer.WriteLineAsync(await _dispatcher.HandleAsync(line));   // 逐筆處理
}
```

- 這個迴圈在 pipe 執行緒上，**不可以呼叫 Revit API**。需要 Revit 的方法交給 `App.Queue.EnqueueAsync`。
- `ping` 不需要 Revit API，但為了驗證整條路，讓它也走一次佇列，回傳 `{ "pong": true, "revitVersion": app.Application.VersionNumber }`。
- `_dispatcher` 依 `method` 找對應的處理函式；找不到回 `-32601`。
- 開發版的 pipe 名稱固定，例如 `RevitController.dev`。正式版改成含 Revit 版本與程序 ID（見 [01-architecture.md](../01-architecture.md#開發版-vs-正式版)）。

### 限定目前使用者

| 框架 | 做法 |
| --- | --- |
| net8 | `PipeOptions.CurrentUserOnly`（待查證行為細節） |
| net48 | `PipeOptions.CurrentUserOnly` 可能不存在（待查證）；改用 `PipeSecurity` 設定 ACL，只允許目前使用者的 SID |

這個差異放在 `Compat/` 資料夾處理（Day 2 的規則）。

## Step 6. Sidecar 端：PipeTransport

Sidecar 透過介面與 add-in 溝通，Day 6 的 Mock 也實作同一個介面。因為 Sidecar 引用 Mock（Day 2），介面若放在 Sidecar 會造成循環引用，所以**介面放在 Core**，實作 `PipeTransport` 放在 `Sidecar/Revit/`（與 [02-tech-stack.md](../02-tech-stack.md) 目錄樹略有不同，Day 7 回填時一併更新）：

```csharp
// src/RevitController.Core/Rpc/IRevitTransport.cs
public interface IRevitTransport
{
    Task<RpcResponse> CallAsync(string method, object? @params, CancellationToken ct);
}
```

`PipeTransport` 的重點：

| 情況 | 行為 |
| --- | --- |
| 第一次呼叫 | `NamedPipeClientStream.ConnectAsync(timeout)` 建立連線 |
| Revit 沒開 / pipe 不存在 | 連線逾時，回傳明確的錯誤（例如 `RevitNotConnected`，名稱待決定），**不要無限等待** |
| 寫入或讀取時 `IOException`（Revit 被關掉） | 丟棄這條連線；下一次呼叫重新連線 |
| 同時有多個呼叫 | 目前用 `SemaphoreSlim(1)` 一次只送一筆，因為 Revit 端本來就是逐筆執行 |

「自動重連」的意思是：Revit 重開後，**下一次呼叫**會自動建立新連線，不需要重啟 Sidecar。

`id` 由 Sidecar 遞增產生，收到回應時檢查 `id` 是否相符。

## Step 7. 手動驗證

Sidecar 還沒有 MCP（Day 5），先在 `Program.cs` 加一個暫時的命令列參數：

```bash
dotnet run --project src/RevitController.Sidecar -- --ping
```

```powershell
# 預期：印出 pong 與 Revit 版本號
```

測試順序：Revit 開著 → `--ping` 成功；關閉 Revit → `--ping` 回傳連線錯誤；重開 Revit → `--ping` 再次成功。

---

## 驗收

- [ ] `RpcRequest`、`RpcResponse`、`RpcError` 與錯誤碼定義在 Core，Addin 與 Sidecar 共用
- [ ] Sidecar 呼叫 `ping` 拿到回應，內容包含 Revit 版本號（兩個 Revit 版本都測）
- [ ] 呼叫不存在的方法回傳 `-32601`
- [ ] Revit 沒開時，Sidecar 在設定的時間內回傳錯誤，不會卡住
- [ ] Revit 關閉再開啟後，不重啟 Sidecar，`ping` 仍然成功
- [ ] Pipe 只允許目前使用者連線（用另一個 Windows 帳號測試，或記錄為待驗證）
- [ ] PipeServer 迴圈中沒有任何 Revit API 呼叫

## 常見問題

**Q：Sidecar 連得上，但送出請求後一直沒有回應？**
先看 add-in log：請求有沒有被讀到？有沒有進佇列？是不是卡在 ExternalEvent（見 Day 3）？另外確認兩端都有寫入換行並 flush。

**Q：同時開兩個 Revit 會怎樣？**
開發版 pipe 名稱固定，第二個 Revit 的 PipeServer 會建立失敗或搶到連線（行為待查證）。開發期間只開一個 Revit；正式版用含程序 ID 的名稱解決。

**Q：notification（沒有 id 的請求）要支援嗎？**
格式上保留，但 M0 不使用。之後可以用來推送 job 進度。

## 學習筆記

（寫下今天的心得、卡關點、想問的問題）
