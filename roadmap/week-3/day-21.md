# Day 21 · 整合測試、效能基準、M2 驗收

## 今日目標

在所有支援的 Revit 版本上跑完整流程（規則 → 執行 → 分群 → 狀態更新 → 匯出），量測大型模型的效能，完成 M2 驗收。最後做一次回顧，並依本週的實際結果調整第四、五週的計畫。

## 你會學到

- 用分段計時找出碰撞流程的瓶頸
- 怎麼判斷「把幾何轉成 mesh 交給背景執行緒」（SPEC §5.3）是否值得提前做
- 用「重跑演練」檢查狀態追蹤在真實操作下是否可靠
- 怎麼把一週的實測結果變成後續計畫的調整

---

## Step 1. 部署與完整流程

用 Day 7 的腳本部署到所有支援的版本：

```powershell
./scripts/build-all.ps1
./scripts/deploy-addin.ps1 -Versions 2024,2025     # 依實際支援的版本
```

從 Mac 透過 MCP，依序呼叫（可以讓 AI harness 執行，也可以用 MCP Inspector 手動）：

```
create_clash_rule × 3（hard / clearance / duplicate，含 ignore）
run_clash_test → get_job_status（直到完成）
list_clashes(group_by: "group") → list_clashes(filter.group_id)
show_clash → update_clash（改狀態、指派、留言）
export_clashes("excel") / export_clashes("bcf21")
```

每一步的參數與結果都應該在 audit log 中查得到。

## Step 2. 效能基準

準備一個大型模型（數萬個元素；可以是基準模型放大，或公司過去的大案子，注意保密）。在 add-in 加入分段計時：

```csharp
// Addin/Diagnostics/PhaseTimer.cs（骨架）
public sealed class PhaseTimer
{
    private readonly Dictionary<string, TimeSpan> _phases = new();
    public IDisposable Measure(string phase)
    {
        var sw = Stopwatch.StartNew();
        return new Scope(() => { _phases.TryGetValue(phase, out var t); _phases[phase] = t + sw.Elapsed; }); // net48 沒有 GetValueOrDefault
    }
    public IReadOnlyDictionary<string, TimeSpan> Result => _phases;
}

using (timer.Measure("collect")) { /* FilteredElementCollector */ }
using (timer.Measure("geometry")) { /* get_Geometry + 快取 */ }
```

記錄到 `docs/benchmarks/m2.md`（只填實測值）：

| 項目 | 規則 1（hard） | 規則 2（clearance） | 規則 3（duplicate） |
| --- | --- | --- | --- |
| A 組 / B 組元素數 | | | |
| 初篩後候選組合數 | | | |
| 收集元素（collect） | | | |
| 幾何擷取（geometry） | | | |
| 初篩（broad） | | | |
| 精算 / 距離（narrow） | | | |
| 忽略規則（ignore） | | | |
| 分群（group，sidecar） | | | |
| 寫入與比對（store + diff，sidecar） | | | |
| 總時間 | | | |
| Revit 程序峰值記憶體 | | | |
| 執行期間 Revit 最長無回應時間 | | | |

- 記憶體可以用工作管理員或 `Process.GetCurrentProcess().PeakWorkingSet64`（在 add-in 中讀）。量測方式寫進文件，之後才能比較。
- 「最長無回應時間」取決於批次大小。若超過幾秒，先調小批次，再看是否需要更大的改動。
- 每個 Revit 版本各跑一次，記錄版本間的差異。

## Step 3. M2 Checklist

在**所有支援的 Revit 版本**上各跑一次：

| # | 項目 | 驗證方式 | R1 | R2 |
| --- | --- | --- | --- | --- |
| 1 | 三種碰撞類型都能執行 | 基準模型上 hard / clearance / duplicate 各一條規則 | [ ] | [ ] |
| 2 | Clearance 距離合理 | 人工抽查 3 筆 `distance_mm` | [ ] | [ ] |
| 3 | 忽略規則不漏報 | 標註的真碰撞全部被報出（Day 16 表格） | [ ] | [ ] |
| 4 | 誤報減少 | 開關忽略規則的數量比較 | [ ] | [ ] |
| 5 | 分群有效 | 群組數明顯少於碰撞數；抽查 3 個群組 | [ ] | [ ] |
| 6 | 重跑對應正確 | Step 4 演練：new / 保留 / resolved 都正確 | [ ] | [ ] |
| 7 | 不完整的執行不會誤標 resolved | 取消 job、卸載連結後重跑 | [ ] | [ ] |
| 8 | 狀態歷史完整 | 每次變更都有 actor、reason、時間 | [ ] | [ ] |
| 9 | 不合法的狀態轉換被拒絕 | 嘗試 `resolved → approved` | [ ] | [ ] |
| 10 | BCF 在至少兩個外部工具中正確定位 | Day 20 驗證表 | [ ] | — |
| 11 | 重複匯出 BCF 沿用 Topic GUID | 比對兩次匯出的 GUID | [ ] | — |
| 12 | Core / Sidecar 測試在 Mac 上全部通過 | `dotnet test` | [ ] | — |
| 13 | 效能基準已記錄 | `docs/benchmarks/m2.md` | [ ] | [ ] |

全部打勾後，到 [PROGRESS.md](../PROGRESS.md) 把 Week 3 標記完成。

## Step 4. 重跑演練

模擬真實的協調會議流程，檢查狀態追蹤在實際操作下是否可靠：

1. 執行所有規則，記下各狀態的數量。
2. 把一個群組指派給某人並改為 `active`；把兩筆改為 `approved`、一筆改為 `ignored`，各留一則留言。
3. 匯出 BCF。
4. 在 Revit 中：移動一根管線消除一個群組的碰撞、新畫一根管線製造新碰撞、刪除重建一個元素。
5. 重跑，檢查：

| 預期 | 實際 |
| --- | --- |
| 被消除的碰撞變成 `resolved`，歷史中記錄 `actor = system` 與 `job_id` | |
| 新碰撞為 `new` | |
| `approved` / `ignored` / 指派與留言都保留 | |
| 刪除重建的元素出現在 `possible_matches` | |
| 再次匯出 BCF，舊 Topic 的 GUID 不變 | |

6. 卸載一個連結模型後重跑：涉及該連結的碰撞**不能**變成 `resolved`，摘要中要有提示。

## Step 5. 回填文件

- Day 15 – 20 的文件中，把實際跑過的結果、修正過的程式碼、查證過的 API 名稱回填；把已確認的「（待查證）」標記移除或改成確認的內容。
- Day 19：BCF 元素名稱、相機欄位、單位、座標基準的實際做法。
- [SPEC.md](../../SPEC.md)：§5.2 自訂忽略規則的格式、§5.4 新增的欄位（`distance_mm`、狀態轉換表）、§11 的 BCF 與 IFC GUID 查證結果。
- 遇到的問題寫進 [troubleshooting/](../troubleshooting/)（例如 BCF 定位偏移的排查方式）。

## Step 6. 依結果調整第四、五週的計畫

第四、五週的文件是在本週之前寫的，依今天的結果檢查並修改 [week-4/](../week-4/) 與 [week-5/](../week-5/)：

| 本週的結果 | 影響 | 要修改的地方 |
| --- | --- | --- |
| Navisworks 對 BCF 的支援（Day 20） | Day 27 要走 BCF、XML 交換，還是寫外掛 | Day 27 的步驟與工時 |
| 效能基準：最長無回應時間太長，或大型模型跑太久 | SPEC §5.3「mesh 交給背景執行緒」是否要提前 | 在 Week 4 插入一天，或排入 PROGRESS.md 技術債 |
| 群組大小與數量 | AI 判讀以群組還是逐筆為單位 | Day 22 的工作流程、Day 23 的成本估算 |
| 狀態轉換表中 AI 的權限 | AI 可以改哪些狀態 | Day 22 的準則、Day 25 的「接受 / 改判」 |
| ACC 是否支援 BCF 或帶視角的 Issue（若 Day 20 有順便查） | ACC 連接器的範圍 | Day 32 – 33 |
| `ifc_guid` 與 IFC exporter 不一致 | BCF 定位不可靠 | 在進入 M3 前修正，可能要佔用 Week 4 的緩衝 |

修改時在文件開頭註明「依 Week 3 結果調整」與日期，讓之後回頭看的人知道為什麼改。

## 回顧（寫下來）

**這週哪一個功能最接近使用者真正需要的？為什麼？**

>

**Solid 最短距離最後用了哪種方法？精度夠嗎？**

>

**BCF 在外部工具中第一次開啟時，哪裡出錯？最後怎麼解的？**

>

**狀態追蹤在什麼情況下還會出錯（UniqueId 改變、連結未載入以外）？**

>

**目前最不確定的概念？（這會變成 Week 4 的學習重點）**

>

## 第四週預覽

到本週為止，平台能「找出碰撞、整理成群、追蹤狀態、匯出給別人」，但判斷每一筆是不是真的問題，還是要靠人。第四週開始讓 AI 參與判讀，並把操作搬進 Revit 介面：

| 目標 | 牽涉的層 |
| --- | --- |
| AI 判讀流程與準則（system prompt、`ai_assessment`） | AI harness、MCP 工具描述 |
| 用基準模型的標註評估 AI 準確率 | 評估腳本、fixtures |
| Dockable Pane 碰撞清單 | Add-in（WPF、`IDockablePaneProvider`） |
| Pane 內的檢視、接受 / 改判、匯出 | Add-in + Sidecar（共用 service） |
| Pane 內的 AI 對話 | Add-in、Sidecar、harness |
| Navisworks 互通 | 依 Day 20 的查證結果決定 |

詳細內容見 [week-4/README.md](../week-4/README.md)。

## Week 4 前要先想好的兩件事

1. **用哪個 AI harness 與模型？** SPEC §12 開放問題 4。Day 22 就要開始寫 prompt，Day 23 要估算成本。
2. **評估集的標註夠不夠？** AI 判讀需要「可接受」與「真碰撞」都有足夠數量的標註。若 Day 8 的標註只標了真碰撞，要先補標。

---

## 學習筆記

（寫下今天的心得、卡關點、想問的問題）
