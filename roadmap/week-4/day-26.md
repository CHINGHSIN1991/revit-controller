# Day 26 · Pane 內的 AI 對話

> 本文件依計畫推估，開始前請依 Day 21 回顧結果修訂。

## 今日目標

在 Pane 中加入對話輸入框，使用者可以用自然語言下指令（例如「只看 3F 的風管對梁」），由 AI harness 執行。Pane 要顯示 AI 呼叫了哪些工具與結果，並且可以中斷。

AI 端保持廠商中立：本專案只要求 harness 支援 MCP，今天定義一層介面隔開，具體接哪個 harness 依查證結果決定。

## 你會學到

- Harness 的整合方式：Pane 透過 sidecar 轉送，或直接呼叫 harness
- 用 sidecar 自己的 audit 資料顯示「AI 呼叫了哪些工具」，不依賴特定 harness
- 串流回應（streaming）在 WPF 中的顯示
- 取消（`CancellationToken`）怎麼從 Pane 一路傳到 add-in 的 job

---

## Step 1. 選擇整合方式

| 選項 | 做法 | 優點 | 缺點 |
| --- | --- | --- | --- |
| A. 經由 sidecar | Pane → Local API → sidecar 的 `IHarnessClient` → harness | Harness 憑證只放在 sidecar；Pane 不需要知道是哪個 harness | 多一層轉送 |
| B. Pane 直接呼叫 harness | Pane 內建 harness 的 client | 少一層 | Add-in 要引用 harness 相關套件（違反 SPEC §3.2）；憑證進到 Revit 程序 |

建議 **A**。Harness 可能跑在 Mac、也可能跑在 Windows 上；它本身怎麼被呼叫（CLI、HTTP API、SDK）因產品而異，**全部待查證**。所以先定介面：

```csharp
// src/RevitController.Sidecar/Harness/IHarnessClient.cs（骨架）
public interface IHarnessClient
{
    // 送出一段對話，串流回傳 AI 的文字輸出
    IAsyncEnumerable<HarnessEvent> RunAsync(ChatRequest request, CancellationToken ct);
}

public sealed record ChatRequest(string SessionId, string UserMessage, string? ContextHint);
public abstract record HarnessEvent;
public sealed record TextDelta(string Text) : HarnessEvent;
public sealed record Completed(string? Summary) : HarnessEvent;
public sealed record Failed(string Message) : HarnessEvent;
```

第一個實作依 SPEC §12 開放問題 4（使用哪個 harness）的結果決定。在還沒決定前，先做一個 `FakeHarnessClient`：回傳固定文字，並實際呼叫幾個 MCP 工具，讓 Pane 可以先開發。

## Step 2. 工具呼叫的顯示：用 sidecar 自己的資料

不同 harness 回報工具呼叫的格式不同，但每一次 MCP 工具呼叫都會經過 sidecar，Day 5 起就寫進 audit log。所以 Pane 顯示「呼叫了哪些工具」不需要 harness 配合：

```
Pane ──POST /api/chat──────────▶ Sidecar 建立 chat session，交給 IHarnessClient
Pane ◀─串流 /api/chat/{id}/events── 兩種事件合併：
                                     1. harness 的文字輸出（TextDelta）
                                     2. sidecar 自己觀察到的 MCP 工具呼叫（ToolCallStarted / ToolCallFinished）
```

問題是：sidecar 怎麼知道某一筆 MCP 呼叫屬於哪個 chat session？選項：

| 方式 | 說明 |
| --- | --- |
| 專用 token | 每個 chat session 發一個短期 token 給 harness，MCP 呼叫帶這個 token 就知道是哪個 session |
| 時間區間 | Session 進行中的所有 MCP 呼叫都算進來；簡單但同時有其他 client 時會混在一起 |

專用 token 比較精確，但要確認 harness 能否在每次執行時指定 MCP 連線的 header（待查證）。先用時間區間，記錄為技術債。

## Step 3. 串流到 Pane

串流的傳輸方式：Server-Sent Events（SSE）或分塊回應的 JSON lines 都可以，選一個 net48 的 `HttpClient` 也容易讀的格式。

```csharp
// SidecarApiClient（骨架）：逐行讀取事件
using var resp = await _http.SendAsync(req, HttpCompletionOption.ResponseHeadersRead, ct);
using var reader = new StreamReader(await resp.Content.ReadAsStreamAsync());
string? line;
while ((line = await reader.ReadLineAsync()) != null)
{
    ct.ThrowIfCancellationRequested();
    var evt = Json.Deserialize<ChatEvent>(line);   // 用 Day 4 選定的 JSON 函式庫
    onEvent(evt);                                  // ViewModel 更新畫面
}
```

畫面：

| 區塊 | 內容 |
| --- | --- |
| 對話紀錄 | 使用者訊息、AI 文字（逐字附加） |
| 工具呼叫 | 每一列：工具名稱、主要參數、耗時、成功或錯誤 |
| 控制 | 送出、停止 |

更新 `ObservableCollection` 必須在 WPF UI thread 上。如果事件在背景執行緒讀取，要透過 `Dispatcher` 切回去。

## Step 4. 中斷執行

按下「停止」時要一路停下來：

```
Pane CancellationTokenSource.Cancel()
  └▶ HTTP 請求中斷 / 呼叫 POST /api/chat/{id}/cancel
       └▶ Sidecar 取消 IHarnessClient.RunAsync 的 ct（harness 端怎麼中止：待查證）
       └▶ Sidecar 取消此 session 期間啟動、仍在執行的 job（Day 11 的取消機制）
            └▶ 不再送下一批 RPC 給 add-in
```

注意：

- 已經執行完的 `update_clash` 不會回復。取消只代表「不再繼續」，Pane 要顯示停止前做了哪些事（Step 2 的工具呼叫列表）。
- Add-in 中正在 UI thread 上執行的那一批無法中途打斷，會在這一批結束後停止。
- 取消後，Pane 的狀態要回到可以輸入的狀態，不能卡在「執行中」。

## Step 5. 權限與安全

- Pane 中的對話使用和 AI 判讀相同的工具權限：本週只有唯讀工具和 `update_clash`。
- 使用者輸入的文字會原樣交給 harness。如果 Pane 會自動附上目前畫面的上下文（例如選取中的碰撞 ID、樓層），只附 ID 與結構化欄位，不要附上模型中的自由文字。
- 對話內容寫進 audit log 時，注意是否包含客戶專案的機密資訊；保存期限依 SPEC §12 開放問題 5 的結論。

## Step 6. 測試

| 測試 | 方式 |
| --- | --- |
| 串流顯示 | `FakeHarnessClient` 每 200 ms 回傳一段文字 |
| 工具呼叫顯示 | Fake 實際呼叫 `list_clashes`，確認 Pane 出現對應列 |
| 取消 | Fake 啟動 `run_clash_test` 後取消，確認 job 狀態變為取消 |
| Sidecar 未啟動 | Pane 顯示錯誤，不會卡住 Revit |
| 真實 harness | 依選定的 harness 手動測試，記錄在 Day 28 |

---

## 驗收

- [ ] `IHarnessClient` 介面與 `FakeHarnessClient` 完成，Pane 可以在沒有真實 harness 時開發與測試
- [ ] Pane 的對話可以串流顯示 AI 文字
- [ ] Pane 顯示這次對話中呼叫的 MCP 工具，資料來自 sidecar
- [ ] 按下停止後，harness 呼叫與執行中的 job 都會停止，Pane 回到可輸入狀態
- [ ] Harness 憑證只存在 sidecar，add-in 沒有新增第三方套件
- [ ] 真實 harness 的整合方式已查證並記錄；若尚未完成，列入 [PROGRESS.md](../PROGRESS.md) 技術債

## 常見問題

**Q：Harness 跑在 Mac，sidecar 在 Windows，怎麼接？**
`IHarnessClient` 的實作可以是「呼叫遠端的 harness 服務」。具體做法取決於 harness 是否提供可遠端呼叫的介面（待查證）。如果沒有，可以考慮在 Windows 上也執行一份 harness，連回本機的 MCP endpoint。

**Q：串流時畫面閃爍或很慢？**
不要每個字元都更新一次畫面。累積一小段（例如 50 ms 內的文字）再附加到 UI。

**Q：AI 在對話中說「已經修好了」？**
本週 AI 沒有任何修改模型的工具。看到這種回答要記錄下來，這是 Day 22 的 prompt 需要加強的地方（不要宣稱做了沒做的事）。

## 學習筆記

（寫下今天的心得、卡關點、想問的問題）
