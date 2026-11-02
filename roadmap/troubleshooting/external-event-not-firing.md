# 排錯 · ExternalEvent 的 handler 沒有執行，請求一直等到 timeout

## 症狀

Sidecar 送出 RPC 後一直等不到回應，最後回傳 `Timeout`。Add-in 的 log 顯示請求已經進佇列，也呼叫了 `Raise()`，但 handler 的 `Execute` 沒有被呼叫。

## 這代表什麼

`ExternalEvent.Raise()` 只是「請 Revit 在方便的時候執行」。Revit 在以下情況不會執行 handler：

| 情況 | 怎麼確認 |
| --- | --- |
| 有 modal 對話框開著（警告視窗、另存新檔、選項…） | 看 Revit 畫面 |
| 使用者正在進行編輯操作（例如正在畫牆、拖曳元素） | 按 Esc 結束操作後是否立刻執行 |
| Revit 正在執行其他長時間工作（開檔、同步、重新生成） | 等待後是否執行 |
| `ExternalEvent` 不是在 API context 中建立的 | 檢查是否在 `OnStartup` 或 command 內呼叫 `ExternalEvent.Create` |

## 檢查順序

### 1. `ExternalEvent.Create` 在哪裡呼叫？

必須在 Revit 的 API context 中建立，通常放在 `IExternalApplication.OnStartup`。在背景執行緒（例如 pipe 執行緒）中建立會失敗。

### 2. `Raise()` 的回傳值

`Raise()` 回傳 `ExternalEventRequest`（待查證列舉值），例如 `Accepted`、`Pending`、`Denied`、`TimedOut`。把它寫進 log：

- `Pending`：上一次的 Raise 還沒執行，這次被合併。handler 要一次處理完佇列中所有請求。
- `Denied`：事件已被 Dispose，或 Revit 拒絕。

### 3. Handler 是否一次只處理一個請求？

如果 handler 每次只從佇列取一個請求，而多次 `Raise()` 被合併成一次，剩下的請求就會卡住。正確寫法是：

```csharp
public void Execute(UIApplication app)
{
    while (_queue.TryDequeue(out var request))
    {
        // 執行並完成 request.Completion
    }
}
```

### 4. Handler 內部丟出例外

`Execute` 內的未處理例外可能讓這次執行中止，`TaskCompletionSource` 永遠不會完成。每個請求都要 try/catch，失敗時呼叫 `SetException` 或回傳錯誤。

## 設計上的預防

- 每個請求都有 timeout（Day 3）。
- Timeout 時回傳的錯誤要讓 AI 知道「可能是 Revit 有對話框開著」，請使用者檢查畫面。
- 之後可以加上偵測 modal 對話框的機制（例如 `DialogBoxShowing` 事件，待查證）。

## 學到的事

（實際遇到後在這裡記錄原因與解法）
