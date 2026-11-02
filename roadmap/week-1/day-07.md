# Day 7 · 安裝程式、雙版本驗證與 M0 驗收

## 今日目標

把 add-in 與 sidecar 的部署寫成腳本，在兩個 Revit 版本上完整跑一次 M0 驗收流程，做一次回顧，並讀懂第二週要做什麼。

## 你會學到

- Add-in 的部署位置與多版本部署
- 用「冷啟動演練」檢查 README 與腳本是否真的可重現
- 怎麼把一週的問題整理成技術債與下週重點

---

## Step 1. 部署腳本

建立 `scripts/deploy-addin.ps1`，依版本把 build 輸出與 manifest 複製到對應的 Addins 資料夾：

```powershell
param([string[]]$Versions = @("2024", "2025"))

foreach ($v in $Versions) {
    $config = "Release.R$($v.Substring(2))"          # 2025 → Release.R25
    $src    = "src/RevitController.Addin/bin/$config"
    $dest   = "$env:APPDATA/Autodesk/Revit/Addins/$v"
    # 1. 複製 DLL 到 $dest/RevitController/
    # 2. 複製 RevitController.addin 到 $dest/，並把 <Assembly> 改成子資料夾路徑
}
```

（骨架；實際的輸出路徑以 Day 2 建好的專案為準）

- Revit 開著的時候 DLL 會被鎖住，部署前要先關閉 Revit。
- 正式的安裝程式（WiX、Inno Setup 等）留到 Week 5 決定，見 [PROGRESS.md](../PROGRESS.md) 技術債。

## Step 2. M0 Checklist

在**兩個 Revit 版本**上各跑一次：

| # | 項目 | 驗證方式 | R1 | R2 |
| --- | --- | --- | --- | --- |
| 1 | 所有版本的 add-in 都能 build | `scripts/build-all` 無錯誤 | [ ] | [ ] |
| 2 | Add-in 在 Revit 啟動時載入 | Add-in log 出現 `OnStartup` | [ ] | [ ] |
| 3 | ExternalEvent 佇列可以執行請求 | Sidecar 呼叫 `ping` 有回應 | [ ] | [ ] |
| 4 | Revit 重開後 sidecar 自動重連 | 關閉再開 Revit，`ping` 仍成功 | [ ] | [ ] |
| 5 | Modal 對話框開著時回傳 `Timeout`，不會卡死 | 開「選項」對話框後呼叫 `ping` | [ ] | [ ] |
| 6 | MCP client 看得到工具清單 | 從 Mac 用 MCP Inspector 連線 | [ ] | [ ] |
| 7 | 沒有 token 的請求被拒絕 | 不帶 token 連線 | [ ] | [ ] |
| 8 | `get_model_overview` 回傳正確資料 | 和 Revit 畫面中的樓層、連結數比對 | [ ] | [ ] |
| 9 | Audit log 有記錄每一次呼叫 | 查看 log 檔 | [ ] | [ ] |
| 10 | Mock 模式下 sidecar 測試全部通過 | Mac 上 `dotnet test` | [ ] | — |

全部打勾後，到 [PROGRESS.md](../PROGRESS.md) 把 Week 1 標記完成。

## Step 3. 冷啟動演練

模擬「新成員第一天」：

```bash
# 在 Mac
git clone <repo> rc-clone && cd rc-clone
dotnet test RevitController.Mac.slnf   # 只照 README 做，不看 roadmap；Mac 上不能 build Addin，所以用 solution filter
dotnet run --project src/RevitController.Sidecar -- --transport mock
```

```powershell
# 在 Windows
git clone <repo> rc-clone; cd rc-clone
./scripts/build-all.ps1
./scripts/deploy-addin.ps1
# 啟動 Revit → 從 Mac 呼叫 get_model_overview
```

任何一步卡住，代表 README 或腳本有缺漏，回去補。演練完刪掉 `rc-clone`。

## Step 4. 回填文件

- Day 1 – 6 的文件中，把實際跑過的版本號、指令輸出、修正過的程式碼回填。
- [02-tech-stack.md](../02-tech-stack.md) 的「常用指令」改成實際指令。
- 遇到的問題寫進 [troubleshooting/](../troubleshooting/)。

## 回顧（寫下來）

**最有收穫的一天是哪天？為什麼？**

>

**卡最久的地方？最後怎麼解的？**

>

**兩個 Revit 版本之間遇到了哪些差異？**

>

**目前最不確定的概念？（這會變成 Week 2 的學習重點）**

>

## 第二週預覽

第一週打通的是「AI → Sidecar → Add-in → Revit」的管線，只有一個唯讀工具。第二週開始做真正的碰撞檢討：

| 目標 | 牽涉的層 |
| --- | --- |
| 準備基準模型與正確答案 | 領域知識、fixtures |
| 碰撞規則與紀錄的資料模型、SQLite | Core、Sidecar |
| 收集元素與擷取幾何 | Add-in（`FilteredElementCollector`、`GeometryElement`） |
| 連結模型的座標轉換 | Add-in（`RevitLinkInstance`、`Transform`） |
| Hard clash 演算法與非同步 job | Add-in + Sidecar |
| `show_clash` 截圖 | Add-in（`View3D`、`ExportImage`） |
| Excel / HTML 報告 | Sidecar（連接器） |

詳細內容見 [week-2/README.md](../week-2/README.md)。

## Week 2 前要先想好的兩件事

1. **基準模型從哪裡來？** 需要建築、結構、MEP 分檔並互相連結的模型，最好是公司過去的真實專案（注意保密）。Day 8 一開始就要用。
2. **Navisworks 比對環境**：Day 14 要用 Navisworks Manage 跑同樣的規則，確認授權與安裝。

---

## 學習筆記

（寫下今天的心得、卡關點、想問的問題）
