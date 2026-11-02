# Revit Controller Roadmap

> **Revit Controller**：透過 AI harness（MCP）操作 Revit 的平台。第一個功能是建築各專業模型的碰撞檢討。
> C# Revit Add-in + .NET 8 Sidecar（MCP Server）+ SQLite｜互動式、多 Revit 版本

這個資料夾是整個專案的「開發與學習路線圖」。每一天都有一份文件，包含：目標、你會學到的知識點、操作步驟、程式骨架、驗收方式、常見問題。

## 文件索引

| 檔案 | 內容 |
| --- | --- |
| [../SPEC.md](../SPEC.md) | 產品規格：目標、決策紀錄、架構、碰撞引擎、MCP 工具、連接器、待驗證事項 |
| [../ROADMAP.md](../ROADMAP.md) | 35 天的總覽版（每天一段），本資料夾是它的展開版 |
| [01-architecture.md](01-architecture.md) | 各個 Process 的責任、通訊方式、執行緒模型、安全邊界 |
| [02-tech-stack.md](02-tech-stack.md) | 技術選型、開發環境、目錄結構、常用指令 |
| [week-1/](week-1/) | M0（Day 1 – 7）：環境、骨架、通訊管線 |
| [week-2/](week-2/) | M1（Day 8 – 14）：Hard clash（含連結模型）、檢視、Excel/HTML 報告 |
| [week-3/](week-3/) | M2（Day 15 – 21）：Clearance / Duplicate、忽略規則、分群、狀態追蹤、BCF 2.1 |
| [week-4/](week-4/) | M3（Day 22 – 28）：AI 判讀、Dockable Pane、Navisworks 互通 |
| [week-5/](week-5/) | M4（Day 29 – 35）：修正建議與執行、ACC 連接器 |
| [PROGRESS.md](PROGRESS.md) | 進度追蹤 checklist、卡關紀錄、技術債 |
| [learning-resources.md](learning-resources.md) | 推薦學習資源（依使用順序） |
| [troubleshooting/](troubleshooting/) | 排錯紀錄 |

## 第一週日程（M0）

| Day | 主題 | 文件 |
| --- | --- | --- |
| Day 1 | 環境建置與版本查證 | [day-01.md](week-1/day-01.md) |
| Day 2 | Solution 結構與多版本 build | [day-02.md](week-1/day-02.md) |
| Day 3 | Add-in 載入與 ExternalEvent 佇列 | [day-03.md](week-1/day-03.md) |
| Day 4 | Add-in ↔ Sidecar 的本機 RPC | [day-04.md](week-1/day-04.md) |
| Day 5 | Sidecar 的 MCP Server | [day-05.md](week-1/day-05.md) |
| Day 6 | `get_model_overview` 與 Mock transport | [day-06.md](week-1/day-06.md) |
| Day 7 | 安裝程式、雙版本驗證、M0 驗收 | [day-07.md](week-1/day-07.md) |

## 第二週日程（M1）

| Day | 主題 | 文件 |
| --- | --- | --- |
| Day 8 | 基準模型與資料模型 | [day-08.md](week-2/day-08.md) |
| Day 9 | 元素收集與幾何擷取 | [day-09.md](week-2/day-09.md) |
| Day 10 | 連結模型與座標轉換 | [day-10.md](week-2/day-10.md) |
| Day 11 | Hard clash 演算法與非同步 job | [day-11.md](week-2/day-11.md) |
| Day 12 | 查詢與檢視工具（`show_clash`） | [day-12.md](week-2/day-12.md) |
| Day 13 | 連接器介面與 Excel / HTML 連接器 | [day-13.md](week-2/day-13.md) |
| Day 14 | 與 Navisworks 比對、M1 驗收 | [day-14.md](week-2/day-14.md) |

## 第三週日程（M2）

| Day | 主題 | 文件 |
| --- | --- | --- |
| Day 15 | Clearance 與 Duplicate | [day-15.md](week-3/day-15.md) |
| Day 16 | 忽略規則 | [day-16.md](week-3/day-16.md) |
| Day 17 | 分群 | [day-17.md](week-3/day-17.md) |
| Day 18 | 狀態追蹤與重跑比對 | [day-18.md](week-3/day-18.md) |
| Day 19 | BCF 2.1 結構與視角轉換 | [day-19.md](week-3/day-19.md) |
| Day 20 | BCF 連接器與外部工具驗證 | [day-20.md](week-3/day-20.md) |
| Day 21 | 整合測試、效能基準、M2 驗收 | [day-21.md](week-3/day-21.md) |

## 第四週日程（M3）

| Day | 主題 | 文件 |
| --- | --- | --- |
| Day 22 | AI 工作流程設計 | [day-22.md](week-4/day-22.md) |
| Day 23 | AI 判讀評估 | [day-23.md](week-4/day-23.md) |
| Day 24 | Dockable Pane 基礎 | [day-24.md](week-4/day-24.md) |
| Day 25 | Dockable Pane 互動 | [day-25.md](week-4/day-25.md) |
| Day 26 | Pane 內的 AI 對話 | [day-26.md](week-4/day-26.md) |
| Day 27 | Navisworks 互通 | [day-27.md](week-4/day-27.md) |
| Day 28 | 使用者試用、M3 驗收 | [day-28.md](week-4/day-28.md) |

## 第五週日程（M4）

| Day | 主題 | 文件 |
| --- | --- | --- |
| Day 29 | 修正策略設計 | [day-29.md](week-5/day-29.md) |
| Day 30 | Dry-run 與預覽 | [day-30.md](week-5/day-30.md) |
| Day 31 | 核准與執行 | [day-31.md](week-5/day-31.md) |
| Day 32 | ACC 連接器（一）：認證與專案 | [day-32.md](week-5/day-32.md) |
| Day 33 | ACC 連接器（二）：匯出與同步 | [day-33.md](week-5/day-33.md) |
| Day 34 | 安全性與穩定性檢查 | [day-34.md](week-5/day-34.md) |
| Day 35 | 收尾、v1 發布、下一階段規劃 | [day-35.md](week-5/day-35.md) |

一天一份文件，但不綁定日曆：沒時間就順延，順序比進度表重要。第四、五週的內容會依第三週的實際結果調整。

## 文件的可信度

**本資料夾中的程式碼都是骨架，撰寫時沒有實際編譯或執行過。** 原因是撰寫時的環境沒有 Windows、沒有 Revit，Mac 上也沒有安裝 .NET SDK。

使用時請注意：

- 程式碼只示範結構與關鍵 API，不是可以直接貼上的完整實作。
- 標示「（待查證）」的地方，代表 API 名稱、行為或版本差異需要對照實際安裝版本的 RevitAPI 文件確認。
- Day 1 與 Day 2 會實際建好工具鏈。之後每完成一天，請把實際跑過的結果（版本號、指令輸出、修正過的程式碼）回填到當天文件，讓文件逐步變成「驗證過的版本」。
- 版本號一律以 Day 1 實際安裝的結果為準，填入 [02-tech-stack.md](02-tech-stack.md)。

## Repository 根目錄

```
revit-controller/            ← Git repo root
├── SPEC.md
├── ROADMAP.md
├── roadmap/                 ← 本資料夾
├── src/
│   ├── RevitController.Core/      netstandard2.0
│   ├── RevitController.Addin/     依 Revit 版本分別 build
│   ├── RevitController.Sidecar/   net8.0（MCP Server、連接器、SQLite）
│   └── RevitController.Mock/      net8.0（模擬 Add-in，供 Mac 開發與 CI）
├── tests/
├── installer/
├── fixtures/                ← 基準模型的標註檔、Mock 用的 JSON、AI 評估集
├── prompts/                 ← AI 判讀的 prompt 與準則（Week 4）
├── tools/                   ← 評估程式等開發工具（Week 4）
└── docs/                    ← 效能基準、BCF 驗證紀錄、AI 評估報告、Navisworks 互通評估
```

## 各週的 Definition of Done

| 週 | Definition of Done |
| --- | --- |
| Week 1（M0） | 從 Mac 透過 MCP 呼叫 `get_model_overview`，在至少兩個 Revit 版本上都能取得結果；Mock 讓 sidecar 測試可以在沒有 Revit 的環境中執行 |
| Week 2（M1） | 基準模型上的 hard clash（含連結模型）結果與 Navisworks 比對，差異都能解釋；可以匯出 Excel / HTML |
| Week 3（M2） | 重跑時能正確對應前次碰撞與狀態；BCF 2.1 能在至少兩個外部工具中開啟並正確定位 |
| Week 4（M3） | AI 判讀在評估集上達到設定門檻；使用者可以在 Revit 內完成檢視、確認與匯出 |
| Week 5（M4） | 修正可以 dry-run、核准後執行、整組 rollback，且完整寫入 audit log |

## 每天的工作節奏建議

1. 先讀當天文件的「今日目標」與「你會學到」。
2. 照 Step 操作，遇到卡關先看「常見問題」和 [troubleshooting/](troubleshooting/)。
3. 完成「驗收」清單，再到 [PROGRESS.md](PROGRESS.md) 打勾。
4. 把實際驗證過的結果回填到當天文件（見「文件的可信度」）。
5. 在當天文件最底下的「學習筆記」留一兩句心得。
