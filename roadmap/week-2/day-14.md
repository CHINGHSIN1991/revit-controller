# Day 14 · 與 Navisworks 比對、M1 驗收

## 今日目標

用相同的規則在 Navisworks Clash Detective 跑基準模型，與本平台的結果逐筆比對，分析差異並修正 bug；無法消除的差異要記錄原因。最後在兩個 Revit 版本上跑完 M1 checklist，寫出 M1 驗收報告，做一次回顧，並讀懂第三週要做什麼。

## 你會學到

- Navisworks Clash Detective 的基本操作：Selection Sets、測試類型、容差
- Navisworks 的結果與 Revit solid 運算結果為何會不同（tessellated mesh vs solid）
- 怎麼把比對差異整理成可重現的測試案例
- 怎麼把一週的問題整理成技術債與下週重點

---

## Step 1. 部署

沿用 Day 7 的 `scripts/build-all.ps1` 與 `scripts/deploy-addin.ps1`，把本週的 add-in 部署到兩個 Revit 版本。Sidecar 要重新啟動，確認 migration（Day 8）自動執行。

部署後先跑一次冒煙測試：從 Mac 呼叫 `get_model_overview` → `list_clash_rules`，確認管線正常。

## Step 2. 在 Navisworks 跑同樣的規則

1. 從 Revit 匯出每個模型的 NWC（Navisworks exporter 的位置與設定依版本而定，待查證），或直接在 Navisworks 開啟 `.rvt`（是否支援依版本而定，待查證）。
2. 在 Navisworks Manage 中 Append 所有模型。確認連結模型的位置和 Revit 中一致（座標設定見 Day 10 Step 6）。
3. 依 `ClashRule` 的 `set_a` / `set_b` 建立 Selection Sets（或 Search Sets，依類別篩選）。
4. 在 Clash Detective 建立測試，類型與容差對應如下：

| 本平台 | Navisworks 測試類型 | 說明 |
| --- | --- | --- |
| `hard`、`tolerance_mm = 0` | Hard | 以三角面判斷是否穿插 |
| — | Hard (Conservative) | 較保守，可能回報更多「可能相交」的配對 |
| `clearance`（Week 3） | Clearance | |
| `duplicate`（Week 3） | Duplicates | |

5. 匯出結果報告（XML 或 HTML，格式待查證），確保報告中有能對應回 Revit 的識別資訊（Element ID、UniqueId 或 IFC GUID，哪些欄位可用待查證）。

把 Navisworks 版本、匯出設定、測試設定都記錄下來，之後重跑才能重現。

## Step 3. 逐筆比對

寫一個小工具（放在 `tests/` 或 `scripts/`，Mac 上可以跑），輸入三份資料：

| 輸入 | 來源 |
| --- | --- |
| 本平台結果 | `export_clashes` 或直接讀 SQLite |
| Navisworks 結果 | Step 2 的報告 |
| 人工標註 | `fixtures/baseline/expected-clashes.json`（Day 8） |

以「兩個元素的參照」為 key 做配對（順序無關），輸出四類：

| 類別 | 意義 |
| --- | --- |
| 兩邊都有 | 一致 |
| 只有本平台有 | 可能是本平台誤報，或 Navisworks 漏報 |
| 只有 Navisworks 有 | 可能是本平台漏報 |
| 都沒有，但標註有 | 兩邊都漏報，優先檢查 |

如果 Navisworks 報告只有 `ElementId`，要加上檔案名稱才能唯一識別（同一個 `ElementId` 在不同檔案中代表不同元素）。

## Step 4. 分析差異

每一筆差異都要歸到一個原因：

| 原因 | 典型現象 | 處理 |
| --- | --- | --- |
| 幾何精度 | Navisworks 用三角化的 mesh，曲面（圓管）被近似；本平台用 solid | 記錄，不消除；考慮最小體積門檻 |
| 容差 | 交集極小（接觸）的配對，一邊算、一邊不算 | 決定預設的最小交集體積或穿透深度 |
| 詳細等級 | 匯出 NWC 的詳細等級與 Day 9 的 `ViewDetailLevel.Fine` 不同 | 對齊設定後重跑 |
| 族群幾何 | 巢狀族群、隱藏的族群幾何、空 solid 的處理不同 | 檢查 Day 9 的幾何展開 |
| 連結 | 某個連結 instance 沒算到，或座標錯位 | 檢查 Day 10；這是 bug |
| 忽略規則 | Navisworks 預設忽略同一檔案或同一群組內的碰撞（待查證） | 記錄，Week 3 Day 16 處理 |
| Boolean 失敗 | 本平台標記為待確認 | 對照 Day 11 的 `errors` |
| 本平台的 bug | 以上都不是 | 修正 |

每一類挑一筆，做成**可重現的測試案例**：記錄兩個元素、規則、預期結果與原因。能用 Mock fixture 重現的放進 `tests/`；需要 Revit 的寫進 `fixtures/baseline/` 的標註檔，加上 `kind: edge_case` 與說明。

## Step 5. M1 Checklist

在**兩個 Revit 版本**上各跑一次：

| # | 項目 | 驗證方式 | R1 | R2 |
| --- | --- | --- | --- | --- |
| 1 | 規則可以建立、列出、刪除 | MCP 呼叫三個規則工具 | [ ] | [ ] |
| 2 | 主模型對連結的 hard clash | `run_clash_test` → `get_job_status` 完成 | [ ] | [ ] |
| 3 | 連結對連結、同一連結放置兩次 | 同上，檢查兩個 instance 各自有結果 | [ ] | [ ] |
| 4 | 未載入的連結被回報 | 卸載一個連結後執行 | [ ] | [ ] |
| 5 | 執行中 Revit 不會凍結、可以取消 | 執行中操作視圖、呼叫取消 | [ ] | [ ] |
| 6 | `list_clashes` 篩選與分頁 | 依樓層、規則篩選 | [ ] | [ ] |
| 7 | `get_clash_detail` 有 `level`、`grid_ref`、`ifc_guid` | 抽查 3 筆 | [ ] | [ ] |
| 8 | `show_clash` 回傳的截圖看得到兩個元素 | 抽查 3 筆，含 1 筆連結元素 | [ ] | [ ] |
| 9 | Excel / HTML 匯出 | `export_clashes` 兩種都跑 | [ ] | [ ] |
| 10 | 與 Navisworks 的差異都有原因 | Step 4 的表格沒有「未知」 | [ ] | — |
| 11 | Mock 模式下 sidecar 測試全部通過 | Mac 上 `dotnet test` | [ ] | — |

全部打勾後，到 [PROGRESS.md](../PROGRESS.md) 把 Week 2 標記完成。

## Step 6. M1 驗收報告

寫在 `docs/m1-acceptance.md`（位置可依專案調整），內容：

1. 測試環境：Revit 版本、Navisworks 版本、基準模型版本（以實際安裝為準）。
2. 規則清單與兩邊的設定。
3. 數量總表：本平台 / Navisworks / 標註，四類比對結果的筆數。
4. 差異清單：每筆的原因與處理方式（修正 / 記錄 / 留到之後）。
5. 效能：每條規則的執行時間、元素數量、候選配對數量。
6. 已知限制（例如連結元素無法單獨上色）。

## Step 7. 回填文件

- Day 8 – 13 的文件中，把標示「（待查證）」的 API 改成實際驗證的結果，並回填修正過的程式碼。
- [SPEC.md](../../SPEC.md) §11：IFC GUID、Navisworks 互通的發現寫進去。
- 遇到的問題寫進 [troubleshooting/](../troubleshooting/)。
- 無法消除的差異、延後處理的項目寫進 [PROGRESS.md](../PROGRESS.md) 的技術債。

## 回顧（寫下來）

**Revit 幾何 API 中，哪一個和你原本想的最不一樣？**

>

**與 Navisworks 的差異中，最意外的是哪一類？**

>

**哪一段程式碼你最沒有信心？為什麼？**

>

**兩個 Revit 版本之間這週又遇到了哪些差異？**

>

**目前最不確定的概念？（這會變成 Week 3 的學習重點）**

>

## 第三週預覽

第二週完成了 hard clash 的完整流程：偵測、查看、匯出。第三週讓結果「可以長期追蹤」：

| Day | 目標 | 牽涉的層 |
| --- | --- | --- |
| 15 | Clearance 與 Duplicate | Add-in（最短距離、重疊判斷）、Core（單位） |
| 16 | 忽略規則 | Core（`ignore` 的型別）、Add-in（hosted、joined、同系統） |
| 17 | 分群 | Core（依共同元素、空間鄰近、樓層與軸線） |
| 18 | 狀態追蹤與重跑比對 | Core + Sidecar（穩定 ID、自動標記 Resolved、`update_clash`） |
| 19 | BCF 2.1 結構與視角轉換 | Core（viewpoint、IFC GUID） |
| 20 | BCF 連接器與外部工具驗證 | Sidecar（連接器） |
| 21 | 整合測試、效能基準、M2 驗收 | 全部 |

詳細內容見 [week-3/README.md](../week-3/README.md)。

## Week 3 前要先想好的兩件事

1. **用哪兩個外部工具驗證 BCF？** M2 要求 BCF 能在至少兩個外部工具中開啟並正確定位。先確認可以取得哪些工具（例如 BIMcollab、Solibri、Revit 本身的 BCF 外掛，授權與 BCF 版本支援待查證）。
2. **重跑比對的測試情境**：準備一份「修改過的基準模型」，刻意移動、刪除、新增幾個元素，Day 18 用來驗證狀態追蹤（哪些變 Resolved、哪些是新的）。

---

## 學習筆記

（寫下今天的心得、卡關點、想問的問題）
