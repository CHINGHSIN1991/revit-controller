# Day 11 · Hard clash 演算法與非同步 job

## 今日目標

實作 hard clash：先用 bounding box 初篩（broad phase），再精算（narrow phase）取得交集體積與中心點。把它包成 `run_clash_test` 非同步 job：Sidecar 管 job 狀態，分批送 RPC 給 add-in，每批在 UI thread 上只做一小段，能回報進度、能取消，結果寫入 Result Store。

## 你會學到

- Broad phase / narrow phase 的碰撞偵測概念
- `BoundingBoxXYZ`、`Outline`、`BoundingBoxIntersectsFilter`
- `ElementIntersectsSolidFilter` 的用法與限制（不能直接跨文件）
- `BooleanOperationsUtils.ExecuteBooleanOperation` 與常見的例外
- 在 ExternalEvent 中分批處理長工作，避免 UI 凍結
- Job 的狀態機：建立、執行、完成、失敗、取消

---

## Step 1. 兩階段的碰撞偵測

| 階段 | 做什麼 | 成本 | 結果 |
| --- | --- | --- | --- |
| Broad phase | 比對兩個元素的 bounding box 是否相交 | 低 | 候選配對（可能誤報，不會漏報） |
| Narrow phase | 用實際 solid 計算交集 | 高 | 真正相交的配對、交集體積、中心點 |

A 組 1,000 個、B 組 1,000 個元素，就有一百萬個配對。Broad phase 的目的是把精算的次數降到候選配對的數量。

## Step 2. Broad phase

每個元素先算出「主模型座標下」的 bounding box（連結元素要轉換，見 Day 10 Step 3）：

```csharp
// src/RevitController.Addin/Clash/HardClashEngine.cs
internal sealed class Box { public XYZ Min, Max; }

static bool Overlaps(Box a, Box b, double tolFt)
    => a.Min.X <= b.Max.X + tolFt && b.Min.X <= a.Max.X + tolFt
    && a.Min.Y <= b.Max.Y + tolFt && b.Min.Y <= a.Max.Y + tolFt
    && a.Min.Z <= b.Max.Z + tolFt && b.Min.Z <= a.Max.Z + tolFt;
```

- 元素的 bounding box：`element.get_BoundingBox(null)` 取模型範圍（傳 `null` 代表不依視圖）。
- 兩兩比對是 O(n×m)。先這樣做，Week 3 Day 21 做效能基準時再決定要不要改成空間索引（例如依軸排序後 sweep）。
- 同一文件內也可以用 `BoundingBoxIntersectsFilter(new Outline(min, max))` 讓 Revit 幫你篩，但跨連結時不能用（連結元素不在主模型的 collector 中）。

## Step 3. Narrow phase

```csharp
// 同一文件內（主模型對主模型、同一連結內部）
var filter = new ElementIntersectsSolidFilter(solidA);
var hits = new FilteredElementCollector(doc, candidateIdsB).WherePasses(filter).ToElementIds();

// 跨文件（主模型對連結、連結對連結）：兩邊的 solid 都已轉到主模型座標
Solid? inter = null;
try
{
    inter = BooleanOperationsUtils.ExecuteBooleanOperation(solidA, solidB, BooleanOperationsType.Intersect);
}
catch (Autodesk.Revit.Exceptions.InvalidOperationException ex)
{
    // 不中斷 job：記錄配對與錯誤訊息，結果標記為「待確認」
}
if (inter != null && inter.Volume > 0)
{
    var volumeMm3 = Mm.Mm3FromFeet3(inter.Volume);
    var center    = inter.ComputeCentroid();       // 內部座標，英尺
}
```

| 方式 | 優點 | 限制 |
| --- | --- | --- |
| `ElementIntersectsSolidFilter` | Revit 內部最佳化，速度快 | 只能篩 collector 所在文件的元素；只回答「有沒有相交」，沒有交集體積 |
| `ElementIntersectsElementFilter` | 直接用元素比 | 同樣不能跨文件 |
| `BooleanOperationsUtils` | 能跨文件、能算體積與中心點 | 慢；對某些幾何會丟例外 |

同一文件內用 filter 找到的配對，如果需要 `volume_mm3`，仍要對那一對再做一次 boolean 運算。是否值得，用基準模型量一下時間再決定。

## Step 4. 例外的容錯策略

`BooleanOperationsUtils` 常見失敗原因：幾何太複雜、面幾乎重合、極薄的 solid。策略：

| 選項 | 做法 | 本專案 |
| --- | --- | --- |
| 略過並記錄 | 不產生碰撞紀錄，寫進 job 的 `errors` | 可能漏報 |
| 標記待確認 | 以 bounding box 交集產生紀錄，`volume_mm3 = null`，標記 `needs_review` | **採用** |
| 重試 | 稍微縮小或放大 solid 後再算（待查證是否可行） | 之後再評估 |

錯誤數量要出現在 `get_job_status` 的結果中，Day 14 比對時會用到。

## Step 5. 分批：add-in 端

Add-in 端保存每個 job 的「工作清單」與游標，每次 ExternalEvent 只處理一批：

```csharp
// src/RevitController.Addin/Clash/ClashSession.cs
internal sealed class ClashSession
{
    public string SessionId = "";
    // ElemKey：add-in 內部用的 (文件, ElementId, link instance) 鍵，回傳前才轉成 Core 的 ClashElementRef
    public List<(ElemKey a, ElemKey b)> Candidates = new();   // broad phase 的結果
    public int Cursor;
    public bool Cancelled;
}
```

新增三個 RPC 方法（參數與回傳的 DTO 放在 `Core/Rpc/ClashMessages.cs`）：

| 方法 | 在 UI thread 上做什麼 | 回傳 |
| --- | --- | --- |
| `clash.prepare(rule)` | 收集元素、擷取幾何、broad phase | `session_id`、候選配對數、warnings（`LinkNotLoaded` 等） |
| `clash.runBatch(session_id, budget_ms)` | 從游標開始精算，超過 `budget_ms` 就停 | 本批的碰撞紀錄、新游標、錯誤 |
| `clash.cancel(session_id)` | 釋放 session | — |

- `budget_ms` 建議先設 200 ms 左右，實際值用基準模型量測後調整（使用者操作 Revit 時是否感覺卡頓）。
- `clash.prepare` 本身也可能很久（大量元素的幾何擷取），必要時也要拆批（待評估）。
- 和 Week 1 一樣：pipe 執行緒只負責收發，**不碰 Revit API**；所有計算都在 handler 裡。

## Step 6. Job 管理：Sidecar 端

```csharp
// src/RevitController.Sidecar/Jobs/JobManager.cs
public sealed class JobManager
{
    public string Start(IReadOnlyList<string> ruleIds) { /* 建 job、回傳 job_id、背景執行 RunAsync */ return ""; }
    public JobStatus Get(string jobId) => throw new NotImplementedException();
    public void Cancel(string jobId) { /* 設 CancellationToken，不再送下一批 */ }

    async Task RunAsync(Job job, CancellationToken ct)
    {
        // foreach rule：
        //   prepare → while (cursor < total && !ct.IsCancellationRequested) runBatch
        //   每批：計算 clash_id（Day 8）、寫入 ResultStore、更新 processed
        // 結束：Completed / Failed / Cancelled
    }
}
```

`get_job_status` 回傳範例：

```jsonc
{ "job_id": "j_01", "state": "running", "processed": 1200, "total": 4800,
  "clash_count": 37, "errors": 2, "warnings": ["LinkNotLoaded: MEP-B"] }
```

| 狀態 | 進入條件 |
| --- | --- |
| `queued` | 建立後 |
| `running` | 第一個 `clash.prepare` 送出 |
| `completed` | 所有規則跑完 |
| `failed` | RPC 錯誤（例如 Revit 關閉、`NoActiveDocument`） |
| `cancelled` | 使用者或 AI 取消 |

Revit 在 job 執行中關閉時，add-in 的 session 會消失。Sidecar 收到 RPC 錯誤後把 job 標記為 `failed`，不要自動重跑。

## Step 7. Mock fixture

在 `fixtures/mock/clash/` 放一組假的 `clash.prepare` / `clash.runBatch` 回應，讓 Mock transport 能模擬「三批完成、其中一批有錯誤」。這樣 Sidecar 的 job 邏輯（進度、取消、錯誤）可以在 Mac 上寫單元測試。

---

## 驗收

- [ ] 基準模型的一條規則（主模型對連結）能跑完，碰撞數量和 `expected-clashes.json` 大致吻合（差異 Day 14 再分析）
- [ ] 連結對連結的規則能跑完
- [ ] 執行中拖動 Revit 視圖不會明顯卡頓
- [ ] `get_job_status` 的進度會增加；`cancel` 之後不再送下一批
- [ ] Boolean 運算失敗不會中斷 job，數量出現在 `errors`
- [ ] Mock 模式下 JobManager 的單元測試通過（含取消與錯誤）

## 常見問題

**Q：碰撞數量比預期多很多？**
常見原因：同一對元素被多個 solid 重複計算（一個元素有多個 solid 時，要在元素層級合併）、A/B 組類別重疊時同一對算了兩次（`ClashId` 排序可以去重）、相接觸（交集體積極小）的元素也被算進來。容差與最小體積門檻留到 Day 14 根據比對結果決定。

**Q：`clash.runBatch` 回傳太大？**
每批只回傳新的碰撞紀錄，不回傳整個清單。單筆紀錄也只放 SPEC §5.4 的必要欄位。

**Q：可以用多執行緒加速嗎？**
Revit API 物件不能在背景執行緒使用。見 Day 9 常見問題。

## 學習筆記

（寫下今天的心得、卡關點、想問的問題）
