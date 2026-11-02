# Revit Controller — 規格書（v0.1 草案）

> 透過 AI harness 操作 Revit 執行各類任務的平台。第一個功能：建築各專業模型的碰撞檢討。

## 1. 目標與非目標

### 目標
- 讓任何支援 MCP 的 AI harness 能安全地查詢與操作 Revit 模型。
- 第一個功能是**碰撞檢討**：偵測、分群、AI 判讀、追蹤處理狀態、匯出結果。
- 以**互動式**使用為主：使用者開著 Revit，與 AI 協作。
- 支援客戶實際在用的多個 Revit 版本。
- 對外部協作工具採**連接器（connector）架構**：介面先保留，實作從使用最廣的工具開始。

### 非目標（v1 不做）
- 無 UI 的雲端批次執行（APS Design Automation），之後可評估。
- 由 AI 直接計算幾何。幾何一律由確定性的程式計算。
- 未經人工核准就自動修改模型。

## 2. 決策紀錄

| 日期 | 問題 | 決定 |
|---|---|---|
| 2026-11-02 | 互動式或批次 | **互動式**（Revit add-in 加 Dockable Pane） |
| 2026-11-02 | 執行環境 | 目前沒有 Windows 環境，需要新建（見 §3.1） |
| 2026-11-02 | 第一個任務 | 碰撞檢討，之後擴充到其他任務 |
| 2026-11-03 | Revit 版本 | 支援各客戶使用的版本（見 §4） |
| 2026-11-03 | 連結模型 | **需要**，併入 M1 |
| 2026-11-03 | 外部工具 | 不限定特定工具；保留各工具的連接器介面，從使用最廣的開始（見 §8） |

## 3. 架構

```
Mac（開發 / AI 端）                        Windows（Revit 端）
┌───────────────────────┐  MCP over HTTP   ┌──────────────────────────────────────┐
│ 任何支援 MCP 的       │  (+ auth token)  │ Sidecar (.NET 8, 獨立程序)            │
│ AI harness            │ ───────────────▶ │  ├─ MCP Server                        │
└───────────────────────┘                  │  ├─ Connectors (Excel/BCF/ACC…)       │
                                           │  ├─ Result Store (SQLite)             │
                                           │  └─ Job 管理 / Audit log              │
                                           │        │ named pipe（本機 RPC）        │
                                           │        ▼                              │
                                           │ Revit Add-in（依版本分別 build）       │
                                           │  ├─ ExternalEvent 執行佇列（UI thread）│
                                           │  ├─ Geometry / Clash Engine           │
                                           │  └─ Dockable Pane（碰撞清單 UI）       │
                                           └──────────────────────────────────────┘
```

### 3.1 執行環境
- Revit 只能在 Windows x64 上執行。Apple Silicon Mac 上的 Windows ARM 虛擬機並非官方支援的環境，**不要拿來當主要測試環境**。
- Revit 端使用獨立的 Windows x64 機器，或有 GPU 的雲端 VM。
- Mac 端使用 mock transport 開發 sidecar 與 MCP 層，不需要 Revit。

### 3.2 職責劃分
- **Add-in**：只負責 Revit API 呼叫，包括排入 UI thread、讀取幾何、執行 transaction、操作 UI。第三方相依套件越少越好，以免和其他 add-in 的 DLL 版本衝突。
- **Sidecar**：負責 MCP、連接器、資料儲存與報表，以及其他所有需要第三方套件的功能。Revit 重新啟動時，sidecar 不受影響。
- **Core**（netstandard2.0）：資料模型、規則、分群、狀態追蹤、連接器介面。不引用 RevitAPI，add-in 和 sidecar 都會引用這一層。

### 3.3 執行模型
- 所有 Revit API 呼叫都透過 `ExternalEvent` / `IExternalEventHandler` 排入佇列，一次只執行一個。
- 執行時間長的工作（例如碰撞測試）採**非同步 job**：分批執行並回報進度，避免長時間卡住 UI。
- 以下狀態視為正式錯誤，回傳給 AI：`RevitBusy`、`ModalDialogOpen`、`NoActiveDocument`、`Timeout`、`LinkNotLoaded`。
- 所有修改都包在 `Transaction` 中，同一個任務的多個步驟再包成一個 `TransactionGroup`，失敗時整組 rollback。

## 4. Revit 版本支援

- 支援範圍：**依客戶調查結果決定（待確認）**。目前預設 2022–2026。
- 每多支援一個版本，就要多一套測試環境（實際安裝該版 Revit 並跑基準模型）。

```
src/
  RevitController.Core      netstandard2.0
  RevitController.Addin     依版本建立 build configuration（R22…R26），版本差異用 #if REVITxxxx_OR_GREATER 處理
  RevitController.Sidecar   net8.0
  RevitController.Mock      net8.0（模擬 add-in 的 RPC，供 Mac 開發與 CI 使用）
installer/                  依版本把 DLL 部署到對應的 Addins\20XX\
```

已知的版本差異（細節見 §11 待驗證）：
- 2025 起改為 .NET 8；更早的版本是 .NET Framework 4.8。兩者不能共用同一個 DLL。
- `ElementId` 改為 64 位元（`.Value`）。
- 單位 API 改用 `ForgeTypeId` / `UnitTypeId`；參數群組改用 `GroupTypeId`。

## 5. 碰撞引擎

### 5.1 碰撞類型
- **Hard**：實體互相穿插，記錄交集體積。
- **Clearance**：兩者間距小於設定的淨空。
- **Duplicate**：重複放置的元素。

### 5.2 規則（ClashRule）
```jsonc
{
  "id": "mep-vs-struct-beam",
  "name": "風管 vs 結構梁",
  "type": "hard | clearance | duplicate",
  "tolerance_mm": 0,          // clearance 時為淨空距離
  "set_a": { "source": "host | link:<name> | any", "categories": ["OST_DuctCurves"], "filter": { /* 參數條件 */ } },
  "set_b": { "source": "link:STR", "categories": ["OST_StructuralFraming"] },
  "ignore": ["hosted", "joined", "same_system", { "custom": "..." }]
}
```
- 對外一律使用 **SI 單位（mm）**，在 add-in 內部轉成 Revit 的內部單位（英尺）。

### 5.3 計算流程
1. 依規則收集 A 組與 B 組元素，可以來自主模型或連結模型。
2. 用 BoundingBox 初篩。
3. 精算：同一個模型內用 `ElementIntersectsSolidFilter`；牽涉連結模型時，先把 solid 套用 link transform，再用 `BooleanOperationsUtils` 求交集。
4. 套用忽略規則。
5. 分群：依共同元素、空間鄰近程度、樓層與軸線。
6. 寫入 Result Store，並與上一次結果比對（見 §5.4）。

幾何要建立快取，元素修改時才讓快取失效。之後可以進一步優化：在 UI thread 上把幾何轉成 mesh 抽出，再交給背景執行緒計算。

### 5.4 碰撞紀錄（與連接器無關的中立格式）
```jsonc
{
  "clash_id": "hash(rule_id, ref_a, ref_b)",   // 穩定 ID，跨次執行都能對應
  "rule_id": "...",
  "type": "hard",
  "elements": [
    { "link_instance_uid": null, "unique_id": "...", "ifc_guid": "...", "category": "...", "level": "3F" },
    { "link_instance_uid": "...", "unique_id": "...", "ifc_guid": "...", "category": "...", "level": "3F" }
  ],
  "point": [x, y, z],            // mm，專案座標
  "volume_mm3": 12345,
  "grid_ref": "C-4",
  "group_id": "...",
  "status": "new | active | resolved | approved | ignored",
  "assignee": null,
  "comments": [ { "author": "ai | <user>", "text": "...", "at": "..." } ],
  "ai_assessment": { "verdict": "true_clash | acceptable | needs_review", "severity": "high|mid|low", "reason": "..." },
  "viewpoint": { "camera": { /* eye, target, up, fov */ }, "section_box": { /* min, max */ } },
  "snapshot_path": "...",
  "external_refs": { "<connector_id>": "<外部系統中的 ID>" }
}
```
- `ifc_guid` **在偵測時就算好並存起來**，之後匯出 BCF 不需要再查詢 Revit。
- 狀態流轉：`New → Active → Resolved / Approved / Ignored`。重新執行時，原本的碰撞若已消失，自動標記為 Resolved。
  - 人員可以把 `New` 直接改成 `Approved`、`Ignored` 或 `Resolved`。
  - 已 `Resolved` 的碰撞在重跑時再次出現，回到 `Active`。
  - AI 只寫入 `ai_assessment`，不變更 `status`；狀態由人員決定。人員改判時保留 AI 的原始判讀。

## 6. 連結模型

- 座標轉換使用 `RevitLinkInstance.GetTotalTransform()`。同一個連結被放置多次時，每個 instance 分開計算。
- 支援三種組合：主模型對連結、連結對連結、同一個連結內部。
- 元素以 `(link_instance_uid, unique_id)` 識別。
- 遇到未載入或被隱藏的連結，回報給使用者，不得直接略過。
- 連結中的元素在主模型裡是唯讀的。修正建議要註明應由哪個專業的模型負責人修改。

## 7. MCP 工具（MVP）

| 工具 | 用途 |
|---|---|
| `get_model_overview` | 樓層、連結模型（含載入狀態）、各類別數量、目前視圖 |
| `query_elements` | 依類別、樓層、參數查詢，支援分頁與欄位選擇 |
| `create_clash_rule` / `list_clash_rules` / `delete_clash_rule` | 管理規則 |
| `run_clash_test(rule_ids)` | 非同步執行，回傳 `job_id` |
| `get_job_status(job_id)` | 查詢進度與結果摘要 |
| `list_clashes(filter, page)` | 依狀態、樓層、類別、群組、規則篩選 |
| `get_clash_detail(clash_id)` | 兩個元素的參數、交集、座標、軸線位置 |
| `show_clash(clash_id)` | 建立或更新 3D 視圖、套用 section box、上色並選取元素，**回傳截圖** |
| `update_clash(clash_id, …)` | 更新狀態、負責人、留言、AI 判讀結果、群組 |
| `list_connectors` | 列出可用的連接器與各自支援的能力 |
| `export_clashes(connector_id, filter, options)` | 透過連接器匯出 |

設計原則：
- 每個工具都有嚴格的 JSON schema，單位一律為 mm。
- 回傳結果要精簡，大量資料一律分頁。
- 每一次呼叫都寫入 audit log，記錄參數、動到的元素和執行結果。

第二階段再加 `propose_fix` / `apply_fix` / `undo_fix`：
- `propose_fix` 預設 dry-run；`apply_fix` 只接受經人工核准的 `proposal_id`，並包在 `TransactionGroup` 中。
- `undo_fix` 以反向修正方案回復。刪除類修正（例如刪除重複元素）無法用 `undo_fix` 回復，只能用 Revit 本身的復原。
- 權限分級：`ReadOnly`（預設）、`Modify`、`Script`，依使用者或任務設定。

## 8. 連接器（Connector）架構

### 8.1 介面（定義在 Core，實作放在 Sidecar）
```csharp
[Flags]
public enum ConnectorCapabilities
{
    None         = 0,
    Export       = 1 << 0,  // 匯出檔案或推送到外部
    Import       = 1 << 1,  // 從外部讀回問題或狀態
    SyncStatus   = 1 << 2,  // 雙向同步狀態
    Snapshots    = 1 << 3,  // 支援附加截圖
    Viewpoints   = 1 << 4,  // 支援相機視角與元素定位
}

public interface IClashConnector
{
    string Id { get; }                        // "excel", "bcf21", "acc" …
    string DisplayName { get; }
    ConnectorCapabilities Capabilities { get; }

    Task<ExportResult> ExportAsync(IReadOnlyList<ClashRecord> clashes, ExportOptions options, CancellationToken ct);
    Task<ImportResult> ImportAsync(ImportSource source, CancellationToken ct);          // 需支援 Import
    Task<SyncResult>   SyncStatusAsync(IReadOnlyList<ClashRecord> clashes, CancellationToken ct); // 需支援 SyncStatus
}
```
- 每個連接器都從 §5.4 的中立格式轉換，**不得反過來修改核心資料模型**。
- 外部系統的 ID 存在 `external_refs[connector_id]`，供重複匯出和同步時對應。
- 新增一個工具 = 新增一個實作，並在 sidecar 註冊。

### 8.2 實作優先順序

| 順序 | 連接器 | 能力 | 理由 |
|---|---|---|---|
| 1 | **Excel / HTML 報告** | Export, Snapshots | 所有客戶都能使用，不需要任何外部軟體 |
| 2 | **BCF 2.1** | Export（之後加 Import）, Snapshots, Viewpoints | 開放標準，支援的工具最多，一次就能涵蓋 Solibri、BIMcollab、Trimble Connect 等 |
| 3 | **Navisworks** | 待驗證（見 §11） | 最廣泛使用的碰撞工具。可以透過 BCF 間接互通；直接對接的方式待確認。在 M1 中也當作比對基準 |
| 4 | **ACC Issues（APS API）** | Export, Import, SyncStatus | 唯一能做到真正雙向同步狀態的；只有客戶有使用 ACC 時才需要 |
| 之後 | BCF 3.0、BCF API（線上服務）、其他平台 | — | 依客戶需求加入 |

## 9. Revit 內的 UI（Dockable Pane）

- 碰撞清單：可以依狀態、群組、規則、樓層篩選。
- 點一筆碰撞就跳到該視圖並選取相關元素（與 `show_clash` 共用同一段程式）。
- 每筆旁邊顯示 AI 的判讀理由，人員可以一鍵接受或改判。
- 有 AI 對話輸入框，接到 harness。
- 匯出按鈕：列出所有支援 Export 的連接器。

## 10. 里程碑

| 階段 | 內容 | 驗收 |
|---|---|---|
| **M0** | Windows 環境、版本分別 build 的 add-in 骨架、ExternalEvent 佇列、sidecar、mock transport、`get_model_overview` | 從 Mac 透過 MCP 取得 Revit 模型概要，至少兩個 Revit 版本都通過 |
| **M1** | Hard clash（含**連結模型**）、`list/show/get_clash_detail`、截圖、Excel/HTML 連接器 | 在基準模型上與 Navisworks 結果比對，差異都能解釋 |
| **M2** | Clearance、Duplicate、忽略規則、分群、狀態追蹤、**BCF 2.1 匯出** | 重新執行時能正確對應上次的碰撞；BCF 能在至少兩個外部工具中開啟並正確定位 |
| **M3** | AI 判讀與報告、Dockable Pane、Navisworks 互通（依 §11 驗證結果） | 人工抽樣 AI 判讀的準確率達到設定門檻 |
| **M4** | 修正建議與核准後執行、ACC 連接器（如有需要） | 修正可以 rollback，且完整寫入 audit log |

**基準模型**：M1 開始前，準備一組包含各專業與連結模型的測試模型，並人工標註正確答案。之後調整規則與 AI 判讀時，都用它做 regression 測試。

## 11. 待驗證事項

以下內容來自初步討論，寫入正式設計前要逐項確認：

- [ ] 各 Revit 版本對應的 .NET 版本，以及 Revit 2027 以後的目標框架。
- [ ] `DisplayUnitType` 移除、`ElementId` 改為 64 位元、`GroupTypeId` 等 API 變更確切發生在哪一版。
- [ ] RevitAPI 參考 NuGet 套件的名稱與授權（例如 Nice3point.Revit.Api）。
- [ ] 各工具對 BCF 的支援程度：版本（2.1 / 3.0）與方向（匯入或匯出），特別是 Navisworks、ACC、Revit 本身。
- [ ] Navisworks 的互通方式：外部碰撞結果能否匯入 Clash Detective、能否交換碰撞測試定義（XML）、是否需要另外開發 Navisworks 外掛。
- [ ] ACC Issues API 的權限、限制，以及是否能建立帶有視角的 Issue。
- [ ] Revit IFC GUID 的計算方式，確認與 IFC exporter 一致。
- [ ] 在 Revit 2025 以後的環境中，add-in 的相依 DLL 是否有隔離機制。

## 12. 開放問題

1. 客戶實際使用的 Revit 版本分布，用來決定支援下限。
2. 每家客戶最常用的碰撞或問題追蹤工具，用來決定連接器 3、4 的順序。
3. Sidecar 與 AI 端之間的認證、網路部署方式（區網、VPN 或雲端）。
4. 預計使用哪個 AI harness 與模型，以及 token 與成本預算。
5. Audit log 與碰撞資料的保存位置和保存期限（本機、共用伺服器，或寫入模型的 ExtensibleStorage）。
6. §5.4 的「專案座標」指哪一個座標系。建議：內部儲存用 Revit 內部座標，對外輸出（報告、BCF）用共用座標。
7. 規則中的 `link:<name>` 如何對應到連結。建議：使用 `RevitLinkType` 名稱去掉副檔名。
