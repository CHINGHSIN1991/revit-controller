# Day 31 · 核准與執行

> 本文件依計畫推估，開始前請依 Day 21 與 Day 28 回顧結果修訂。

## 今日目標

實作 `apply_fix`：只有經過人工核准的方案才能執行。執行時把修改包在 `TransactionGroup` 中，失敗就整組 rollback；成功就 `Assimilate()` 成一個 undo 項目，寫入完整的 audit log，重新檢查相關碰撞並更新狀態。另外提供 undo，讓已執行的修正可以回復。

這是 M4 驗收的核心：**修正可以 rollback，且完整寫入 audit log**。

## 你會學到

- Human-in-the-loop：核准點放在哪裡、核准畫面要顯示什麼
- 權限分級：唯讀、可修改、可執行腳本
- `Assimilate()` 與 undo 的關係
- Worksharing 環境下的元素擁有權：`WorksharingUtils.CheckoutElements` 與借用失敗

---

## Step 1. 核准流程

```
AI ──propose_fix──▶ Sidecar：存 proposal（previewed）
                     │
人 ──在 Pane 中核准──▶ Sidecar：proposal → approved，記錄核准者與時間
                     │
AI ──apply_fix(proposal_id)──▶ Sidecar 檢查已核准、未過期、未執行 ──▶ Add-in 執行
```

規則：

1. `apply_fix` **只接受 `proposal_id`**，不接受修改內容。AI 無法在核准後偷換內容。
2. 核准時記錄 proposal 的內容雜湊；執行前再比對一次，不一致就拒絕。
3. 核准有效期限（例如 30 分鐘，可設定）。模型在這段期間可能被別人改過。
4. 一個 proposal 只能執行一次（Day 34 的冪等性也靠這條）。

核准入口有兩種，v1 先做第一種：

| 入口 | 做法 | 優缺點 |
| --- | --- | --- |
| Dockable Pane | 顯示 preview，按「核准」按鈕 | 人一定在 Revit 前；最可靠 |
| Harness 的權限機制 | 由 MCP client 在呼叫 `apply_fix` 前詢問使用者 | 依 harness 而定，sidecar 無法確認真的有人看過 |

不同 harness 的確認機制不一致，所以**核准紀錄必須存在 sidecar**，不能只依賴 harness。

## Step 2. 核准畫面要顯示什麼

在 Day 25 的 Pane 加一個「待核准修正」區塊，每筆顯示：

- 碰撞摘要與 AI 的理由
- 修改內容：哪些元素、移動多少 mm、參數改前 / 改後
- 影響範圍：相連元素數量
- Dry-run 結果：原碰撞是否消失、新碰撞數量、warning
- 修改前後截圖
- 按鈕：核准、拒絕（可填理由）、在模型中顯示

人工拒絕的理由要存起來，和 Day 25 的改判紀錄一樣，作為之後調整策略的資料。

## Step 3. 權限分級

`src/RevitController.Core/Security/ToolPermission.cs`：

```csharp
public enum ToolPermission { ReadOnly = 0, Modify = 1, Script = 2 }
```

| 等級 | 可用工具 | 預設 |
| --- | --- | --- |
| `ReadOnly` | 查詢、碰撞測試、`show_clash`、匯出、`propose_fix` | 所有 token |
| `Modify` | 加上 `apply_fix`、`undo_fix` | 需在設定中開啟 |
| `Script` | 執行任意腳本（v1 不提供） | 不開放 |

權限綁在 token 上（Day 34 會改成安裝時產生），在 sidecar 的 MCP 層檢查。沒有權限的工具不要出現在工具清單中，避免 AI 反覆嘗試。這個設計要補進 SPEC §7。

## Step 4. `fix.apply` handler

```csharp
public FixResult Execute(Document doc, FixProposal p)
{
    EnsureEditable(doc, p);                              // Step 5
    using var group = new TransactionGroup(doc, $"RC fix {p.ProposalId}");
    group.Start();
    using (var tx = new Transaction(doc, "RC apply fix"))
    {
        var collector = new FailureCollector();
        AttachPreprocessor(tx, collector);
        tx.Start();
        var changes = FixApplier.Apply(doc, p);          // 回傳改前 / 改後的值
        if (tx.Commit() != TransactionStatus.Committed || collector.Errors.Count > 0)  // 列舉值待查證
        {
            group.RollBack();
            return FixResult.Failed(p, collector);
        }
        group.Assimilate();                              // 合併成一個 undo 項目
        return FixResult.Applied(p, changes, collector.Warnings);
    }
}
```

`Assimilate()` 讓使用者在 Revit 裡按一次 undo 就能回到修改前。它和 group 的 `Commit()` 的差異（undo 清單中是一項還是多項）請實測確認（待查證）。

## Step 5. Worksharing：元素擁有權

中央模型中，要修改的元素可能被別人借用。修改前先取得擁有權：

```csharp
static void EnsureEditable(Document doc, FixProposal p)
{
    if (!doc.IsWorkshared) return;
    var ids = ResolveIds(doc, p.Targets.Concat(p.AffectedUniqueIds));
    var checkedOut = WorksharingUtils.CheckoutElements(doc, ids);   // 回傳值與例外待查證
    var missing = ids.Except(checkedOut).ToList();
    if (missing.Count > 0)
        throw new RcException("ElementNotOwned", missing);           // 回傳給 AI 與使用者
}
```

| 情況 | 處理 |
| --- | --- |
| 元素被別人借用 | 回傳 `ElementNotOwned` 與擁有者（`WorksharingUtils.GetWorksharingTooltipInfo`，待查證），不執行 |
| 中央模型無法連線 | 回傳錯誤，不執行 |
| 本機副本過舊 | 提示使用者先同步（v1 不自動同步） |

這是 PROGRESS 技術債中的「Worksharing（中央模型）」，處理完後補進 SPEC。同步到中央之後 undo 是否仍可用，待實測（待查證）。

## Step 6. Audit log 與狀態更新

每次 `apply_fix` 都寫一筆完整紀錄（SPEC §7：參數、動到的元素、結果）：

```jsonc
{
  "tool": "apply_fix", "at": "...", "token_id": "...",
  "proposal_id": "...", "clash_id": "...",
  "approved_by": "<user>", "approved_at": "...",
  "changes": [ { "unique_id": "...", "param": "Offset", "old_mm": 2800, "new_mm": 2600 } ],
  "result": "applied | failed | rolled_back", "warnings": [ "..." ]
}
```

執行成功後：

1. 重算原碰撞與影響範圍（沿用 Day 30 的 `ClashRechecker`，這次結果寫入 Result Store）。
2. 原碰撞消失 → 狀態改為 `resolved`，留言記錄是哪個 proposal 修好的。
3. 產生新碰撞 → 狀態 `new`，留言註明來源。

Audit log 先寫「開始執行」，完成後再寫結果。這樣 Revit 中途當機時，log 裡也看得出有一筆沒完成（Day 34 測試）。

## Step 7. Undo

| 時機 | 做法 |
| --- | --- |
| 執行中失敗 | `TransactionGroup.RollBack()`，自動回復 |
| 執行後、使用者想回復 | `undo_fix(fix_id)`：用 audit log 中的改前值產生反向 proposal，走同樣的 `fix.apply` 流程（也需要核准） |
| 使用者直接在 Revit 按 undo | 因為有 `Assimilate()`，一次 undo 回到修改前；但 sidecar 不會知道，下次重跑碰撞時會發現 |
| `DeleteDuplicate` 執行後 | 元素已刪除，反向 proposal 無法重建。`undo_fix` 回傳明確錯誤，只能靠 Revit 本身的 undo（`Assimilate()` 的那一項）；寫進 Day 35 的已知限制 |

反向 proposal 的好處是會留下完整的 audit 紀錄。限制：如果元素在這段時間被別人改過，反向修改可能不正確，所以執行前要比對「目前值 = 當時的新值」，不一致就拒絕。

---

## 驗收

- [ ] 未核准、已過期、內容雜湊不一致的 proposal 都無法執行
- [ ] 核准成功後執行，Revit undo 清單中只多一個項目
- [ ] 執行中發生 error 時整組 rollback，模型不留下部分修改
- [ ] Audit log 記錄提出者、核准者、改前改後的值、結果
- [ ] 原碰撞的狀態自動更新為 `resolved`
- [ ] `undo_fix` 可以回復，且同樣寫入 audit log
- [ ] Worksharing 模型中，元素被別人借用時回傳 `ElementNotOwned` 而不是當掉
- [ ] SPEC 已補上權限分級與 Worksharing 處理方式

## 常見問題

**Q：AI 可以一次核准並執行多個 proposal 嗎？**
v1 不行。每個 proposal 要由人核准。之後可以考慮「核准一個群組」，但前提是同一群組的方案彼此不衝突。

**Q：核准後模型被別人改了怎麼辦？**
有效期限加上執行前的狀態比對。如果目標元素的目前值和 dry-run 時不一樣，拒絕執行，請 AI 重新 `propose_fix`。

**Q：`CheckoutElements` 成功了，但修改還是失敗？**
可能是 workset 不可編輯或元素被鎖定（待查證其他原因）。把錯誤原文寫進 audit log，帶回 troubleshooting。

## 學習筆記

（寫下今天的心得、卡關點、想問的問題）
