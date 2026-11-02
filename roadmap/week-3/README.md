# Week 3 · M2：Clearance / Duplicate、忽略規則、分群、狀態追蹤、BCF 2.1

第二週做出了 hard clash（含連結模型）、`show_clash` 截圖與 Excel / HTML 報告。第三週要把「一次性的碰撞清單」變成「可以持續追蹤的碰撞檢討」：

- 補齊另外兩種碰撞類型：**Clearance**（淨空不足）與 **Duplicate**（重複放置）。
- 用**忽略規則**降低誤報，用**分群**把幾百筆碰撞收斂成幾十個問題。
- 重新執行時用穩定的 `clash_id` 對應上次的結果，讓**狀態**（New → Active → Resolved / Approved / Ignored）能延續。
- 匯出 **BCF 2.1**，在外部工具中開啟並定位到正確的元素與視角。

**本週的程式碼都是骨架，撰寫時沒有編譯或執行過**（沒有 Windows、沒有 Revit，Mac 上也沒有 .NET SDK）。標示「（待查證）」的地方要對照實際的 RevitAPI 文件或 buildingSMART 規格確認。完成後請回填實際結果，見 [../README.md](../README.md)「文件的可信度」。

> 本週的檔案路徑已與 [week-2](../week-2/README.md) 的命名對齊；實作完成後以實際專案為準。

## 每日日程

| Day | 主題 | 文件 |
| --- | --- | --- |
| Day 15 | Clearance 與 Duplicate | [day-15.md](day-15.md) |
| Day 16 | 忽略規則 | [day-16.md](day-16.md) |
| Day 17 | 分群 | [day-17.md](day-17.md) |
| Day 18 | 狀態追蹤與重跑比對 | [day-18.md](day-18.md) |
| Day 19 | BCF 2.1 結構與視角轉換 | [day-19.md](day-19.md) |
| Day 20 | BCF 連接器與外部工具驗證 | [day-20.md](day-20.md) |
| Day 21 | 整合測試、效能基準、M2 驗收 | [day-21.md](day-21.md) |

## Week 3 Definition of Done

重跑時能正確對應前次碰撞與狀態；BCF 2.1 能在至少兩個外部工具中開啟並正確定位。

拆成可以打勾的項目：

- [ ] `hard`、`clearance`、`duplicate` 三種規則都能執行，結果寫入 Result Store
- [ ] 忽略規則（`hosted`、`joined`、`same_system`、自訂）在基準模型上減少誤報，且沒有漏掉任何標註的真碰撞
- [ ] `list_clashes` 可以依群組查詢；群組數明顯少於碰撞數
- [ ] 修改模型後重跑：新增的為 `new`、仍存在的保留原狀態、消失的自動變成 `resolved`
- [ ] `update_clash` 可以更新狀態、負責人、留言，且有狀態變更歷史
- [ ] BCF 2.1 匯出檔在至少兩個外部工具中開啟，相機與元素定位正確
- [ ] 有一份效能基準（數萬元素模型的執行時間與記憶體）

## 本週程式碼的落點

```
src/RevitController.Core/
├── Models/
│   ├── ClashRule.cs              ← 擴充 type: clearance / duplicate、ignore 清單（Day 15、16）
│   ├── ClashRecord.cs            ← 加 group_id、status、comments（Day 17、18）
│   └── ClashStatus.cs            ← 狀態 enum 與合法轉換表（Day 18）
├── Clash/
│   ├── （ClashId 沿用 Models/ClashId.cs，Day 15 加入正規化排序）
│   ├── IgnoreRules.cs            ← 規則名稱解析、自訂條件的資料結構（Day 16）
│   ├── Grouping/
│   │   ├── UnionFind.cs          ← 共同元素分群（Day 17）
│   │   └── SpatialGrouper.cs     ← 網格雜湊 + 樓層 / 軸線（Day 17）
│   └── RerunDiff.cs              ← 新舊結果比對（Day 18）
└── Units/Mm.cs                   ← 沿用 Day 9，補上 Clearance 需要的換算（Day 15）

src/RevitController.Addin/
├── Clash/
│   ├── ClearanceDetector.cs      ← 擴大 bbox 初篩 + 最短距離（Day 15）
│   ├── DuplicateDetector.cs      ← 同類別、同類型、幾何重疊（Day 15）
│   ├── SolidDistance.cs          ← solid 最短距離近似（Day 15）
│   └── IgnoreEvaluator.cs        ← hosted / joined / same_system 的 Revit API 判斷（Day 16）
└── Diagnostics/PhaseTimer.cs     ← 分段計時（Day 21）

src/RevitController.Sidecar/
├── Store/
│   ├── Migrations/003_status_history.sql   ← 狀態歷史、群組表（Day 17、18）
│   └── ResultStore.cs            ← 擴充：依群組查詢、upsert、歷史（Day 17、18）
├── Mcp/
│   └── ClashTools.cs             ← list_clashes 加 group 篩選與群組摘要（Day 17）、update_clash（Day 18）
└── Connectors/Bcf21/
    ├── BcfModel.cs               ← Topic / Viewpoint / Camera DTO（Day 19）
    ├── BcfMapper.cs              ← ClashRecord → BCF（Day 19）
    ├── BcfXmlWriter.cs           ← System.Xml.Linq 輸出（Day 19）
    └── Bcf21Connector.cs         ← IClashConnector、打包 .bcfzip（Day 20）

tests/
├── RevitController.Core.Tests/   ← ClashId、UnionFind、SpatialGrouper、RerunDiff、狀態轉換（Mac 可跑）
└── RevitController.Sidecar.Tests/← BcfMapper、BCF XML、ZIP 結構（Mac 可跑）

docs/benchmarks/m2.md             ← 效能基準（Day 21）；docs/ 是本週新增的資料夾，也可以改放 roadmap/ 底下
```

本週有一半的邏輯（ID、分群、重跑比對、狀態機、BCF 轉換）放在 Core 或 Sidecar，**可以在 Mac 上用 Mock 寫測試**。只有距離計算與忽略規則的判斷必須在 Windows 上跑 Revit 驗證。

## 本週的學習重點

| 主題 | 內容 |
| --- | --- |
| Revit 單位 | 內部單位是英尺；`UnitUtils.ConvertToInternalUnits`、`UnitTypeId.Millimeters`；對外一律 mm |
| 幾何距離 | Revit API 沒有現成的 solid 對 solid 最短距離；用 face 投影或 mesh 近似，精度與效能的取捨 |
| 忽略規則的 API | `FamilyInstance.Host`、`JoinGeometryUtils.AreElementsJoined`、`MEPSystem` / `ConnectorManager`；這些都只在同一個文件內有效 |
| 誤報與漏報 | 忽略規則寧可保守；用基準模型的標註量測兩者 |
| 分群演算法 | union-find（disjoint set）、網格雜湊、DBSCAN 的概念 |
| 穩定 ID | `hash(rule_id, ref_a, ref_b)` 前要先排序；UniqueId 在刪除重建後會變 |
| 狀態機 | 合法轉換表、誰可以觸發、變更歷史（誰、何時、為什麼） |
| BCF 2.1 | ZIP 內的檔案結構、`markup.bcf`、`viewpoint.bcfv`、相機、IFC GUID 定位、Topic GUID 的延續 |
| 外部工具互通 | 各工具支援的 BCF 版本與方向**尚未查證**，Day 20 實測後更新 SPEC §11 |
| 效能量測 | `Stopwatch` 分段計時、記憶體量測、瓶頸定位 |

## 本週最大的不確定性

1. **Solid 最短距離**：Clearance 的精度直接取決於 Day 15 選的近似方法。先求「不漏報」，精度可以後續提升。
2. **BCF 在外部工具的行為**：相機座標系、單位、元素定位都要在 Day 20 實測。規格細節以 buildingSMART BCF-XML 的官方文件為準。
3. **IFC GUID 的正確性**：Day 10 算出的 `ifc_guid` 是否與外部工具載入的 IFC 一致（SPEC §11），Day 20 會第一次真正驗證。
