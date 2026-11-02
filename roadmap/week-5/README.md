# Week 5 · 修正建議與執行、ACC 連接器（M4）

前四週做的都是「讀」：收集元素、算碰撞、分群、判讀、匯出。第五週第一次讓 AI **改模型**：提出修正方案 → dry-run 預覽 → 人工核准 → 執行 → 可以整組 rollback。後半週加上 ACC Issues 連接器，讓碰撞和客戶的 ACC 專案雙向同步，最後做安全性檢查與 v1 發布。

> **本週內容依計畫推估。** 第五週建立在前幾週的實際成果上：`show_clash` 截圖（Day 12）、狀態追蹤與重跑比對（Day 18）、AI 判讀流程（Day 22）、Dockable Pane（Day 24 – 25）、Navisworks 互通的結論（Day 27）。開始本週前，請依 Day 21 與 Day 28 的回顧結果修訂這些文件，見 [PROGRESS.md](../PROGRESS.md) 技術債「Week 4、5 的文件內容」。

本資料夾中的程式碼都是**未編譯、未執行過的骨架**。Revit Transaction 相關 API 的細節、APS / ACC 的 endpoint 與欄位，都要對照實際版本的官方文件確認。

## 每日日程

| Day | 主題 | 文件 |
| --- | --- | --- |
| Day 29 | 修正策略設計 | [day-29.md](day-29.md) |
| Day 30 | Dry-run 與預覽 | [day-30.md](day-30.md) |
| Day 31 | 核准與執行 | [day-31.md](day-31.md) |
| Day 32 | ACC 連接器（一）：認證與專案 | [day-32.md](day-32.md) |
| Day 33 | ACC 連接器（二）：匯出與同步 | [day-33.md](day-33.md) |
| Day 34 | 安全性與穩定性檢查 | [day-34.md](day-34.md) |
| Day 35 | 收尾、v1 發布、下一階段規劃 | [day-35.md](day-35.md) |

## Week 5 Definition of Done

- [ ] 修正可以 dry-run、核准後執行、整組 rollback，且完整寫入 audit log
- [ ] 所有修改完整寫入 audit log（參數、動到的元素、執行結果、核准者）
- [ ] （客戶有使用 ACC 時）碰撞與 ACC Issues 可以雙向同步狀態
- [ ] 安全性檢查清單與異常測試結果已整理
- [ ] v1 已發布：安裝程式、使用手冊、部署文件、已知限制

前兩項是 M4 的正式驗收（SPEC §10），後三項是本週的工作產出。

## 本週新增的 MCP 工具

| 工具 | 用途 | 會不會改模型 |
| --- | --- | --- |
| `propose_fix(clash_id, strategy?)` | 產生修正方案，在 `TransactionGroup` 中試做後 rollback，回傳預覽 | 不會（dry-run） |
| `apply_fix(proposal_id)` | 執行已核准的方案 | 會，需要人工核准 |
| `undo_fix(fix_id)` | 回復一次已執行的修正（實作方式見 Day 31） | 會，需要人工核准 |
| `export_clashes("acc", …)` | 既有工具，新增 ACC 連接器 | 不會（改的是 ACC） |

`undo_fix` 的名稱與是否獨立成工具，在 Day 31 依實作方式決定。

## 本週程式碼的落點

```
src/
├── RevitController.Core/
│   ├── Models/FixProposal.cs          ← 修正方案 DTO、FixKind、FixTarget（Day 29）
│   ├── Models/FixResult.cs            ← dry-run / apply 的結果（Day 30、31）
│   └── Security/ToolPermission.cs     ← 權限分級（Day 31）
├── RevitController.Addin/
│   ├── Handlers/FixProposeHandler.cs  ← fix.propose：TransactionGroup + RollBack（Day 30）
│   ├── Handlers/FixApplyHandler.cs    ← fix.apply：Assimilate、Worksharing（Day 31）
│   └── Execution/FailureCollector.cs  ← IFailuresPreprocessor（Day 30）
├── RevitController.Sidecar/
│   ├── Mcp/FixTools.cs                ← propose_fix / apply_fix（Day 30、31）
│   ├── Approval/ApprovalStore.cs      ← 核准紀錄（Day 31）
│   ├── Connectors/Acc/                ← AccAuth、AccClient、AccConnector（Day 32、33）
│   └── Audit/                         ← 補齊修改類工具的紀錄（Day 31、34）
└── RevitController.Mock/
    └── fixtures/fix-preview.json      ← Mac 上測試 propose_fix 用（Day 30）
tests/
├── RevitController.Core.Tests/        ← FixProposal 驗證、權限判斷
└── RevitController.Sidecar.Tests/     ← 核准流程、ACC 連接器（stub HttpMessageHandler）
installer/                             ← 正式安裝程式與簽章（Day 35）
```

## 本週的學習重點

| 主題 | 內容 |
| --- | --- |
| MEP 幾何 | `LocationCurve`、`MEPCurve`、接頭（fitting）與 connector 的連接關係 |
| 移動 vs 改參數 | `ElementTransformUtils.MoveElement` 與修改 `Offset` 類參數的差異 |
| Transaction 家族 | `Transaction`、`SubTransaction`、`TransactionGroup` 的巢狀規則；`RollBack` 與 `Assimilate` |
| Failure 處理 | `IFailuresPreprocessor`、`FailuresAccessor`：warning 要記錄、error 要中止 |
| Human-in-the-loop | 核准點放在哪、核准畫面顯示什麼、核准紀錄怎麼存 |
| Worksharing | 元素擁有權、`WorksharingUtils.CheckoutElements`、借用失敗 |
| APS / ACC | OAuth 2-legged vs 3-legged、token 保存、hub → project → container → issues |
| 同步 | 雙向同步的衝突策略、rate limit、exponential backoff |
| 安全 | Threat modeling、prompt injection、冪等性 |
| 發布 | 版本號、change log、安裝程式簽章 |

## 本週的安全原則

第五週是整個專案風險最高的一週，因為工具第一次能改模型、第一次連到外部雲端服務。以下幾條從 Day 29 開始就要遵守，Day 34 再整體檢查一次：

1. **沒有核准就不改模型**（SPEC §1 非目標）。`apply_fix` 只接受 `proposal_id`，不接受 AI 直接傳入的修改內容。
2. **連結模型唯讀**（SPEC §6）。牽涉連結元素的碰撞，修正方案只能是「指派給負責專業」。
3. **憑證不進版本控制**。APS 的 client secret、token 只放在 OS 的安全儲存或環境變數，範例一律用佔位字串。
4. **每個修改都可追溯**。audit log 記錄誰提出、誰核准、改了哪些元素、改前改後的值、結果。
