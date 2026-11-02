# Day 35 · 收尾、v1 發布、下一階段規劃

> 本文件依計畫推估，開始前請依 Day 21 與 Day 28 回顧結果修訂。

## 今日目標

在所有支援的 Revit 版本上做最終回歸測試，完成 M4 驗收；把部署從 PowerShell 腳本換成正式安裝程式並簽章；整理使用手冊、部署文件、已知限制；發布 v1。最後回顧整個專案，規劃下一階段。

## 你會學到

- 發布流程：版本號、change log、安裝程式簽章
- 怎麼用基準模型做跨版本的最終回歸測試
- 怎麼把碰撞檢討建立的基礎（佇列、RPC、MCP、連接器、audit log）延伸到其他 Revit 任務

---

## Step 1. 最終回歸測試

每個支援的 Revit 版本各跑一次（欄位數依 Day 1 與客戶調查決定的版本範圍調整）：

| # | 項目 | 驗證方式 | R22 | R23 | R24 | R25 | R26 |
| --- | --- | --- | --- | --- | --- | --- | --- |
| 1 | 安裝程式可安裝、可移除 | 乾淨的 Windows 上執行 | [ ] | [ ] | [ ] | [ ] | [ ] |
| 2 | M0：`get_model_overview` | 從 Mac 透過 MCP 呼叫 | [ ] | [ ] | [ ] | [ ] | [ ] |
| 3 | M1：hard clash（含連結模型）與基準答案一致 | 跑 regression 腳本 | [ ] | [ ] | [ ] | [ ] | [ ] |
| 4 | M2：重跑對應前次碰撞；BCF 可在外部工具開啟 | 重跑 + 開 BCF | [ ] | [ ] | [ ] | [ ] | [ ] |
| 5 | M3：Pane 可檢視、確認、匯出；AI 判讀達門檻 | 評估腳本 + 手動 | [ ] | [ ] | [ ] | [ ] | [ ] |
| 6 | M4：dry-run 不改模型 | Day 30 Step 7 的檢查 | [ ] | [ ] | [ ] | [ ] | [ ] |
| 7 | M4：核准後執行、整組 rollback、undo | Day 31 驗收項目 | [ ] | [ ] | [ ] | [ ] | [ ] |
| 8 | M4：audit log 完整 | 比對操作與 log | [ ] | [ ] | [ ] | [ ] | [ ] |
| 9 | ACC 同步（若有客戶使用） | 測試專案 | [ ] | — | — | — | — |
| 10 | Mock 模式測試全部通過 | Mac 上 `dotnet test` | [ ] | — | — | — | — |

ACC 與 Mock 只需在一個環境測，因為它們在 sidecar，與 Revit 版本無關。

全部打勾後，到 [PROGRESS.md](../PROGRESS.md) 把 Week 5 標記完成。

## Step 2. 版本號與 change log

| 項目 | 做法 |
| --- | --- |
| 版本號 | 在 `Directory.Build.props` 統一設定，add-in、sidecar、Core 同一個版本；採 Semantic Versioning，v1 為 `1.0.0` |
| Add-in 與 sidecar 相容性 | RPC 握手時互相回報版本，不相容就拒絕並提示 |
| Change log | `CHANGELOG.md`，依里程碑整理新增功能、已知限制 |
| Git tag | `v1.0.0`，發布前確認 working tree 乾淨 |

## Step 3. 安裝程式與簽章

這是 PROGRESS 技術債「部署：PowerShell 腳本 → 正式安裝程式與簽章」。

| 選項 | 說明 |
| --- | --- |
| WiX Toolset | 產生 MSI；適合企業派送（待評估） |
| Inno Setup | 產生 EXE；設定較簡單（待評估） |

安裝程式要做的事：

1. 偵測已安裝的 Revit 版本，只部署對應版本的 add-in（`Addins\20XX\`）。
2. 部署 sidecar，設定由 add-in 拉起（Day 34 的決定）。
3. 產生隨機 token（Day 34），顯示一次給使用者。
4. 移除時清除 add-in manifest，詢問是否保留 SQLite 資料與 audit log。

簽章：

- Add-in DLL、sidecar 執行檔、安裝程式都要用程式碼簽章憑證簽署，避免被防毒或公司政策擋下。
- 憑證的取得方式（公司既有或另外購買）與簽章工具的用法待查證。
- **私鑰與憑證密碼不得進版本控制**，簽章在受控的機器或 CI secret 中進行。
- Revit 對未簽章的 add-in 是否會跳出安全警告、各版本行為是否不同（待查證）。

## Step 4. 文件

| 文件 | 讀者 | 內容 |
| --- | --- | --- |
| 使用手冊 | BIM 協調人員 | 安裝、連接 AI、建立規則、跑碰撞、檢視與判讀、匯出、修正與核准、連接 ACC |
| 部署文件 | IT 人員 | 系統需求、網路與 port、token 管理、資料位置、升級與移除 |
| 已知限制 | 所有人 | 見下表 |
| SPEC | 開發者 | 補上 `propose_fix` / `apply_fix`、權限分級、Worksharing、§11 查證結果 |

已知限制至少整理以下來源：

- Day 14 與 Navisworks 比對時無法消除的差異
- Day 30 dry-run 實測發現的副作用
- Day 31 Worksharing 與 undo 的限制（包括 `DeleteDuplicate` 無法用 `undo_fix` 回復）
- Day 33 ACC 不支援的欄位或功能
- Day 34 異常測試中未解決的項目

## Step 5. 回填與冷啟動演練

- Day 29 – 34 的文件中，把實際跑過的結果、修正過的程式碼、查證結果回填，移除已確認的「（待查證）」。
- 在乾淨的 Windows 上，只照使用手冊與部署文件安裝並跑完一次碰撞檢討（同 Day 7 的冷啟動演練）。卡住就回去補文件。

## 整體回顧（寫下來）

**35 天下來，最有收穫的一週是哪週？為什麼？**

>

**整個專案中卡最久的地方？最後怎麼解的？**

>

**SPEC 中哪些假設後來證明是錯的？**

>

**多個 Revit 版本之間最麻煩的差異是什麼？**

>

**AI 判讀與修正建議，實際上幫使用者省下多少時間？哪裡還不夠好？**

>

**如果重新開始，哪個架構決定會改？**

>

## 下一階段規劃

v1 建立的基礎與它們在其他任務中的用途：

| 基礎 | 位置 | 延伸用途 |
| --- | --- | --- |
| ExternalEvent 佇列、分批 job | Add-in | 任何需要長時間執行的 Revit 任務 |
| Named pipe RPC | Add-in ↔ Sidecar | 新任務只要新增 RPC method |
| MCP Server、工具 schema、權限分級 | Sidecar | 新任務只要新增工具 |
| `propose → 核准 → apply` 與 `TransactionGroup` | Add-in + Sidecar | 任何會修改模型的任務 |
| 連接器介面 | Core + Sidecar | 新的外部工具 |
| Audit log、Mock、基準模型 regression | 全部 | 品質與稽核 |

候選方向（依客戶需求排序，SPEC §12 待回答的問題會影響優先順序）：

| 方向 | 內容 | 前置條件 |
| --- | --- | --- |
| 其他 Revit 任務 | 模型品質檢查（命名、參數完整性）、批次修改參數、圖紙與視圖整理、數量統計 | 沿用 propose / apply 流程 |
| BCF 3.0 | 新增連接器；依 SPEC §11 查證各工具的支援程度 | Day 19 – 20 的 BCF 2.1 實作 |
| BCF API（線上服務） | 直接與支援 BCF API 的平台同步 | 認證方式依平台而定 |
| Navisworks 直接對接 | 依 Day 27 的結論，若當時只做評估，排入實作 | Navisworks 外掛開發環境 |
| APS Design Automation | 無 UI 的雲端批次執行（SPEC §1 非目標，之後評估） | Add-in 中與 UI 無關的邏輯要能在 Design Automation 環境執行（待查證限制） |
| 碰撞計算效能 | 擷取 mesh、背景執行緒計算 | Day 21 的效能基準 |
| ACC webhook | 取代輪詢同步 | 需要對外可連線的服務，與目前的安全邊界衝突，要重新設計 |
| 多 Revit 同時連線 | Sidecar 管理多個 add-in | Day 34 的 pipe 命名 |

## 下一階段前要先想好的三件事

1. **哪個任務最值得做？** 回去問 Day 28 試用的使用者，他們最花時間的工作是什麼。
2. **v1 的實際使用數據**：哪些工具最常被呼叫、哪些方案最常被拒絕，從 audit log 統計。
3. **支援版本的調整**：新的 Revit 版本是否已發布、是否要放棄最舊的版本（SPEC §4）。

---

## 學習筆記

（寫下今天的心得、卡關點、想問的問題）
