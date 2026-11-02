# Revit Controller — 開發 Roadmap

> 對應 [SPEC.md](SPEC.md) §10 的里程碑。每個里程碑約 7 個工作天（1 人），每一天的格式都一樣：**目標 → 工作項目 → 產出 → 知識點**。
> 「知識點」列出完成當天工作前需要理解的概念或 API，可以當作學習清單或 code review 的檢查重點。

總覽：

| 里程碑 | 主題 | 天數 |
|---|---|---|
| M0 | 環境、骨架、通訊管線 | Day 1–7 |
| M1 | Hard clash（含連結模型）、檢視、Excel/HTML 報告 | Day 8–14 |
| M2 | Clearance / Duplicate、忽略規則、分群、狀態追蹤、BCF 2.1 | Day 15–21 |
| M3 | AI 判讀、Dockable Pane、Navisworks 互通 | Day 22–28 |
| M4 | 修正建議與執行、ACC 連接器 | Day 29–35 |

---

## M0 — 環境、骨架、通訊管線（Day 1–7）

**驗收**：從 Mac 透過 MCP 呼叫 `get_model_overview`，在至少兩個 Revit 版本上都能取得結果。

### Day 1 — 環境建置與 §11 查證
- **工作**
  - 準備 Windows x64 開發機（實體機或雲端 GPU VM），安裝至少兩個 Revit 版本（建議一個 .NET Framework 4.8 版、一個 .NET 8 版）。
  - 安裝 Visual Studio / Rider、.NET SDK 8、.NET Framework 4.8 Developer Pack、Git。
  - 設定 Mac 到 Windows 的連線（RDP、SSH、檔案同步）。
  - 查證 SPEC §11 中與版本相關的項目，並更新 SPEC §4。
- **產出**：可遠端操作的 Windows 開發機；更新後的版本對照表。
- **知識點**
  - Revit 各版本與 .NET 執行環境的對應關係（.NET Framework 4.8 vs .NET 8）。
  - Revit 授權方式（教育版、試用版、公司授權），以及開發機的授權限制。
  - Apple Silicon 上的 Windows ARM 為何不適合當作 Revit 主要測試環境。

### Day 2 — Solution 結構與多版本 build
- **工作**
  - 建立 SPEC §4 的 solution：`Core`（netstandard2.0）、`Addin`、`Sidecar`（net8.0）、`Mock`（net8.0）。
  - Addin 專案依版本建立 build configuration（R22…R26），設定條件編譯符號 `REVIT20XX` 和 `REVIT20XX_OR_GREATER`。
  - 以 NuGet 參考各版本的 RevitAPI（確認套件名稱與授權）。
  - 撰寫 build script，一次產出所有版本的 DLL。
- **產出**：`dotnet build` 可以產出各版本的 add-in DLL。
- **知識點**
  - SDK-style csproj、`TargetFramework` 與多 configuration 的寫法。
  - netstandard2.0 為何能同時被 net48 和 net8.0 引用，以及它的 API 限制。
  - 條件編譯（`#if`）的使用時機，以及如何避免 `#if` 散落在整個專案中（改用 adapter 模式集中處理）。
  - RevitAPI.dll 為什麼要設 `Private=false`（不複製到輸出目錄）。

### Day 3 — Add-in 載入與 ExternalEvent 佇列
- **工作**
  - 撰寫 `.addin` manifest 與 `IExternalApplication`，在 Revit 啟動時載入。
  - 實作 `ExternalEvent` + `IExternalEventHandler` 的請求佇列：請求進佇列，在 UI thread 上執行，再用 `TaskCompletionSource` 回傳結果。
  - 實作 timeout 與錯誤狀態（`RevitBusy`、`NoActiveDocument`、`Timeout`）。
- **產出**：Revit 啟動時 add-in 自動載入，可以從外部排入請求並在 UI thread 上執行。
- **知識點**
  - Revit API 只能在 Revit 的 API context 中呼叫；在背景執行緒直接呼叫 API 會出錯或當機。
  - `IExternalApplication` 與 `IExternalCommand` 的差異；`.addin` manifest 的欄位（`AddInId`、`VendorId`、`FullClassName`）。
  - `ExternalEvent.Raise()` 的語意：不是立即執行，而是等 Revit 閒置時執行；同一個事件重複 Raise 會合併。
  - `TaskCompletionSource` 如何把 callback 轉成 async/await。
  - Revit 的 add-in 資料夾路徑：`%AppData%\Autodesk\Revit\Addins\20XX\` 與 `%ProgramData%` 版本的差異。

### Day 4 — Add-in ↔ Sidecar 的本機 RPC
- **工作**
  - 在 Core 定義 RPC 訊息格式（JSON-RPC 2.0：request id、method、params、error code）。
  - Add-in 端：named pipe server，收到請求後轉進 ExternalEvent 佇列。
  - Sidecar 端：named pipe client，處理斷線重連（Revit 重開時）。
  - 實作第一個方法：`ping`。
- **產出**：Sidecar 可以呼叫 add-in 的 `ping` 並拿到回應；Revit 重開後能自動重連。
- **知識點**
  - Named pipe 的原理與 .NET API（`NamedPipeServerStream` / `NamedPipeClientStream`），以及在 net48 與 net8 上的差異。
  - JSON-RPC 2.0 的 request、response、notification 格式與錯誤碼。
  - 訊息邊界的處理（length-prefix 或換行分隔）。
  - Add-in 端應使用 net48 也能用的 JSON 函式庫，或自己寫簡單的序列化，以減少相依衝突。

### Day 5 — Sidecar 的 MCP Server
- **工作**
  - 在 sidecar 中實作 MCP server（Streamable HTTP transport），註冊工具清單。
  - 加上 bearer token 認證。
  - 把 MCP 工具呼叫轉成對 add-in 的 RPC 呼叫。
  - 建立 audit log：記錄每一次工具呼叫的參數、結果與耗時。
- **產出**：從 Mac 用 MCP client（例如 MCP Inspector 或 AI harness）連到 sidecar，看得到工具清單。
- **知識點**
  - MCP 的核心概念：tools、tool 的 JSON schema、request/response 流程、transport（stdio vs Streamable HTTP）。
  - 為何要跨機器時選用 HTTP transport，以及基本的認證與網路安全（只開放區網或 VPN）。
  - 工具描述（description）的寫法會直接影響 AI 是否正確使用工具。
  - 結構化 log 的格式（JSON lines）與後續查詢方式。

### Day 6 — `get_model_overview` 與 Mock transport
- **工作**
  - 在 add-in 實作 `get_model_overview`：樓層、連結模型（含載入狀態）、各類別元素數量、目前視圖。
  - 實作 Mock：用 JSON fixture 模擬 add-in 的 RPC 回應，讓 sidecar 可以在 Mac 上跑。
  - 對 sidecar 寫第一批單元測試（以 Mock 執行）。
- **產出**：`get_model_overview` 在真實 Revit 與 Mock 上都能運作；CI 可以在沒有 Revit 的環境中執行測試。
- **知識點**
  - `Document`、`UIDocument`、`UIApplication` 的關係與取得方式。
  - `FilteredElementCollector` 的基本用法：`OfCategory`、`OfClass`、`WhereElementIsNotElementType`，以及 quick filter 與 slow filter 的效能差異。
  - `Level`、`View`、`RevitLinkInstance`、`RevitLinkType` 的關係；連結的載入狀態要從哪裡讀。
  - `BuiltInCategory` 與 `Category` 的對應。
  - 透過介面抽象 transport，讓 Mock 和真實實作可以互換。

### Day 7 — 安裝程式、雙版本驗證與緩衝
- **工作**
  - 撰寫安裝腳本，把各版本的 DLL 與 manifest 部署到對應的 Addins 資料夾，並安裝 sidecar。
  - 在兩個 Revit 版本上完整跑過 M0 驗收流程。
  - 整理 README 的開發環境設定說明；處理前幾天遺留的問題。
- **產出**：M0 驗收通過；新成員可以照 README 建好環境。
- **知識點**
  - 安裝工具的選擇（腳本、WiX、Inno Setup），以及是否需要管理員權限。
  - Revit 載入 add-in 時的安全提示與數位簽章。
  - 用 Revit journal 檔或 log 排查 add-in 載入失敗的問題。

---

## M1 — Hard clash（含連結模型）、檢視、Excel/HTML 報告（Day 8–14）

**驗收**：在基準模型上跑 hard clash，結果與 Navisworks 比對，差異都能解釋。

### Day 8 — 基準模型與資料模型
- **工作**
  - 準備基準模型：建築、結構、MEP 分檔，並用連結組合。人工標註已知的碰撞作為正確答案。
  - 在 Core 實作 `ClashRule` 與 `ClashRecord`（SPEC §5.2、§5.4）。
  - 在 sidecar 建立 SQLite Result Store 與 schema migration。
  - 實作規則管理工具：`create_clash_rule`、`list_clash_rules`、`delete_clash_rule`。
- **產出**：基準模型與標註檔；資料模型與儲存層。
- **知識點**
  - 碰撞檢討的業務背景：各專業之間常見的碰撞類型（MEP vs 結構、管線 vs 天花板等）。
  - 基準資料集（ground truth）的設計：要涵蓋真碰撞、可接受的碰撞、邊界情況。
  - SQLite 在 .NET 的使用（Microsoft.Data.Sqlite），以及 schema migration 的做法。
  - 穩定 ID 的設計：用雜湊組合規則與元素參照。

### Day 9 — 元素收集與幾何擷取
- **工作**
  - 依規則收集 A 組與 B 組元素：類別、參數篩選、主模型或連結模型。
  - 擷取元素的 solid，處理 `GeometryInstance`（族群實體）的展開。
  - 建立幾何快取，並在元素修改時讓快取失效。
- **產出**：給定一條規則，能取得兩組元素及它們的 solid。
- **知識點**
  - `Element.get_Geometry(Options)` 的用法，以及 `Options.DetailLevel`、`IncludeNonVisibleObjects` 的影響。
  - 幾何樹結構：`GeometryElement` → `Solid` / `GeometryInstance` → `GetInstanceGeometry()`；空 solid（`Volume == 0`）要排除。
  - 參數篩選：`ParameterFilterRuleFactory`、`ElementParameterFilter`。
  - 快取失效的依據：`DocumentChanged` 事件。

### Day 10 — 連結模型與座標轉換
- **工作**
  - 對連結模型中的元素，套用 `RevitLinkInstance.GetTotalTransform()` 把 solid 轉到主模型座標。
  - 處理同一個連結被放置多次的情況。
  - 處理未載入或隱藏的連結：回報給使用者，不得略過。
  - 計算元素的 IFC GUID 並存入紀錄。
- **產出**：主模型與連結模型的元素都能在同一個座標系統中比較。
- **知識點**
  - `Transform` 的組成（Origin、BasisX/Y/Z）與 `SolidUtils.CreateTransformed`。
  - 內部座標、專案基準點、測量點、共用座標的差異；回報給使用者時要用哪一個。
  - `RevitLinkInstance.GetLinkDocument()` 的回傳值何時為 null。
  - 連結中的元素在主模型裡是唯讀的。
  - Revit UniqueId 與 IFC GUID 的轉換規則（`ExportUtils.GetExportId` 或 IFC exporter 的演算法，需查證）。

### Day 11 — Hard clash 演算法
- **工作**
  - 初篩：BoundingBox 相交（考慮轉換後的 bounding box）。
  - 精算：同一模型內用 `ElementIntersectsSolidFilter`；跨連結時用 `BooleanOperationsUtils.ExecuteBooleanOperation` 取得交集 solid，並計算體積與中心點。
  - 處理 boolean 運算失敗的例外。
  - 實作 `run_clash_test` 非同步 job：分批執行、回報進度、可取消。
- **產出**：`run_clash_test` 與 `get_job_status` 可以運作，結果寫入 Result Store。
- **知識點**
  - Broad phase / narrow phase 的碰撞偵測概念。
  - `BoundingBoxXYZ` 與 `Outline`、`BoundingBoxIntersectsFilter`。
  - `ElementIntersectsSolidFilter` 與 `ElementIntersectsElementFilter` 的限制（不能直接跨文件）。
  - `BooleanOperationsUtils` 常見的 `InvalidOperationException`，以及容錯策略（略過並記錄，或改用 bounding box 結果標記為待確認）。
  - 在 ExternalEvent 中分批處理長工作，避免 UI 凍結。

### Day 12 — 查詢與檢視工具
- **工作**
  - 實作 MCP 工具：`list_clashes`（篩選與分頁）、`get_clash_detail`。
  - 實作 `show_clash`：建立或更新專用的 3D 視圖、設定 section box、對兩個元素上色、選取元素、匯出截圖。
  - 碰撞紀錄加上樓層與軸線位置（`grid_ref`）。
- **產出**：AI 可以列出碰撞、查看細節，並拿到碰撞位置的截圖。
- **知識點**
  - `View3D.CreateIsometric`、`SetSectionBox`、`ViewOrientation3D`。
  - 視圖覆寫：`OverrideGraphicSettings` 與 `View.SetElementOverrides`；連結元素的上色限制。
  - `UIDocument.Selection.SetElementIds` 與 `ShowElements`；選取連結元素需要 `Reference` 的 `CreateLinkReference`。
  - `Document.ExportImage` 與 `ImageExportOptions`。
  - 修改視圖需要 `Transaction`；匯出圖片不需要。
  - 找出最近的軸線交點：`Grid.Curve` 的計算。

### Day 13 — Excel / HTML 連接器
- **工作**
  - 在 Core 定義 `IClashConnector` 與 `ConnectorCapabilities`（SPEC §8.1）。
  - 實作 Excel 連接器：一筆碰撞一列，附上截圖。
  - 實作 HTML 報告：可以獨立開啟、內含截圖與篩選功能。
  - 實作 `list_connectors` 與 `export_clashes`。
- **產出**：碰撞結果可以匯出成 Excel 與 HTML。
- **知識點**
  - 連接器模式（strategy / plugin）與能力旗標（`[Flags] enum`）的設計。
  - Excel 函式庫的選擇與授權（例如 ClosedXML、EPPlus 的授權差異）。
  - 單一檔案 HTML 報告：把圖片以 base64 內嵌時要注意檔案大小。

### Day 14 — 與 Navisworks 比對與修正
- **工作**
  - 用相同的規則在 Navisworks Clash Detective 跑基準模型，與本平台的結果逐筆比對。
  - 分析差異：容差、幾何精度、忽略規則、族群幾何的不同處理方式。
  - 修正 bug，記錄無法消除的差異及其原因。
- **產出**：M1 驗收報告。
- **知識點**
  - Navisworks Clash Detective 的基本操作：Selection Sets、測試類型（Hard、Hard Conservative、Clearance、Duplicates）、容差設定。
  - Navisworks 的結果與 Revit solid 運算結果為何會不同（Navisworks 使用 tessellated mesh）。
  - 如何把比對差異整理成可重現的測試案例。

---

## M2 — Clearance / Duplicate、忽略規則、分群、狀態追蹤、BCF 2.1（Day 15–21）

**驗收**：重新執行時能正確對應上次的碰撞；BCF 能在至少兩個外部工具中開啟並正確定位。

### Day 15 — Clearance 與 Duplicate
- **工作**
  - Clearance：先把 bounding box 擴大淨空距離做初篩，再計算兩個 solid 的最短距離。
  - Duplicate：同類別、同族群類型、幾何幾乎重疊的元素。
  - 單位轉換：對外一律用 mm，內部用英尺。
- **產出**：三種碰撞類型都可以執行。
- **知識點**
  - Revit 內部單位（英尺）與 `UnitUtils.ConvertToInternalUnits` / `UnitTypeId`。
  - Solid 之間最短距離的計算方式：可以用 face 或 edge 投影近似，或擷取 mesh 後計算；精度與效能的取捨。
  - Duplicate 的判斷標準：位置容差、方向、族群類型。

### Day 16 — 忽略規則
- **工作**
  - 實作內建忽略規則：`hosted`（門窗與牆）、`joined`（已接合元素）、`same_system`（同一 MEP 系統）。
  - 實作自訂規則：依參數條件忽略。
  - 在基準模型上驗證誤報減少的情況。
- **產出**：誤報明顯減少，且不會漏掉標註的真碰撞。
- **知識點**
  - `FamilyInstance.Host` 與 hosted 元素的關係。
  - `JoinGeometryUtils.AreElementsJoined`。
  - MEP 系統：`MEPSystem`、`Connector`、`ConnectorManager`，以及如何判斷兩個元素在同一個系統中。
  - 誤報（false positive）與漏報（false negative）的取捨：忽略規則寧可保守。

### Day 17 — 分群
- **工作**
  - 依共同元素分群（同一根管穿過多根梁算一群）。
  - 依空間鄰近與樓層、軸線分群。
  - `list_clashes` 支援依群組查詢；群組摘要（數量、代表截圖）。
- **產出**：大量碰撞可以收斂成少量的群組。
- **知識點**
  - 圖的連通元件（union-find / disjoint set）用於「共同元素」分群。
  - 空間分群：以網格（grid hashing）或 DBSCAN 的概念處理鄰近點。
  - 群組 ID 的穩定性：重新執行時群組如何對應。

### Day 18 — 狀態追蹤與重跑比對
- **工作**
  - 重新執行時，用穩定的 `clash_id` 與前次結果比對：新增的標為 New，仍存在的保留原狀態，消失的自動標為 Resolved。
  - 實作 `update_clash`：狀態、負責人、留言。
  - 記錄狀態變更歷史。
- **產出**：連續執行兩次（中間修改模型），狀態變化正確。
- **知識點**
  - 狀態機（state machine）的設計：哪些轉換合法、誰可以觸發。
  - 元素被刪除或重建時 UniqueId 會改變，對穩定 ID 的影響與處理方式。
  - 稽核需求：狀態變更歷史要記錄誰、何時、為什麼。

### Day 19 — BCF 2.1 結構與視角
- **工作**
  - 研讀 BCF 2.1 規格，實作資料轉換：`ClashRecord` → Topic（`markup.bcf`）、Viewpoint（`.bcfv`）、截圖（`.png`）。
  - 把 Revit 的相機（`ViewOrientation3D`）轉成 BCF 的 `PerspectiveCamera` / `OrthogonalCamera`。
  - 元素以 IFC GUID 填入 `Components/Selection`。
- **產出**：BCF 資料轉換邏輯與單元測試。
- **知識點**
  - BCF 2.1 的檔案結構：`bcf.version`、`project.bcfp`、每個 Topic 一個資料夾（`markup.bcf`、`viewpoint.bcfv`、`snapshot.png`）。
  - BCF 的 XML schema（XSD）與驗證方式。
  - BCF 相機座標使用公尺與 IFC 座標系，與 Revit 座標的轉換（含共用座標）。
  - BCF 2.1 與 3.0 的主要差異。

### Day 20 — BCF 連接器與外部工具驗證
- **工作**
  - 實作 BCF 2.1 連接器：打包成 `.bcfzip`，`external_refs` 記錄 Topic GUID。
  - 在至少兩個外部工具中開啟（例如 BIMcollab Zoom、Solibri，或 Navisworks 加外掛），確認相機位置與元素定位正確。
  - 查證各工具的 BCF 支援程度，更新 SPEC §11。
- **產出**：可用的 BCF 匯出；外部工具的驗證紀錄。
- **知識點**
  - 外部工具如何用 IFC GUID 對應元素：需要對方也載入同一模型匯出的 IFC，或 Revit 模型本身。
  - Revit 的 IFC 匯出設定對 GUID 的影響。
  - 重複匯出時保持同一個 Topic GUID，讓外部工具視為更新而不是新增。

### Day 21 — 整合測試與緩衝
- **工作**
  - 在所有支援的 Revit 版本上跑完整流程：規則 → 執行 → 分群 → 狀態更新 → 匯出。
  - 效能測試：大型模型（數萬個元素）的執行時間與記憶體用量。
  - 修正問題，更新文件。
- **產出**：M2 驗收報告與效能基準。
- **知識點**
  - Revit 中的效能量測（`Stopwatch`、分段計時）與瓶頸定位。
  - 何時值得把幾何轉成 mesh 交給背景執行緒計算（SPEC §5.3 的進階優化）。

---

## M3 — AI 判讀、Dockable Pane、Navisworks 互通（Day 22–28）

**驗收**：人工抽樣 AI 判讀的準確率達到設定門檻。

### Day 22 — AI 工作流程設計
- **工作**
  - 定義 AI 判讀流程：取群組 → 查細節 → 看截圖 → 給出 `ai_assessment`（判定、嚴重度、理由）→ 寫回 `update_clash`。
  - 撰寫 system prompt 與判讀準則：哪些情況屬於可接受（例如保溫層輕微接觸），哪些屬於嚴重。
  - 檢查所有 MCP 工具的描述與 schema，讓 AI 更容易正確使用。
- **產出**：AI 判讀的 prompt、準則文件與工作流程。
- **知識點**
  - Agent loop 的概念：觀察 → 規劃 → 執行 → 驗證。
  - Prompt 設計：給 AI 明確的判斷標準與輸出格式（structured output）。
  - 多模態輸入：截圖如何幫助 AI 判斷，以及圖片解析度與 token 成本的取捨。
  - Context 管理：大量碰撞要分批處理，避免超出 context。

### Day 23 — AI 判讀評估
- **工作**
  - 用基準模型的標註建立評估集。
  - 跑 AI 判讀，計算準確率、誤判類型，並記錄成本（token、時間）。
  - 迭代 prompt 與工具回傳內容，直到達到門檻。
- **產出**：評估腳本與準確率報告。
- **知識點**
  - 評估指標：accuracy、precision / recall、混淆矩陣。
  - 為何要固定評估集：避免針對個案調 prompt 而造成整體退步。
  - 成本估算：每筆碰撞平均的 token 與工具呼叫次數。

### Day 24 — Dockable Pane 基礎
- **工作**
  - 建立 WPF Dockable Pane 並註冊到 Revit。
  - 顯示碰撞清單，支援依狀態、群組、規則、樓層篩選。
  - 從 sidecar 取得資料（Pane 也是 sidecar 的 client）。
- **產出**：Revit 內可以看到碰撞清單。
- **知識點**
  - `IDockablePaneProvider`、`DockablePaneId`、`RegisterDockablePane`；必須在啟動時註冊。
  - WPF 基礎：XAML、資料繫結、MVVM。
  - WPF 在 net48 與 net8 版 Revit 中的差異，以及與其他 add-in 的資源衝突。
  - UI 事件中不能直接呼叫 Revit API 修改模型，要透過 ExternalEvent。

### Day 25 — Dockable Pane 互動
- **工作**
  - 點選碰撞時呼叫 `show_clash` 的同一段邏輯：跳到視圖並選取元素。
  - 顯示 AI 判讀理由，提供接受與改判按鈕。
  - 加入匯出按鈕，列出支援 Export 的連接器。
- **產出**：使用者可以在 Revit 內完成檢視、判讀確認與匯出。
- **知識點**
  - 共用邏輯：UI 與 MCP 工具呼叫同一個 service，避免兩套行為不一致。
  - 人工改判 AI 結果時要記錄，作為之後改進 prompt 的資料。

### Day 26 — Pane 內的 AI 對話
- **工作**
  - 在 Pane 中加入對話輸入框，連到 AI harness（例如「只看 3F 的風管對梁」）。
  - 顯示 AI 的執行過程（呼叫了哪些工具）與結果。
  - 支援中斷執行。
- **產出**：使用者可以在 Revit 內用自然語言操作碰撞檢討。
- **知識點**
  - Harness 的整合方式：Pane 透過 sidecar 轉送，或直接呼叫 harness 的 API。
  - 串流回應（streaming）的顯示。
  - 中斷與取消（`CancellationToken`）要一路傳到 add-in 的 job。

### Day 27 — Navisworks 互通
- **工作**
  - 依 SPEC §11 的查證結果，選擇互通方式：BCF（透過外掛）、碰撞測試定義 XML 交換，或開發 Navisworks 外掛。
  - 實作選定的方式，或寫成評估報告並排入後續計畫。
- **產出**：Navisworks 連接器（或評估報告）。
- **知識點**
  - Navisworks 的擴充方式：.NET API、外掛架構、Clash Detective API。
  - Clash Detective 測試定義的 XML 匯入與匯出格式。
  - NWC / NWD 檔案與 Revit 模型的關係，以及 Navisworks 中的元素如何對應回 Revit。

### Day 28 — 整合測試與緩衝
- **工作**
  - 讓真實使用者試用完整流程，收集回饋。
  - 修正問題，更新文件。
- **產出**：M3 驗收報告與使用者回饋清單。
- **知識點**
  - 可用性測試的進行方式：給任務、觀察、不引導。

---

## M4 — 修正建議與執行、ACC 連接器（Day 29–35）

**驗收**：修正可以 rollback，且完整寫入 audit log。

### Day 29 — 修正策略設計
- **工作**
  - 定義可以自動修正的類型：移動管線或風管（偏移、繞行）、調整高程、刪除重複元素。
  - 定義 `propose_fix` 的輸出格式：要修改哪些元素、怎麼改、預期效果、影響範圍。
  - 標示無法在主模型修改的情況（連結模型的元素），改為指派給負責的專業。
- **產出**：修正策略文件與 `propose_fix` 的 schema。
- **知識點**
  - MEP 管線的幾何：`LocationCurve`、`MEPCurve`，以及移動後與接頭（fitting）的連接關係。
  - `ElementTransformUtils.MoveElement` 與參數修改（例如 `Offset`）的差異。
  - 修改一個元素可能牽動其他元素（接頭、hosted 元素）。

### Day 30 — Dry-run 與預覽
- **工作**
  - 實作 `propose_fix`：在 `TransactionGroup` 中試做修改，重新檢查碰撞，記錄結果後 rollback。
  - 回傳預覽：修改前後的截圖、是否產生新碰撞或 warning。
- **產出**：AI 可以提出修正方案並預覽結果，模型不會被改動。
- **知識點**
  - `Transaction`、`SubTransaction`、`TransactionGroup` 的差異與巢狀規則；`RollBack` 與 `Assimilate`。
  - `IFailuresPreprocessor` 與 `FailuresAccessor`：處理修改時跳出的 warning 和 error。
  - Dry-run 的限制：有些變更（例如重新生成）在 rollback 後仍可能留下副作用，需要驗證。

### Day 31 — 核准與執行
- **工作**
  - 實作 `apply_fix`：需要人工核准（在 Pane 中確認，或由 harness 的權限機制控制）。
  - 執行修改，寫入 audit log，重新檢查相關碰撞並更新狀態。
  - 提供 undo（rollback 整組修改）。
- **產出**：修正可以安全執行並回復。
- **知識點**
  - 人在迴圈（human-in-the-loop）的設計：核准點的位置、顯示哪些資訊讓人判斷。
  - 權限分級：唯讀、可修改、可執行腳本；依使用者或任務設定。
  - Worksharing 環境下的修改：元素擁有權（`WorksharingUtils.CheckoutElements`）、借用失敗的處理。

### Day 32 — ACC 連接器（一）：認證與專案
- **工作**
  - 註冊 APS 應用程式，實作 OAuth 認證。
  - 取得 hub、project 與 Issues 容器。
  - 查證 ACC Issues API 的權限與限制，更新 SPEC §11。
- **產出**：sidecar 可以連到客戶的 ACC 專案。
- **知識點**
  - APS（Autodesk Platform Services）的應用程式註冊與 OAuth（2-legged vs 3-legged）。
  - Token 的保存與更新；機密資訊不得進入版本控制。
  - ACC 的資料結構：hub → project → container → issues。

### Day 33 — ACC 連接器（二）：匯出與同步
- **工作**
  - 匯出：把碰撞建立成 ACC Issue（含截圖、位置、元素參照），`external_refs` 記錄 Issue ID。
  - 同步：雙向同步狀態與留言，處理衝突（兩邊同時修改）。
- **產出**：碰撞與 ACC Issues 可以雙向同步。
- **知識點**
  - ACC Issues API 的欄位（類型、狀態、指派、位置、附件）。
  - 同步衝突的處理策略：最後修改者優先，或保留兩邊並提示使用者。
  - API rate limit 與重試（exponential backoff）。

### Day 34 — 安全性與穩定性檢查
- **工作**
  - 檢查認證、網路曝露範圍、token 保存、audit log 的完整性。
  - 測試異常情境：Revit 當機、sidecar 重啟、網路中斷、執行中取消。
- **產出**：安全性檢查清單與異常測試結果。
- **知識點**
  - Threat modeling 的基本做法：誰可以呼叫工具、可以做到什麼。
  - AI 工具的風險：prompt injection（例如模型中的文字參數被當成指令）、權限過大的工具。
  - 冪等性（idempotency）：重試時不會重複執行修改。

### Day 35 — 收尾與後續規劃
- **工作**
  - 在所有支援的 Revit 版本上做最終回歸測試。
  - 整理使用手冊、部署文件、已知限制。
  - 回顧整體進度，規劃下一階段（其他 Revit 任務、BCF 3.0、Design Automation 等）。
- **產出**：v1 發布；下一階段規劃。
- **知識點**
  - 發布流程：版本號、change log、安裝程式簽章。
  - 如何把碰撞檢討建立的基礎（佇列、RPC、MCP、連接器、audit log）延伸到其他 Revit 任務。
