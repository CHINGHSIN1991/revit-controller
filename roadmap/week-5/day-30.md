# Day 30 · Dry-run 與預覽

> 本文件依計畫推估，開始前請依 Day 21 與 Day 28 回顧結果修訂。

## 今日目標

實作 `propose_fix`：在 `TransactionGroup` 中試做 Day 29 定義的修改，重新檢查相關碰撞，記錄結果與截圖後整組 rollback。AI 可以拿到「修改後會怎樣」的預覽，但模型完全不會被改動。

## 你會學到

- `Transaction`、`SubTransaction`、`TransactionGroup` 的差異與巢狀規則
- `RollBack` 與 `Assimilate` 的意義
- `IFailuresPreprocessor` 與 `FailuresAccessor`：如何收集修改時跳出的 warning 和 error
- Dry-run 的限制：哪些副作用在 rollback 之後可能還在

---

## Step 1. Transaction 家族

| 類別 | 用途 | 規則 |
| --- | --- | --- |
| `Transaction` | 一次對模型的修改，會產生一個 undo 項目 | 修改模型一定要在開啟的 Transaction 內 |
| `SubTransaction` | Transaction 內部的一小段，可以單獨 rollback | 必須在已開啟的 Transaction 內使用 |
| `TransactionGroup` | 把多個 Transaction 包成一組 | 結束時可以 `RollBack()`（全部取消）或 `Assimilate()`（合併成一個 undo 項目） |

巢狀關係：`TransactionGroup` ⊃ `Transaction` ⊃ `SubTransaction`。細節（例如 group 可否巢狀、`Commit` 與 `Assimilate` 的差異）請對照 RevitAPI 文件確認（待查證）。

Dry-run 的做法：

```
TransactionGroup.Start()
  Transaction.Start() → 套用修改 → Commit()     ← 必須 commit，Revit 才會 regenerate
  重新檢查碰撞、擷取截圖、讀取 warning
TransactionGroup.RollBack()                        ← 整組取消，模型回到原狀
```

為什麼內層要 `Commit()` 而不是直接 `RollBack()`：commit 後模型才會完成 regenerate，重算碰撞時拿到的是修改後的幾何。外層 group 的 `RollBack()` 會把已 commit 的內層一起取消。

## Step 2. 收集 warning 與 error

修改管線常會跳出「元素未連接」「重疊」等 warning。互動模式下 Revit 會彈對話框，卡住 ExternalEvent。用 `IFailuresPreprocessor` 在 commit 時攔下來：

```csharp
// src/RevitController.Addin/Execution/FailureCollector.cs（骨架）
public sealed class FailureCollector : IFailuresPreprocessor
{
    public List<string> Warnings { get; } = new();
    public List<string> Errors { get; } = new();

    public FailureProcessingResult PreprocessFailures(FailuresAccessor fa)
    {
        foreach (var msg in fa.GetFailureMessages())
        {
            var text = msg.GetDescriptionText();
            if (msg.GetSeverity() == FailureSeverity.Warning)
            {
                Warnings.Add(text);
                fa.DeleteWarning(msg);          // 不讓對話框跳出（待查證）
            }
            else Errors.Add(text);
        }
        return Errors.Count > 0
            ? FailureProcessingResult.ProceedWithRollBack   // 待查證列舉值
            : FailureProcessingResult.Continue;
    }
}
// FailuresAccessor 的方法名稱、FailureSeverity 與 FailureProcessingResult 的列舉值都待查證
```

掛到 Transaction 上：

```csharp
var opts = tx.GetFailureHandlingOptions();
opts.SetFailuresPreprocessor(collector);
tx.SetFailureHandlingOptions(opts);
// 方法名稱與呼叫時機（Start 前或後）待查證
```

原則：**warning 記錄並回報，error 一律中止**。不要自動解決 error（例如自動刪除元素），那等於讓程式替使用者做了沒核准的決定。

## Step 3. `fix.propose` handler

`src/RevitController.Addin/Handlers/FixProposeHandler.cs`，在 UI thread 上執行（透過 Day 3 的 ExternalEvent 佇列）：

```csharp
public FixPreview Execute(Document doc, FixProposal p)
{
    var preview = new FixPreview { ProposalId = p.ProposalId };
    preview.BeforeSnapshot = Snapshot(doc, p);          // 沿用 show_clash 的截圖邏輯

    using var group = new TransactionGroup(doc, "RC dry-run");
    group.Start();
    try
    {
        using (var tx = new Transaction(doc, "RC apply fix (dry-run)"))
        {
            var collector = new FailureCollector();
            AttachPreprocessor(tx, collector);
            tx.Start();
            FixApplier.Apply(doc, p);                   // Day 31 與 apply 共用
            var status = tx.Commit();
            preview.Warnings = collector.Warnings;
            preview.Errors = collector.Errors;
            preview.Committed = status == TransactionStatus.Committed;  // 列舉值待查證
        }
        if (preview.Committed)
        {
            preview.RecheckResult = ClashRechecker.Run(doc, p);  // 只重算相關規則與範圍
            preview.AfterSnapshot = Snapshot(doc, p);
        }
    }
    finally
    {
        group.RollBack();                               // 不論成功與否都回復
    }
    return preview;
}
```

- `FixApplier.Apply` 是唯一真正改模型的程式碼，Day 31 的 `apply_fix` 也呼叫它。這樣 dry-run 和正式執行的行為一致。
- 在開著的 group 中擷取截圖是否可行、是否需要先切換視圖，待實測（待查證）。不行的話，改成只回傳重算結果，截圖留給 Day 31 執行後再拍。

## Step 4. 重新檢查碰撞

`ClashRechecker` 不需要重跑整個模型，只要：

1. 原本的碰撞是否消失（用 Day 11 的精算邏輯，只算這一對元素）。
2. 影響範圍內的元素（`AffectedUniqueIds`）在**所有啟用的規則**下，是否產生新碰撞。

回傳格式：

| 欄位 | 說明 |
| --- | --- |
| `original_resolved` | 原碰撞是否消失 |
| `new_clashes` | 新產生的碰撞（只列摘要，不寫入 Result Store） |
| `warnings` / `errors` | Step 2 收集的訊息 |
| `before_snapshot` / `after_snapshot` | 截圖路徑 |

新碰撞**不寫入 Result Store**，因為模型已經 rollback，它們不存在。

## Step 5. Sidecar 的 `propose_fix` 工具

```csharp
// src/RevitController.Sidecar/Mcp/FixTools.cs（骨架；屬性名稱依 Day 5 選定的 MCP SDK）
[McpTool("propose_fix")]
public async Task<FixPreviewDto> ProposeFix(string clashId, string? strategy, CancellationToken ct)
{
    var proposal = await _planner.BuildAsync(clashId, strategy, ct);  // 產生 FixProposal
    var errors = FixProposalRules.Validate(proposal).ToList();
    if (errors.Count > 0) return FixPreviewDto.Rejected(proposal, errors);

    var preview = await _revit.CallAsync<FixPreview>("fix.propose", proposal, ct);
    proposal.Status = "previewed";
    await _store.SaveProposalAsync(proposal, preview, ct);
    await _audit.WriteAsync("propose_fix", new { clashId, proposal.ProposalId }, preview, ct);
    return FixPreviewDto.From(proposal, preview);
}
```

工具描述要寫清楚「這個工具不會修改模型」，並把 Day 29 修正策略文件的重點摘進去。

## Step 6. Mock 與測試

在 `RevitController.Mock/fixtures/fix-preview.json` 放一份預錄的 preview。Mac 上可以測：

- 驗證失敗的方案（連結元素）不會送到 add-in
- 成功的 preview 會存進 SQLite，狀態為 `previewed`
- audit log 有寫入

## Step 7. 驗證 rollback 真的乾淨

Dry-run 的風險是「以為沒改，其實有改」。在 Windows 上用基準模型實測：

| 檢查 | 方法 |
| --- | --- |
| 元素位置與參數回到原值 | dry-run 前後各讀一次，逐欄比對 |
| Undo 清單沒有多出項目 | 手動查看 Revit 的 undo 下拉選單 |
| 文件沒有被標成已修改 | 關閉時是否詢問儲存（待查證是否受 rollback 影響） |
| 視圖、選取狀態 | 截圖用的 3D 視圖是否殘留（若在 group 外建立就會殘留） |
| Worksharing 擁有權 | 是否因為試做而借用了元素（待查證） |

把發現的副作用寫進已知限制，Day 35 整理。

---

## 驗收

- [ ] `propose_fix` 回傳 preview：原碰撞是否消失、新碰撞、warning、截圖
- [ ] Dry-run 後模型的元素位置、參數與 undo 清單都沒有改變
- [ ] Warning 不會跳出對話框卡住 ExternalEvent；error 會讓整組 rollback
- [ ] 驗證失敗的方案（連結元素）在 sidecar 端就被擋下
- [ ] Mock 模式下 `propose_fix` 的測試在 Mac 上通過
- [ ] 能說明為什麼內層 Transaction 要先 `Commit()` 再由 group `RollBack()`

## 常見問題

**Q：為什麼不用 `SubTransaction` 做 dry-run？**
`SubTransaction` 必須在已開啟的 Transaction 內，而且 rollback 的範圍只到那一段。用 `TransactionGroup` 包住完整的 Transaction，最接近正式執行時的狀態，也最容易保證整組回復。

**Q：Dry-run 很慢怎麼辦？**
多半是 regenerate 與重算碰撞。只重算影響範圍內的元素；一次 `propose_fix` 只試一個方案，AI 要比較多個方案時分次呼叫。

**Q：Rollback 後 Revit 還是顯示「已修改」？**
可能是截圖時建立了視圖或改了視圖設定（在 group 外）。把截圖用的視圖操作也放進 group，或接受這個副作用並寫進已知限制。

## 學習筆記

（寫下今天的心得、卡關點、想問的問題）
