# Week 2 · Hard clash、檢視與報告（M1）

第一週打通了「AI → Sidecar → Add-in → Revit」的管線，但只有一個唯讀工具 `get_model_overview`。第二週開始做真正的碰撞檢討：**準備基準模型 → 收集元素與幾何 → 處理連結模型 → 計算 hard clash → 讓 AI 查詢與看截圖 → 匯出 Excel / HTML → 與 Navisworks 比對**。

每日文件在本資料夾。**程式碼都是骨架，撰寫時沒有編譯或執行過**（見 [../README.md](../README.md)「文件的可信度」）。Revit 幾何相關的 API 特別容易寫錯，標示「（待查證）」的地方請對照實際安裝版本的 RevitAPI 文件確認後回填。

## 每日日程

| Day | 主題 | 文件 |
| --- | --- | --- |
| Day 8 | 基準模型與資料模型 | [day-08.md](day-08.md) |
| Day 9 | 元素收集與幾何擷取 | [day-09.md](day-09.md) |
| Day 10 | 連結模型與座標轉換 | [day-10.md](day-10.md) |
| Day 11 | Hard clash 演算法與非同步 job | [day-11.md](day-11.md) |
| Day 12 | 查詢與檢視工具（`show_clash`） | [day-12.md](day-12.md) |
| Day 13 | 連接器介面與 Excel / HTML 連接器 | [day-13.md](day-13.md) |
| Day 14 | 與 Navisworks 比對、M1 驗收 | [day-14.md](day-14.md) |

哪幾天可以只在 Mac 上做：

| Day | Mac（Mock） | Windows（Revit） |
| --- | --- | --- |
| 8 | 資料模型、SQLite、規則工具 | 準備與標註基準模型 |
| 9 – 11 | Mock fixture、job 管理 | 收集、幾何、連結、碰撞計算 |
| 12 | `list_clashes`、`get_clash_detail` | `show_clash`、截圖 |
| 13 | 全部 | — |
| 14 | 比對腳本 | Navisworks 比對 |

## Week 2 Definition of Done

基準模型上的 hard clash（含連結模型）結果與 Navisworks 比對，差異都能解釋；可以匯出 Excel / HTML。

- [ ] 基準模型（建築、結構、MEP 分檔並互相連結）與人工標註檔已放進 `fixtures/`
- [ ] `run_clash_test` 在主模型對連結、連結對連結上都能算出 hard clash，結果寫入 SQLite
- [ ] 未載入的連結會回報 `LinkNotLoaded`，不會被略過
- [ ] `list_clashes`、`get_clash_detail`、`show_clash`（含截圖）可以從 Mac 透過 MCP 呼叫
- [ ] `export_clashes` 可以匯出 Excel 與 HTML
- [ ] 與 Navisworks 的比對結果整理成 M1 驗收報告，每一筆差異都有原因
- [ ] 兩個 Revit 版本都跑過一次

## 本週新增的 MCP 工具與 RPC 方法

| MCP 工具（Sidecar） | 背後的 RPC 方法（Add-in） | Day |
| --- | --- | --- |
| `create_clash_rule` / `list_clash_rules` / `delete_clash_rule` | 無（只存在 SQLite） | 8 |
| 無（診斷用，不開放為 MCP 工具） | `clash.debugCollect` | 9 |
| `run_clash_test(rule_ids)` | `clash.prepare`、`clash.runBatch`、`clash.cancel` | 11 |
| `get_job_status(job_id)` | 無（Sidecar 的 job 狀態） | 11 |
| `list_clashes(filter, page)` | 無（查 SQLite） | 12 |
| `get_clash_detail(clash_id)` | 無；需要即時參數時呼叫 `element.params`（待決定） | 12 |
| `show_clash(clash_id)` | `view.showClash` | 12 |
| `list_connectors` | 無 | 13 |
| `export_clashes(connector_id, filter, options)` | 無 | 13 |

RPC 方法名稱沿用 Week 1 的 `model.overview` 點分格式。`update_clash`、`query_elements` 不在本週範圍：狀態追蹤在 Week 3 做。

> SPEC §7 的規則管理工具在 [ROADMAP.md](../../ROADMAP.md) 沒有排到特定日子。因為 `run_clash_test(rule_ids)` 需要先有規則，本 roadmap 把它們放在 Day 8。

## 本週程式碼的落點

延續 [02-tech-stack.md](../02-tech-stack.md) 的目錄結構：

```
fixtures/
├── baseline/README.md             ← 基準模型清單、連結方式、存放位置（Day 8）
├── baseline/expected-clashes.json ← 人工標註的正確答案（Day 8）
└── mock/clash/                    ← Mock 用的碰撞結果 fixture（Day 11）

src/RevitController.Core/
├── Models/ClashRule.cs            ← SPEC §5.2（Day 8）
├── Models/ClashRecord.cs          ← SPEC §5.4（Day 8）
├── Models/ClashId.cs              ← 穩定 ID 雜湊（Day 8）
├── Units/Mm.cs                    ← mm ↔ 英尺換算（Day 9）
├── Rpc/ClashMessages.cs           ← clash.* 的參數與回傳 DTO（Day 11）
└── Connectors/                    ← IClashConnector、ConnectorCapabilities（Day 13）

src/RevitController.Addin/
├── Clash/ElementCollector.cs      ← 依規則收集 A / B 組（Day 9）
├── Clash/GeometryExtractor.cs     ← 擷取 solid、展開 GeometryInstance（Day 9）
├── Clash/GeometryCache.cs         ← 快取與 DocumentChanged 失效（Day 9）
├── Clash/LinkResolver.cs          ← 連結 instance、Transform、載入狀態（Day 10）
├── Clash/IfcGuid.cs               ← IFC GUID（Day 10）
├── Clash/HardClashEngine.cs       ← broad / narrow phase（Day 11）
├── Clash/ClashSession.cs          ← 每個 job 的配對清單與游標（Day 11）
├── Handlers/ClashHandlers.cs      ← clash.prepare / runBatch / cancel（Day 11）
├── Handlers/ShowClashHandler.cs   ← 3D 視圖、section box、上色、截圖（Day 12）
└── Clash/GridLocator.cs           ← grid_ref（Day 12）

src/RevitController.Sidecar/
├── Store/ResultStore.cs           ← SQLite 存取（Day 8）
├── Store/Migrations/              ← 001_init.sql …（Day 8）
├── Mcp/RuleTools.cs               ← 規則管理工具（Day 8）
├── Jobs/JobManager.cs             ← job 建立、進度、取消（Day 11）
├── Mcp/ClashTools.cs              ← run / status / list / detail / show（Day 11 – 12）
├── Connectors/ExcelConnector.cs   ← （Day 13）
├── Connectors/HtmlConnector.cs    ← （Day 13）
├── Connectors/ConnectorRegistry.cs← （Day 13）
└── Mcp/ExportTools.cs             ← list_connectors / export_clashes（Day 13）

src/RevitController.Mock/          ← 加上 clash.* 與 view.showClash 的假回應（Day 11 – 12）

docs/m1-acceptance.md              ← M1 驗收報告（Day 14）
```

（檔名是建議，實際以你建好的專案為準，完成後回填。）

## 本週的學習重點

| 主題 | 內容 |
| --- | --- |
| 碰撞檢討的業務背景 | 各專業常見的碰撞類型、真碰撞與可接受碰撞的差別、基準資料集 |
| 資料模型與儲存 | `ClashRule` / `ClashRecord`、穩定 ID、Microsoft.Data.Sqlite、schema migration |
| Revit 元素收集 | `FilteredElementCollector`、`ElementMulticategoryFilter`、`ElementParameterFilter` |
| Revit 幾何 | `Options`、`GeometryElement`、`Solid`、`GeometryInstance.GetInstanceGeometry()` |
| 座標與連結 | `RevitLinkInstance.GetTotalTransform()`、`SolidUtils.CreateTransformed`、各種基準點 |
| 碰撞演算法 | Broad phase（bounding box）/ narrow phase（`ElementIntersectsSolidFilter`、`BooleanOperationsUtils`） |
| 長時間工作 | 非同步 job、分批 ExternalEvent、進度與取消 |
| 視圖與截圖 | `View3D`、`SetSectionBox`、`OverrideGraphicSettings`、`Document.ExportImage` |
| 連接器模式 | strategy / plugin、`[Flags] enum`、ClosedXML、單一檔案 HTML |
| 驗證方法 | 與 Navisworks Clash Detective 逐筆比對、把差異變成測試案例 |

## 本週要特別注意的風險

1. **幾何運算的例外**：`BooleanOperationsUtils` 對某些幾何會丟例外。不能讓一個元素的失敗中斷整個 job，要記錄下來並標記為待確認。
2. **UI 凍結**：碰撞計算一定要分批，每批的時間要量測，見 Day 11。
3. **連結被略過**：未載入或隱藏的連結如果被默默略過，結果會少掉一整個專業。SPEC §6 規定要回報。
4. **單位**：對外一律 mm，內部是英尺。每個 DTO 欄位名稱都帶單位（`volume_mm3`、`tolerance_mm`），減少混淆。
5. **模型內容是資料，不是指令**：元素名稱、參數值會出現在工具回傳與 HTML 報告中。HTML 要做 escape，回傳給 AI 時也要當作資料。
