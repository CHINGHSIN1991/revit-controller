# Week 4 · AI 判讀、Dockable Pane、Navisworks 互通（M3）

前三週做出的是一個「確定性」的碰撞引擎：規則、偵測、分群、狀態追蹤、Excel / HTML / BCF 匯出。第四週把 AI 和人帶進來：**AI 依準則判讀每一筆碰撞並寫回 `ai_assessment`，人在 Revit 內的 Dockable Pane 中確認或改判，也可以直接用自然語言下指令。** 最後一天前要對 Navisworks 的互通方式做出決定。

AI 端保持廠商中立：本專案只假設 harness 支援 MCP。各 harness 的設定方式、計費、限制都依實際選用的產品查證後再寫入。

## 本週依賴 M2 的結果

本週文件是依計畫推估寫成的，**開始前請依 Day 21 的回顧結果修訂**。特別是：

| M2 的結果 | 影響本週的哪裡 |
| --- | --- |
| 基準模型的標註品質與數量（Day 8、Day 16） | Day 23 評估集的大小與可信度 |
| 穩定 `clash_id` 是否可靠（Day 18） | 評估集能否跨次執行對應 |
| 分群效果（Day 17） | AI 是否能以群組為單位判讀 |
| 狀態機的定義（Day 18） | Day 25 接受、改判後的狀態轉換 |
| BCF 在外部工具的驗證結果（Day 20） | Day 27 Navisworks 選項 A 的起點 |
| 效能基準（Day 21） | `show_clash` 截圖的耗時，影響 AI 判讀的成本 |

## 每日日程

| Day | 主題 | 文件 |
| --- | --- | --- |
| Day 22 | AI 工作流程設計 | [day-22.md](day-22.md) |
| Day 23 | AI 判讀評估 | [day-23.md](day-23.md) |
| Day 24 | Dockable Pane 基礎 | [day-24.md](day-24.md) |
| Day 25 | Dockable Pane 互動 | [day-25.md](day-25.md) |
| Day 26 | Pane 內的 AI 對話 | [day-26.md](day-26.md) |
| Day 27 | Navisworks 互通 | [day-27.md](day-27.md) |
| Day 28 | 使用者試用、M3 驗收 | [day-28.md](day-28.md) |

## Week 4 Definition of Done

AI 判讀在評估集上達到設定門檻；使用者可以在 Revit 內完成檢視、確認與匯出

拆開來看：

- [ ] 判讀準則、system prompt、驗收門檻已寫下，門檻在評估前定案（Day 22）
- [ ] 評估集固定，評估報告顯示達到門檻（Day 23）
- [ ] Pane 可以篩選、跳視圖、接受或改判、匯出（Day 24 – 25）
- [ ] Pane 內可以和 AI 對話、看到工具呼叫、可以中斷（Day 26）
- [ ] Navisworks 互通有結論：連接器或評估報告（Day 27）
- [ ] 真實使用者試用完成，回饋已分類（Day 28）

## 本週的關鍵決定

這幾件事目前沒有定論，各天文件只列出選項與建議，做完後要回填：

| 決定 | 哪一天 | 回填到 |
| --- | --- | --- |
| 驗收門檻的數值 | Day 22 | `prompts/clash-review/criteria.md`、PROGRESS.md |
| Pane 與 sidecar 的通訊方式（建議：sidecar 開本機 HTTP API） | Day 24 | 01-architecture.md |
| 人工改判紀錄的存放方式（`comments` 或新欄位） | Day 25 | SPEC §5.4（如有新增欄位） |
| 使用哪個 harness、怎麼呼叫與中止 | Day 26 | 02-tech-stack.md、SPEC §12 |
| Navisworks 互通方式 | Day 27 | SPEC §8.2、§11 |

## 本週程式碼的落點

`prompts/`、`tools/`、`docs/` 三個頂層資料夾是本週新增的，[roadmap/README.md](../README.md) 的目錄樹之後要一併更新。

```
prompts/clash-review/
├── workflow.md                    ← 判讀流程與工具對應（Day 22）
├── criteria.md                    ← 判讀準則與驗收門檻（Day 22，Day 23 迭代）
└── system.md                      ← system prompt（Day 22）

fixtures/eval/clash-review-eval.json   ← 固定評估集（Day 23）
tools/RevitController.Eval/            ← 評估程式：讀 SQLite + audit log 產出報告（Day 23）
docs/ai-review/eval-reports/           ← 每一輪的評估報告（Day 23）

src/RevitController.Core/
├── Models/AiAssessment.cs         ← verdict / severity / reason（Day 22）
└── Evaluation/ConfusionMatrix.cs  ← 可在 Mac 上測試（Day 23）

src/RevitController.Sidecar/
├── Services/ClashService.cs       ← MCP 與 Local API 共用（Day 24，Day 25 擴充）
├── LocalApi/                      ← 127.0.0.1 的 /api/...，給 Pane 用（Day 24 – 26）
├── Harness/IHarnessClient.cs      ← 與 harness 的介面 + FakeHarnessClient（Day 26）
└── Connectors/Navisworks/         ← 只有 Day 27 決定實作時才建立

src/RevitController.Addin/UI/
├── ClashPaneProvider.cs           ← IDockablePaneProvider、DockablePaneId（Day 24）
├── ClashPane.xaml(.cs)            ← 清單、詳細、對話區（Day 24 – 26）
├── ClashPaneViewModel.cs          ← MVVM，不引用 Revit API（Day 24 – 26）
└── SidecarApiClient.cs            ← HttpClient 呼叫 Local API（Day 24）

docs/navisworks-interop.md         ← Day 27 決定寫評估報告時的產出
```

## 本週的學習重點

| 主題 | 內容 |
| --- | --- |
| Agent loop | 觀察 → 規劃 → 執行 → 驗證，對應到 `list_clashes` → `get_clash_detail` → `update_clash` |
| 輸出格式控制 | 用 JSON schema 的 enum 限制 `ai_assessment`，伺服器端再驗證一次 |
| Prompt injection | 模型中的名稱、參數、留言是資料不是指令；寫入權限越小越好 |
| 評估 | 固定評估集、混淆矩陣、`true_clash` recall 比整體 accuracy 重要 |
| Dockable Pane | `IDockablePaneProvider`、`DockablePaneId`、`RegisterDockablePane`，啟動時註冊 |
| WPF / MVVM | XAML、資料繫結、`INotifyPropertyChanged`；不加全域樣式、不引用第三方 UI 套件 |
| 共用 service | UI 與 MCP 呼叫同一段邏輯，行為一致、audit log 完整 |
| 串流與取消 | 逐行讀取事件、`Dispatcher` 切回 UI thread、`CancellationToken` 一路傳到 job |
| 技術選型 | 實驗 → 證據 → 決策矩陣（Day 27 的 Navisworks） |
| 可用性測試 | 給任務、觀察、不引導（Day 28） |

## 本週的安全議題

本週第一次讓 AI 寫入資料（`ai_assessment`），也第一次在 Revit 程序之外多開一個入口（Local API）。至少要做到：

1. **AI 的寫入權限最小化**：只能寫 `ai_assessment`，不能改 `status`；不能修改模型。
2. **模型內容當作資料**：system prompt 明說，工具回傳時放在結構化欄位；評估集包含 injection 測試元素（Day 22、Day 23）。
3. **Local API 只綁定 127.0.0.1**，需要 token，寫入 audit log 並標記來源是 `pane`（Day 24）。
4. **Harness 憑證只在 sidecar**，不進入 Revit 程序與版本控制（Day 26）。

完整的威脅分析在 Day 34。
