# Day 33 · ACC 連接器（二）：匯出與同步

> 本文件依計畫推估，開始前請依 Day 21 與 Day 28 回顧結果修訂。

## 今日目標

實作 `AccConnector : IClashConnector`：把碰撞建立成 ACC Issue（含截圖、位置、元素參照），在 `external_refs["acc"]` 記錄 Issue ID；再實作雙向同步，讓狀態與留言在兩邊保持一致，並處理兩邊同時修改的衝突。

> ACC Issues API 的欄位名稱、endpoint、rate limit 都**沒有經過驗證**，本文件只描述概念，一律以 APS 官方文件與 Day 32 的查證結果為準。

## 你會學到

- ACC Issues 的主要欄位概念：類型、狀態、指派、位置、附件
- 中立格式（SPEC §5.4）與外部系統之間的對應
- 同步衝突的處理策略
- API rate limit 與重試（exponential backoff）

---

## Step 1. 連接器骨架

`src/RevitController.Sidecar/Connectors/Acc/AccConnector.cs`：

```csharp
public sealed class AccConnector : IClashConnector
{
    public string Id => "acc";
    public string DisplayName => "Autodesk Construction Cloud Issues";
    public ConnectorCapabilities Capabilities =>
        ConnectorCapabilities.Export | ConnectorCapabilities.Import |
        ConnectorCapabilities.SyncStatus;
        // Snapshots / Viewpoints：依 Day 32 查證結果（能否附加圖片、能否帶視角）決定是否加入

    public Task<ExportResult> ExportAsync(IReadOnlyList<ClashRecord> clashes, ExportOptions o, CancellationToken ct) => ...;
    public Task<ImportResult> ImportAsync(ImportSource s, CancellationToken ct) => ...;
    public Task<SyncResult>   SyncStatusAsync(IReadOnlyList<ClashRecord> clashes, CancellationToken ct) => ...;
}
```

SPEC §8.1 的原則：**連接器只做轉換，不反過來修改核心資料模型**。ACC 特有的資訊（Issue 類型、自訂欄位）放在連接器自己的設定或 `external_refs` 中。

## Step 2. 欄位對應

| 中立格式（SPEC §5.4） | ACC Issue（欄位名稱待查證） | 方向 | 備註 |
| --- | --- | --- | --- |
| `clash_id` | 標題或自訂欄位 | → | 方便人在 ACC 中搜尋 |
| 規則名稱、類別、樓層、`grid_ref` | 標題與描述 | → | 例如「3F C-4 風管 vs 梁」 |
| `ai_assessment.reason` | 描述 | → | 標明「AI 判讀」 |
| `status` | 狀態 | ↔ | 對應表見 Step 3 |
| `assignee` | 指派對象 | ↔ | 需要 ACC 使用者或公司的 ID 對照 |
| `comments` | 留言 | ↔ | 只同步新增，不同步刪除 |
| `point` | 位置（若支援） | → | 座標系統與單位待查證 |
| `snapshot_path` | 附件 | → | 上傳流程可能分多步（待查證） |
| 元素 `unique_id` / `ifc_guid` | 描述或連結到模型（若支援） | → | 待查證 |
| — | Issue ID | ← | 存到 `external_refs["acc"]` |

建立前先檢查 `external_refs["acc"]`：已存在就更新，不重複建立。

## Step 3. 狀態對應

ACC 的狀態值待查證，先用概念設計對照表，放在設定檔讓使用者調整：

| 本系統 | ACC（概念） | 說明 |
| --- | --- | --- |
| `new` / `active` | 開啟 | |
| `resolved` | 已回答 / 待確認 | 碰撞消失，等發起人確認 |
| `approved` | 關閉 | |
| `ignored` | 關閉（附註原因） | 或不匯出，依設定 |

狀態不是一對一，所以同步時要記錄「上次同步時兩邊的值」，才能判斷是哪一邊改的。

## Step 4. 雙向同步與衝突

每筆碰撞在 SQLite 多存一份同步狀態：

```csharp
public sealed class AccSyncState
{
    public string ClashId { get; set; } = "";
    public string IssueId { get; set; } = "";
    public string LastSyncedLocalStatus { get; set; } = "";
    public string LastSyncedRemoteStatus { get; set; } = "";
    public DateTimeOffset LastSyncedAt { get; set; }
}
```

判斷邏輯：

| 本地變了 | 遠端變了 | 動作 |
| --- | --- | --- |
| 否 | 否 | 不動 |
| 是 | 否 | 推到 ACC |
| 否 | 是 | 拉回本地，寫入留言「由 ACC 同步」 |
| 是 | 是 | **衝突** |

衝突的處理策略：

| 策略 | 優點 | 缺點 |
| --- | --- | --- |
| 最後修改者優先 | 簡單 | 依賴兩邊時間戳，可能蓋掉重要變更 |
| 保留兩邊並提示使用者 | 不會遺失資訊 | 需要 UI 處理 |

v1 建議：**狀態衝突時保留兩邊並提示**（在 Pane 標示「同步衝突」），留言則兩邊合併（只新增不刪除）。

本系統重跑碰撞把狀態自動改成 `resolved` 時，也走同樣的流程推到 ACC。

## Step 5. Rate limit 與重試

```csharp
static async Task<HttpResponseMessage> SendWithRetryAsync(
    Func<HttpRequestMessage> make, HttpClient http, CancellationToken ct)
{
    for (var attempt = 0; ; attempt++)
    {
        var res = await http.SendAsync(make(), ct);
        var retryable = (int)res.StatusCode == 429 || (int)res.StatusCode >= 500;
        if (!retryable || attempt >= 5) return res;
        var delay = res.Headers.RetryAfter?.Delta
                    ?? TimeSpan.FromSeconds(Math.Pow(2, attempt) + Random.Shared.NextDouble());
        await Task.Delay(delay, ct);
    }
}
```

- 每次重試都重新建立 `HttpRequestMessage`（同一個物件不能送兩次）。
- 加隨機抖動（jitter），避免多個請求同時重試。
- 也可以改用 `Microsoft.Extensions.Http.Resilience` 或 Polly（套件選擇待決定）。
- **建立 Issue 不是冪等的**：如果請求逾時但 ACC 其實已經建立，重試會產生重複 Issue。對策：重試前先用 `clash_id` 查詢是否已存在（查詢方式待查證）。Day 34 會再檢查冪等性。

## Step 6. 匯出流程

```
export_clashes("acc", filter, options)
  → 依 filter 取出碰撞（分頁）
  → 每筆：有 external_refs["acc"] → 更新；沒有 → 建立 Issue → 上傳截圖 → 存 Issue ID
  → 回傳 ExportResult：成功數、失敗數、每筆失敗原因
```

- 長時間執行時沿用 Day 11 的 job 模型：回傳 `job_id`，AI 用 `get_job_status` 查進度。
- 一筆失敗不中斷整批，最後彙整回報。
- 每筆建立或更新都寫 audit log（外部系統的修改也要可追溯）。

## Step 7. 測試

| 測試 | 方式 |
| --- | --- |
| 欄位轉換 | 單元測試，不需要網路 |
| 狀態對應與衝突判斷 | 單元測試，列出四種情況 |
| 429 重試、5xx 重試、放棄 | stub `HttpMessageHandler` |
| 逾時後不重複建立 | stub 先逾時、再回報已存在 |
| 真實 ACC | 用測試專案手動跑一次完整流程，不要用客戶的正式專案 |

---

## 驗收

- [ ] 碰撞可以建立成 ACC Issue，含描述與截圖；Issue ID 存在 `external_refs["acc"]`
- [ ] 重複匯出不會建立重複的 Issue
- [ ] ACC 上改狀態或加留言後，同步能拉回本地
- [ ] 本地重跑碰撞自動 resolved 的結果能推到 ACC
- [ ] 兩邊同時修改時，標示為衝突而不是默默覆蓋
- [ ] 429 與 5xx 會依 backoff 重試，並遵守 `Retry-After`
- [ ] 所有對 ACC 的建立與修改都寫入 audit log

## 常見問題

**Q：ACC 上被刪除的 Issue 怎麼處理？**
本地不刪碰撞，只把 `external_refs["acc"]` 標記為失效，並在 Pane 提示。要不要重新建立由使用者決定。

**Q：一次匯出幾百筆會不會被限流？**
很可能（rate limit 數值待查證）。分批送出、控制並行數量，並在 job 進度中顯示「等待重試」。

**Q：要不要用 webhook 取代輪詢？**
Webhook 需要對外可連線的 URL，與「sidecar 不對外網開放」的安全邊界衝突。v1 用手動或定時同步；webhook 列入下一階段評估。

## 學習筆記

（寫下今天的心得、卡關點、想問的問題）
