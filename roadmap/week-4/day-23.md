# Day 23 · AI 判讀評估

> 本文件依計畫推估，開始前請依 Day 21 回顧結果修訂。

## 今日目標

用基準模型（Day 8）的人工標註建立固定的評估集，寫一支評估程式比對 AI 寫回的 `ai_assessment` 與正確答案，算出準確率、混淆矩陣與成本。依結果迭代判讀準則與工具回傳內容，直到達到 Day 22 定下的門檻。

## 你會學到

- 評估指標：accuracy、precision / recall、混淆矩陣，以及在碰撞檢討中哪個最重要
- 為什麼評估集要固定，改 prompt 時怎麼避免「修好一筆、壞掉三筆」
- 怎麼用與 AI 廠商無關的方式做評估：結果從 SQLite 讀，不綁定特定 harness
- 成本怎麼估：工具呼叫次數、時間、token

---

## Step 1. 建立評估集

Day 8 的標註檔記錄了每一組碰撞的正確答案（真碰撞、可接受、邊界情況）。把它轉成評估集：

```jsonc
// fixtures/eval/clash-review-eval.json（範例）
{
  "version": 1,
  "baseline_model": "基準模型的檔名與版本",
  "items": [
    { "clash_id": "a1b2…", "expected_verdict": "true_clash",   "expected_severity": "high", "note": "風管穿梁，無開口" },
    { "clash_id": "c3d4…", "expected_verdict": "acceptable",   "expected_severity": "low",  "note": "保溫層輕微接觸" },
    { "clash_id": "e5f6…", "expected_verdict": "needs_review", "expected_severity": null,   "note": "邊界情況" }
  ]
}
```

注意：

- 用穩定的 `clash_id`（Day 18）對應，所以基準模型不能在評估期間修改。
- 三種 verdict 都要有足夠的數量；只有真碰撞的評估集無法看出誤判成 `true_clash` 的情況。
- 包含 Day 22 的 prompt injection 測試元素。
- **正確答案不能經由任何工具被 AI 讀到**。不要把標註寫進 `comments` 或元素參數。

評估集定版後只能「新增」，修改既有項目要升 `version` 並說明原因。

## Step 2. 執行判讀

評估的流程：

1. 在 Revit 開啟基準模型，執行所有規則（`run_clash_test`），確認 clash_id 與評估集一致。
2. 清空上次的 `ai_assessment`（評估程式提供 `--reset` 選項，只清這個欄位）。
3. 在 harness 中載入 Day 22 的 system prompt，請它判讀所有 `new` 狀態的碰撞。
4. 判讀完成後，執行評估程式。

評估程式不呼叫 AI，只讀結果。這樣不管用哪個 harness，評估方式都一樣。

## Step 3. 評估程式

建立 `tools/RevitController.Eval/`（net8.0 console，引用 Core），從 sidecar 的 SQLite 讀出 `ai_assessment`，與評估集比對：

```csharp
// tools/RevitController.Eval/Program.cs（骨架）
var eval = EvalSet.Load(args[0]);                 // fixtures/eval/clash-review-eval.json
var store = new ResultStore(args[1]);             // sidecar 的 SQLite 路徑

var matrix = new ConfusionMatrix(new[] { "true_clash", "acceptable", "needs_review" });
foreach (var item in eval.Items)
{
    var actual = store.GetAiAssessment(item.ClashId)?.Verdict ?? "missing";
    matrix.Add(expected: item.ExpectedVerdict, actual: actual);
}

Report.Write(matrix, auditStats: AuditLog.Summarize(args[2]));   // 輸出 Markdown
```

`ConfusionMatrix` 放在 Core，可以在 Mac 上寫單元測試。

## Step 4. 讀懂指標

混淆矩陣（列 = 正確答案，欄 = AI 判讀）：

| 正確 \ AI | true_clash | acceptable | needs_review | missing |
| --- | --- | --- | --- | --- |
| true_clash | ✅ | ❌ 最危險 | ⚠️ | ❌ |
| acceptable | ⚠️ 誤報 | ✅ | ⚠️ | ❌ |
| needs_review | ⚠️ | ⚠️ | ✅ | ❌ |

| 指標 | 公式 | 在本專案的意義 |
| --- | --- | --- |
| Accuracy | 對角線 / 全部 | 整體正確率，但會被數量多的類別主導 |
| `true_clash` recall | 判成 true_clash 的真碰撞 / 所有真碰撞 | 漏報率的反面，**最重要** |
| `true_clash` precision | 真的是 true_clash / 所有判成 true_clash | 誤報多會浪費人工確認時間 |
| `needs_review` 比例 | 判成 needs_review / 全部 | 太高代表 AI 沒有減少工作量 |

真碰撞被判成 `acceptable` 是最嚴重的錯誤：人會因此跳過它。報告中要把這一格的每一筆都列出來。

## Step 5. 記錄成本

| 項目 | 資料來源 | 可信度 |
| --- | --- | --- |
| 每筆碰撞的工具呼叫次數 | Sidecar 的 audit log（Day 5） | 自己記錄的，可信 |
| 各工具的呼叫比例（特別是 `show_clash`） | Audit log | 可信 |
| 總執行時間 | Audit log 第一筆到最後一筆的時間差 | 可信 |
| Token 用量、費用 | Harness 自己的報告 | 各 harness 取得方式不同，待查證 |

報告中把成本換算成「每筆碰撞平均」，之後在 Day 28 和客戶討論時才有依據。

## Step 6. 迭代

每一輪：

1. 看混淆矩陣中錯誤的格子，逐筆看 AI 的 `reason`。
2. 把錯誤分類：準則不清楚、工具沒有回傳需要的資訊、AI 沒照流程、標註本身有問題。
3. 依分類修改：

| 錯誤原因 | 修改的地方 |
| --- | --- |
| 準則不清楚 | `prompts/clash-review/criteria.md` |
| 缺少資訊 | `get_clash_detail` 的回傳欄位 |
| 沒照流程 | `system.md` 的流程段落、工具描述 |
| 標註有問題 | 評估集（升 `version`，記錄原因） |

4. 重跑**整個**評估集，比較前後兩輪的報告。

每一輪的報告存到 `docs/ai-review/eval-reports/YYYY-MM-DD-runN.md`，並在報告開頭記錄這一輪改了什麼。AI 的輸出不完全固定，同一個設定建議至少跑兩次，看結果是否穩定。

## Step 7. 報告格式

```markdown
# AI 判讀評估 — 2026-xx-xx run 3

- 評估集版本：1（共 N 筆）
- 這一輪的修改：criteria.md 增加保溫層的體積條件
- Harness / 模型：（記錄實際使用的名稱與版本）

| 指標 | 本輪 | 上一輪 | 門檻 |
| --- | --- | --- | --- |
| true_clash recall | | | |
| accuracy | | | |
| needs_review 比例 | | | |
| 每筆平均工具呼叫 | | | — |

## true_clash 被判成 acceptable 的項目
## Prompt injection 測試元素的結果
```

---

## 驗收

- [ ] `fixtures/eval/clash-review-eval.json` 已建立，三種 verdict 都有樣本，含 injection 測試元素
- [ ] 正確答案無法經由任何 MCP 工具讀到
- [ ] 評估程式可以從 SQLite 與 audit log 產出報告，`ConfusionMatrix` 有單元測試
- [ ] 至少完成兩輪迭代，每輪都有報告與修改說明
- [ ] 達到 Day 22 的門檻；若未達到，記錄差距與原因，並在 [PROGRESS.md](../PROGRESS.md) 標記
- [ ] 能說明為什麼只看 accuracy 不夠

## 常見問題

**Q：門檻怎麼調都達不到？**
先看錯誤是不是集中在某一類。如果某一類情況連人都需要看截圖才能判斷，把它明確列為 `needs_review` 的條件，比勉強讓 AI 判斷更安全。也要回頭檢查標註是否一致。

**Q：可以為了某一筆錯誤在 prompt 裡寫特例嗎？**
盡量不要。特例會讓 prompt 越來越長，而且常常讓其他項目變差。改成通用的條件寫進準則，再跑整個評估集確認。

**Q：評估集太小怎麼辦？**
先用現有的跑通流程。之後從真實專案中抽樣、人工標註後加入，`version` 往上加。Day 25 收集的人工改判紀錄也是來源之一。

## 學習筆記

（寫下今天的心得、卡關點、想問的問題）
