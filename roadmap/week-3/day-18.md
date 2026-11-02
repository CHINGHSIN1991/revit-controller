# Day 18 · 狀態追蹤與重跑比對

## 今日目標

這是 M2 驗收的核心：**重新執行時能正確對應上次的碰撞**。

- 重跑時用穩定的 `clash_id` 與前次結果比對：新增的標為 `new`，仍存在的保留原狀態，消失的自動標為 `resolved`（SPEC §5.4）。
- 實作 `update_clash`：狀態、負責人、留言。
- 記錄狀態變更歷史：誰、何時、從什麼變成什麼、為什麼。

今天結束時，連續執行兩次（中間修改模型），狀態變化正確。

## 你會學到

- 狀態機的設計：合法轉換表、誰可以觸發
- 重跑比對的三種結果，以及「不完整的執行」為什麼不能把碰撞標成 resolved
- 元素刪除重建時 UniqueId 會改變，對穩定 ID 的影響
- 稽核需求：歷史紀錄要包含哪些欄位

---

## Step 1. 狀態與合法轉換

狀態值固定為 SPEC §5.4 的五個：`new | active | resolved | approved | ignored`。

| 從 \ 到 | new | active | resolved | approved | ignored |
| --- | --- | --- | --- | --- | --- |
| new | — | 人 / AI | 系統（重跑消失）/ 人 | 人 | 人 |
| active | ✗ | — | 系統 / 人 | 人 | 人 |
| resolved | ✗ | 系統（重跑又出現） | — | ✗ | ✗ |
| approved | ✗ | 人（重新打開） | 系統（重跑消失） | — | 人 |
| ignored | ✗ | 人（重新打開） | 系統（重跑消失） | 人 | — |

- `approved`：確認是碰撞，但可以接受（例如保溫層輕微接觸）。
- `ignored`：使用者判定不需要處理。和 Day 16 的忽略規則不同，那些組合根本不會產生紀錄。
- **AI 能做哪些轉換**是 Day 22 的設計題。v1 先限制 AI 只能 `new → active` 與寫留言、`ai_assessment`，`approved` / `ignored` 需要人工（待 Day 22 確認）。
- SPEC §5.4 只寫了 `New → Active → Resolved / Approved / Ignored`。表中允許人直接從 `new` 跳到 `approved` / `ignored` / `resolved`，是對 SPEC 的延伸，也是設計決定，要寫進 SPEC。
- 表中「重跑又出現」回到 `active`，不是 `new`，因為這筆碰撞以前就被看過。這是一個設計決定，寫進 SPEC。

```csharp
// Core/Models/ClashStatus.cs
public enum ClashStatus { New, Active, Resolved, Approved, Ignored }
public enum Actor { System, Human, Ai }

public static class ClashTransitions
{
    public static bool IsAllowed(ClashStatus from, ClashStatus to, Actor actor) { /* 查上表 */ }
}
```

這段在 Core，用表格驅動的單元測試把每一格都測一次。

## Step 2. 重跑比對

```csharp
// Core/Clash/RerunDiff.cs
public sealed record RerunResult(
    IReadOnlyList<ClashRecord> Added,      // 這次有、上次沒有 → new
    IReadOnlyList<ClashRecord> Kept,       // 兩次都有 → 保留狀態，更新幾何欄位
    IReadOnlyList<ClashRecord> Disappeared // 上次有（且不是 resolved）、這次沒有 → resolved
);

public static RerunResult Diff(IEnumerable<ClashRecord> previous, IEnumerable<ClashRecord> current, RunScope scope)
{
    // 以 clash_id 為 key 做集合運算
    // 只比對 scope 涵蓋的 rule_id；其他規則的紀錄不動
}
```

`Kept` 的處理：

| 欄位 | 處理 |
| --- | --- |
| `status`、`assignee`、`comments`、`ai_assessment`、`external_refs` | 保留舊值 |
| `point`、`volume_mm3`、`distance_mm`、`level`、`grid_ref` | 用新值（幾何可能小幅改變） |
| `group_id` | 依 Day 17 的規則重新計算；手動指定的保留 |
| `snapshot_path` | 標記為過期，下次 `show_clash` 時重產 |

## Step 3. 什麼情況不能自動標成 resolved

「這次沒找到」不一定代表「已經解決」。以下情況**不能**把碰撞標成 `resolved`：

| 情況 | 原因 | 處理 |
| --- | --- | --- |
| 這次只跑了部分規則 | 其他規則根本沒執行 | `RunScope` 只包含這次的 `rule_ids` |
| Job 被取消或失敗 | 結果不完整 | 不做比對，或只比對已完成的批次（建議：不比對） |
| 連結模型未載入 | 元素不存在只是因為沒載入 | 涉及該 `link_instance_uid` 的碰撞保持原狀態，並在摘要中提示（SPEC §6：不得略過） |
| 規則內容被修改 | 舊結果不再可比 | 規則加上版本或內容雜湊；規則改變時提示使用者，不自動 resolve |

這張表就是 Day 21 整合測試的反例清單。

## Step 4. UniqueId 改變的影響

`clash_id = hash(rule_id, ref_a, ref_b)`，元素以 `(link_instance_uid, unique_id)` 識別。元素被刪除再重畫、或某些操作（例如複製貼上取代原元素）後，UniqueId 會改變：

- 結果：舊的碰撞變成 `resolved`，同一個位置出現一筆新的 `new`。狀態、負責人、留言不會延續。
- 今天**不解決這個問題**，只做到兩件事：
  1. 在文件與工具描述中說明這個限制。
  2. 在重跑摘要中偵測「疑似同一筆」：消失的與新增的碰撞，若規則相同、位置接近（例如 < 100 mm）、類別相同，列為 `possible_matches` 給使用者或 AI 確認。
- `link_instance_uid` 也一樣：連結被移除再重新加入時，所有相關碰撞都會換 ID（待實測確認）。

## Step 5. 狀態歷史與 `update_clash`

```sql
-- Sidecar/Store/Migrations/003_status_history.sql
CREATE TABLE clash_status_history (
  id         INTEGER PRIMARY KEY AUTOINCREMENT,
  clash_id   TEXT NOT NULL,
  from_status TEXT,
  to_status  TEXT NOT NULL,
  actor_type TEXT NOT NULL,      -- system | human | ai
  actor_id   TEXT,               -- 使用者名稱或 harness / session 識別
  reason     TEXT,
  job_id     TEXT,               -- 系統轉換時記錄是哪次執行
  at         TEXT NOT NULL       -- ISO 8601 UTC
);
```

`update_clash` 的參數（對應 SPEC §7）：

```jsonc
{
  "clash_id": "…",
  "status": "active",            // 選填；不合法的轉換回傳錯誤，附上允許的目標狀態
  "assignee": "MEP-王",           // 選填
  "comment": "已通知機電調整高程", // 選填，附加到 comments
  "reason": "…"                  // 狀態變更時必填
}
```

- 狀態變更與歷史寫入要在**同一個 SQLite transaction** 中。
- `update_clash` 也接受 `group_id`：一次更新整群（例如整群指派給同一人），每筆都寫歷史。
- 每一次呼叫照常寫入 audit log（Day 5）。歷史表是業務資料，audit log 是操作紀錄，兩者不互相取代。

---

## 驗收

- [ ] `ClashTransitions` 的單元測試覆蓋轉換表的每一格（Mac 可跑）
- [ ] `RerunDiff` 的單元測試：新增、保留、消失、部分規則、取消的 job、未載入的連結
- [ ] 實際演練：執行 → 把幾筆改成 `active` / `approved` 並留言 → 在 Revit 中移動一根管線消除一處碰撞、製造一處新碰撞 → 重跑 → 狀態正確
- [ ] 刪除重建一個元素後重跑，摘要中有 `possible_matches`
- [ ] 每次狀態變更都有歷史紀錄，包含 actor 與 reason
- [ ] 不合法的轉換被拒絕，錯誤訊息列出允許的狀態

## 常見問題

**Q：使用者想要把 `resolved` 改回去？**
依轉換表不允許。如果是系統誤判（例如連結沒載入），應該修正 Step 3 的判斷；真的需要時，由重跑讓它「重新出現」回到 `active`。

**Q：兩個人同時更新同一筆？**
SQLite 寫入是序列化的，後寫的覆蓋先寫的，但兩次都有歷史。若要防止覆蓋，可以要求帶上 `expected_status`，不一致時回傳衝突（Day 33 的 ACC 同步會再遇到這個問題）。

**Q：狀態要不要寫回 Revit 模型（ExtensibleStorage）？**
SPEC §12 開放問題 5。v1 存在 sidecar 的 SQLite，換電腦或換人時要另外處理，記在 PROGRESS.md。

## 學習筆記

（寫下今天的心得、卡關點、想問的問題）
