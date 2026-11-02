# Day 25 · Dockable Pane 互動

> 本文件依計畫推估，開始前請依 Day 21 回顧結果修訂。

## 今日目標

讓使用者在 Revit 內完成一筆碰撞的完整檢討：點選碰撞 → 跳到視圖並選取元素 → 看 AI 的判讀理由 → 接受或改判 → 匯出。所有動作都走和 MCP 工具相同的 service，確保 AI 和人看到、做到的是同一件事。

## 你會學到

- 共用邏輯：UI 與 MCP 工具呼叫同一個 service，避免兩套行為
- 人工改判要怎麼記錄，才能成為改進 prompt 的資料
- 不改動中立資料格式（SPEC §5.4）的前提下擴充功能
- 從 Pane 觸發 Revit 操作時的執行緒與錯誤處理

---

## Step 1. 點選碰撞：重用 `show_clash`

`show_clash` 在 Day 12 已經做好：建立或更新 3D 視圖、套用 section box、上色、選取元素、匯出截圖。Pane 不要另外寫一套，而是讓 sidecar 走同一條路：

```
Pane ──POST /api/clashes/{id}/show──▶ Sidecar ClashService.ShowAsync
                                         │（與 MCP 的 show_clash 相同）
                                         ▼
                                   named pipe → Add-in → ExternalEvent → ShowClashHandler
```

```csharp
// ClashService（骨架）
public async Task<ShowClashResult> ShowAsync(string clashId, ShowOptions opt, CancellationToken ct)
{
    var clash = await _store.GetAsync(clashId, ct);
    return await _revit.CallAsync<ShowClashResult>("clash.show", new { clash.Elements, clash.Viewpoint, opt }, ct);
}
```

Pane 呼叫時可以傳 `opt.ExportImage = false`：人在看畫面時不需要截圖，可以省時間。這是唯一的差異，其餘邏輯相同。

繞一圈 sidecar 看似多餘（Pane 本來就在 Revit 裡），但好處是：

- 碰撞資料（元素參照、視角）在 sidecar 的 SQLite 中，本來就要去拿。
- 只有一個 `ShowClashHandler`，修 bug 只修一次。
- Audit log 記錄到人的操作。

錯誤處理：`RevitBusy`、`ModalDialogOpen`、`LinkNotLoaded` 都要在 Pane 的狀態列顯示成人看得懂的訊息。

## Step 2. 顯示 AI 判讀

在清單的每一列與詳細區塊顯示：

| 欄位 | 來源 | 顯示方式 |
| --- | --- | --- |
| verdict | `ai_assessment.verdict` | 圖示或顏色（true_clash / acceptable / needs_review） |
| severity | `ai_assessment.severity` | 文字標籤 |
| reason | `ai_assessment.reason` | 詳細區塊中顯示全文 |
| 尚未判讀 | `ai_assessment` 為空 | 明確標示「未判讀」，不要留白 |

`reason` 來自 AI，可能間接包含模型中的文字。WPF `TextBlock` 不會執行任何內容，但如果之後改用 HTML 或 WebView 顯示，要做跳脫處理。

## Step 3. 接受與改判

兩個按鈕：

| 動作 | 結果 |
| --- | --- |
| 接受 | `status` 依 verdict 變更（例如 true_clash → `active`、acceptable → `approved`，對應規則由團隊決定），記錄「人確認了 AI 的判讀」 |
| 改判 | 跳出小表單：新的 verdict、severity、理由（必填），再依新的 verdict 變更 `status` |

兩者都呼叫 `ClashService.UpdateAsync`，也就是 MCP `update_clash` 用的同一個方法。狀態轉換規則沿用 Day 18 的狀態機，不合法的轉換由 service 拒絕，UI 不另外判斷。

## Step 4. 改判紀錄怎麼存

SPEC §5.4 的 `ClashRecord` 只有 `status`、`comments`、`ai_assessment`，沒有「人工改判」的欄位。兩個做法：

| 做法 | 內容 | 影響 |
| --- | --- | --- |
| A. 寫進 `comments` | `{ "author": "<user>", "text": "改判：acceptable → true_clash。理由：…" }`，`ai_assessment` 保留 AI 原本的結果 | 不改資料格式；但要從文字中解析，統計較麻煩 |
| B. 新增欄位 | 例如 `human_review: { verdict, severity, reason, by, at }` | 好統計；**但這是修改中立格式，要先更新 SPEC §5.4**，並確認各連接器（Excel、BCF）怎麼處理這個欄位 |

不管選哪個，原則是：**AI 的原始判讀不能被覆蓋**。改判紀錄要能回答「AI 說什麼、人改成什麼、誰、何時、為什麼」，Day 23 的評估集可以從這裡補充新樣本。

Day 18 的狀態變更歷史已經記錄誰、何時改了 `status`，可以搭配使用。決定後寫進 SPEC 或 [PROGRESS.md](../PROGRESS.md) 技術債。

## Step 5. 匯出按鈕

```csharp
// ViewModel（骨架）
var connectors = await _client.ListConnectorsAsync(ct);
ExportTargets = connectors.Where(c => c.Capabilities.HasFlag(ConnectorCapabilities.Export)).ToList();

// 使用者選擇連接器後
var result = await _client.ExportAsync(selected.Id, CurrentFilter, options, ct);
StatusMessage = $"已匯出 {result.Count} 筆到 {result.OutputPath}";
```

- 只列出有 `Export` 能力的連接器（SPEC §9）。
- 匯出範圍預設是「目前篩選條件下的碰撞」，畫面上要清楚寫出筆數，避免誤匯出全部。
- 輸出檔案的位置是 sidecar 所在的機器。目前 sidecar 和 Revit 在同一台 Windows，所以 Pane 可以提供「開啟資料夾」按鈕。

## Step 6. 回歸測試

用 Mock transport 在 sidecar 端測試 `ClashService`：

| 測試 | 預期 |
| --- | --- |
| 接受 true_clash | `status` 變為團隊定義的值，歷史有一筆紀錄 |
| 改判但沒有填理由 | 被拒絕 |
| 改判後 | `ai_assessment` 不變，改判紀錄存在 |
| 不合法的狀態轉換 | 被拒絕，回傳錯誤 |
| MCP `update_clash` 與 Local API 做同樣的事 | 資料庫結果相同 |

WPF 畫面本身在 Windows 上手動測試，把步驟記在 Day 28 的試用腳本中。

---

## 驗收

- [ ] 點選碰撞後跳到視圖並選取元素，主模型與連結模型的元素都正常
- [ ] Pane 與 MCP 的 `show_clash` 呼叫同一個 handler
- [ ] 每筆碰撞都顯示 AI 判讀，未判讀的有明確標示
- [ ] 接受與改判都會更新 `status`，且走 Day 18 的狀態機
- [ ] 改判紀錄的存放方式已決定；AI 的原始判讀不會被覆蓋
- [ ] 匯出按鈕只列出支援 Export 的連接器，並顯示將匯出的筆數
- [ ] `ClashService` 的單元測試在 Mock 上通過

## 常見問題

**Q：點了碰撞，Revit 沒反應？**
多半是 ExternalEvent 沒有執行，例如有 modal 對話框開著。確認 Pane 有收到 `ModalDialogOpen` 或 `Timeout` 並顯示。參考 [troubleshooting/external-event-not-firing.md](../troubleshooting/external-event-not-firing.md)。

**Q：連結模型的元素選不到？**
選取連結元素需要用 `Reference` 的 `CreateLinkReference`（Day 12）。確認 Pane 走的是同一個 handler，而不是另外寫的程式。

**Q：使用者一直按「接受」，沒有真的看？**
這是流程問題，不是程式問題。Day 28 試用時觀察，必要時改成必須先點開碰撞才能按接受。

## 學習筆記

（寫下今天的心得、卡關點、想問的問題）
