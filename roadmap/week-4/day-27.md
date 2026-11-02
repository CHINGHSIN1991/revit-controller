# Day 27 · Navisworks 互通

> 本文件依計畫推估，開始前請依 Day 21 回顧結果修訂。

## 今日目標

今天是**決策日**。Navisworks 的互通方式在 SPEC §11 中仍是待驗證事項，今天要用實際測試回答它，然後二選一：

1. 某個方式驗證可行，而且一天內做得完 → 實作 Navisworks 連接器（或最小可用版本）。
2. 都不可行、或需要的工作量超過一天 → 寫評估報告 `docs/navisworks-interop.md`，排入後續計畫。

兩種產出都算完成。**不要預設哪個方式一定可行**，以下每個選項都只是要驗證的假設。

## 你會學到

- Navisworks 的擴充方式：.NET API、外掛架構、Clash Detective API
- Clash Detective 測試定義的 XML 匯入與匯出格式
- NWC / NWD 檔案與 Revit 模型的關係，Navisworks 中的元素如何對應回 Revit
- 怎麼用「實驗 + 證據 + 決策矩陣」做技術選型

---

## Step 1. 整理已有的證據

開始測試前，先回頭看前兩週的紀錄，可能已經有一部分答案：

| 來源 | 可能已有的證據 |
| --- | --- |
| Day 14（與 Navisworks 比對） | Navisworks 的版本與授權；NWC 匯出設定；Navisworks 中的元素屬性能不能看到 Revit 的 ElementId / UniqueId / IFC GUID |
| Day 20（BCF 外部工具驗證） | 是否已經在 Navisworks 加外掛的環境下開過 BCF；相機與元素定位是否正確 |
| SPEC §11 | 目前的查證狀態 |

把已知的寫進今天報告的「背景」段落，今天只測還不知道的部分。

## Step 2. 要驗證的三個選項

### 選項 A：BCF（透過外掛）

假設：Navisworks 本身或透過第三方外掛可以匯入 BCF，碰撞可以在 Navisworks 中以 issue 或 viewpoint 的形式出現。

| 實驗 | 要記錄的證據 |
| --- | --- |
| 用 M2 的 BCF 連接器匯出基準模型的 `.bcfzip`，在 Navisworks 中匯入 | 用了哪個外掛、版本、授權方式；是否需要另外付費 |
| 點選匯入的項目 | 相機是否正確；元素是否被選取或標示 |
| 檢查匯入後的位置 | 是在 Clash Detective 中、還是只是 viewpoint 或外掛自己的面板 |

注意：Navisworks 是否原生支援 BCF、哪些外掛支援哪個 BCF 版本，全部待查證。

### 選項 B：碰撞測試定義的 XML 交換

假設：Clash Detective 可以匯出與匯入碰撞測試的定義（XML），我們可以把 `ClashRule` 轉成這種格式，讓 Navisworks 用相同的規則跑一次。

| 實驗 | 要記錄的證據 |
| --- | --- |
| 在 Navisworks 中手動建立一個測試，匯出 XML | 檔案結構、Selection Set 怎麼描述、容差單位 |
| 修改 XML（例如容差）後再匯入 | 是否接受、修改是否生效 |
| 比對 `ClashRule` 的欄位 | 類別、連結來源、參數篩選、忽略規則各自能否表達 |

這個選項交換的是「規則」，不是「結果」。即使可行，也只能讓兩邊跑同樣的測試，碰撞結果與狀態不會自動對應。格式是否有公開文件、跨版本是否穩定，待查證。

### 選項 C：開發 Navisworks .NET 外掛

假設：用 Navisworks 的 .NET API（含 Clash Detective 相關 API）寫外掛，可以讀取或寫入 Clash Detective 的結果。

| 實驗 | 要記錄的證據 |
| --- | --- |
| 查官方 API 文件與 SDK 範例 | 外掛類型、能否讀寫碰撞結果、能否建立測試、支援的 Navisworks 版本 |
| 寫一個最小外掛（列出目前檔案中的碰撞測試名稱） | 能否載入、開發環境需求（.NET 版本、參考 DLL） |
| 在 Navisworks 中找一個元素，查它的屬性 | 能否取得 Revit 的 UniqueId 或 IFC GUID，用來對應我們的 `ClashRecord` |

```csharp
// 最小外掛的形狀（骨架，命名空間、基底類別、屬性全部待查證）
// [Plugin("RevitController.NavisProbe", "RC")]
// public class ProbePlugin : AddInPlugin
// {
//     public override int Execute(params string[] parameters)
//     {
//         // 取得目前文件 → 取得 Clash Detective 的測試清單 → 輸出名稱與結果數量
//         return 0;
//     }
// }
```

這個選項會多出一個要維護的產品（另一套版本相依、另一個安裝程式），評估時要算進去。

## Step 3. 元素對應的問題

不管選哪個方式，都要回答：**Navisworks 中的一個元素，能不能對應回我們的 `(link_instance_uid, unique_id)` 或 `ifc_guid`？**

| 路徑 | 依賴 |
| --- | --- |
| NWC 從 Revit 匯出時保留的屬性 | 匯出設定（待查證有哪些 ID 屬性被保留） |
| IFC GUID | Navisworks 載入的是 IFC 還是 RVT/NWC；GUID 是否一致（Day 10、Day 20 的查證結果） |
| 連結模型 | 在 Navisworks 中是一個檔案，還是展開成多個；同一個連結被放置多次時怎麼區分 |

如果元素無法可靠對應，選項 A 的定位與選項 C 的結果讀回都會受影響。這一項的結果要獨立寫在報告中。

## Step 4. 決策矩陣

每個選項依實驗結果填寫（✅ 驗證可行／⚠️ 部分可行／❌ 不可行／❓ 未驗證）：

| 評估面向 | A. BCF + 外掛 | B. 測試定義 XML | C. Navisworks 外掛 |
| --- | --- | --- | --- |
| 碰撞結果能進入 Clash Detective | | | |
| 相機與元素定位正確 | | | |
| 能交換規則 | | | |
| 狀態能雙向同步 | | | |
| 需要額外軟體或授權 | | | |
| 重用 M2 的 BCF 連接器 | | | |
| 開發成本（人天） | | | |
| 維護成本（版本相依、安裝） | | | |
| 客戶實際的使用方式是否需要（SPEC §12 開放問題 2） | | | |

選項不互斥。例如可能的結論是「短期用 A，規則同步用 B，C 等客戶需求明確再做」。

## Step 5. 依結論產出

**如果實作連接器：**

```csharp
// src/RevitController.Sidecar/Connectors/Navisworks/NavisworksConnector.cs（骨架）
public sealed class NavisworksConnector : IClashConnector
{
    public string Id => "navisworks";
    public string DisplayName => "Navisworks";
    public ConnectorCapabilities Capabilities => ConnectorCapabilities.Export; // 依驗證結果填寫，不要多宣稱
    // ExportAsync：依選定的方式產生檔案；ImportAsync / SyncStatusAsync 未支援時丟 NotSupportedException
}
```

- `Capabilities` 只能宣稱實際驗證過的能力。
- 外部 ID 存進 `external_refs["navisworks"]`。
- 如果選 A，這個連接器可能只是 BCF 連接器加上 Navisworks 專用的選項，不一定需要新的類別。

**如果寫評估報告**，`docs/navisworks-interop.md` 的大綱：

1. 背景（Step 1 已有的證據）
2. 測試環境（Navisworks 版本、外掛版本、授權）
3. 各選項的實驗與證據（附截圖、檔案範例）
4. 元素對應的結果（Step 3）
5. 決策矩陣（Step 4）
6. 建議與排程：放在 Week 5 之後的哪個階段、前提條件是什麼

## Step 6. 更新 SPEC

- SPEC §11：「Navisworks 的互通方式」與「各工具對 BCF 的支援程度（Navisworks 部分）」兩項，依結果打勾或寫明仍未確認的部分。
- SPEC §8.2：連接器 3 的「能力」欄位填入驗證結果。
- [PROGRESS.md](../PROGRESS.md) 技術債中「Navisworks 互通方式」一列，更新狀態。

---

## 驗收

- [ ] 三個選項都有實驗紀錄，或寫明為什麼沒測（例如沒有授權）
- [ ] 元素對應（Navisworks 元素 ↔ `ClashRecord`）的結果有獨立記錄
- [ ] 決策矩陣填寫完成，每一格都有證據或標示 ❓
- [ ] 產出二選一：Navisworks 連接器（只宣稱驗證過的能力），或 `docs/navisworks-interop.md`
- [ ] SPEC §11、§8.2 與 PROGRESS.md 已更新

## 常見問題

**Q：沒有 Navisworks Manage 授權怎麼辦？**
Day 14 應該已經準備好。如果只有 Navisworks Freedom 之類的檢視器，Clash Detective 相關的實驗做不了，這本身就是報告中的一項結論。

**Q：一天內做不完實驗？**
先做選項 A（成本最低，且重用 M2 的成果），B 和 C 只做文件查證，在報告中標示未實測。不要為了趕進度而跳過記錄證據。

**Q：外掛要用哪個 .NET 版本？**
依 Navisworks 版本而定，和 Revit 一樣可能有世代差異（待查證）。如果選 C，要在 [02-tech-stack.md](../02-tech-stack.md) 新增一張對照表。

## 學習筆記

（寫下今天的心得、卡關點、想問的問題）
