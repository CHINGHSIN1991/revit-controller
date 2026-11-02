# Day 1 · 環境建置與版本查證

## 今日目標

準備好一台可以從 Mac 遠端操作的 Windows x64 開發機，裝好至少兩個 Revit 版本與 .NET 工具鏈；Mac 上也裝好 .NET SDK。最後把 Revit 版本與 .NET 的對照表查證清楚，填進 [02-tech-stack.md](../02-tech-stack.md)。

今天不寫程式。

## 你會學到

- Revit 為什麼只能在 Windows x64 上開發與測試
- Revit 各版本對應的 .NET 執行環境（.NET Framework 4.8 vs .NET 8），以及這對 build 的影響
- Revit 開發機的授權選項
- 怎麼確認自己裝的是哪個 .NET SDK、哪個 Revit 版本

---

## Step 1. 決定 Windows 開發機的形式

| 選項 | 優點 | 缺點 |
| --- | --- | --- |
| 實體 Windows x64 電腦 | 效能穩定、GPU 正常、除錯最方便 | 需要另一台電腦 |
| 雲端 GPU VM（Azure / AWS） | 不用買硬體、可以快照備份 | 按時計費；遠端桌面延遲 |
| Apple Silicon Mac 上的 Windows ARM VM | 方便 | Revit 是 x64 程式，要靠模擬執行，**非官方支援**、GPU 與效能不穩定。**不要當主要測試環境** |

決定後，在 [02-tech-stack.md](../02-tech-stack.md) 的 Windows 表格中記錄選擇與規格（CPU、RAM、GPU）。

> Revit 的硬體需求依版本不同，請查 Autodesk 官方的系統需求頁面。大型模型的碰撞計算吃 CPU 與記憶體，RAM 建議至少 32 GB（待依實際模型大小調整）。

## Step 2. 準備 Revit 授權

| 方式 | 適用 |
| --- | --- |
| 公司既有授權 | 最直接；確認一個授權能裝幾個版本 |
| 試用版 | 有天數限制，只適合短期驗證 |
| Autodesk 開發者計畫（待查證是否仍提供） | 長期開發 |

**至少裝兩個版本：一個 .NET Framework 4.8 世代、一個 .NET 8 世代**（對照表見 Step 4）。這樣從 Day 2 開始就能確認多版本 build 是可行的，而不是到最後才發現問題。

## Step 3. 安裝開發工具

### Windows

依序安裝：

1. Revit（兩個版本）
2. Git
3. .NET SDK 8.x
4. .NET Framework 4.8 Developer Pack（給 net48 版的 add-in）
5. Visual Studio（勾選「.NET 桌面開發」）或 JetBrains Rider
6. 遠端連線：開啟 RDP，或安裝 OpenSSH Server

確認：

```powershell
dotnet --list-sdks          # 應列出 8.x
git --version
```

確認 Revit 的安裝位置（之後設定除錯時要用）：

```powershell
dir "C:\Program Files\Autodesk\" | findstr Revit
```

### Mac

Mac 上用來開發 Core、Sidecar、Mock，以及執行 AI harness：

```bash
brew install --cask dotnet-sdk     # 或從 Microsoft 官網下載 .NET 8 SDK
dotnet --list-sdks
```

> 撰寫這份文件時 Mac 上**沒有**安裝 .NET SDK，今天裝好後請回填版本到 [02-tech-stack.md](../02-tech-stack.md)。

### Mac 到 Windows 的連線

| 用途 | 工具 |
| --- | --- |
| 操作 Revit 畫面 | Microsoft Remote Desktop（Windows App） |
| 傳送程式碼 | Git（兩邊都 clone 同一個 repo），不要用資料夾同步，避免 `bin/`、`obj/` 互相覆蓋 |
| 執行指令 | SSH（選用） |

## Step 4. 查證 Revit 與 .NET 的對照

這是今天最重要的一步，結果會決定 Day 2 的 build 設定。

對每一個要支援的 Revit 版本，確認：

1. **目標框架**：Revit 安裝資料夾中的 `Revit.exe.config`、或 Revit SDK 文件中寫明的 .NET 版本。
2. **RevitAPI.dll 的位置**：通常在 `C:\Program Files\Autodesk\Revit 20XX\RevitAPI.dll`。
3. **重大 API 變更**：查各版本的「What's New in the Revit API」文件，特別注意：
   - 單位 API（`DisplayUnitType` → `ForgeTypeId` / `UnitTypeId`）在哪一版移除舊 API（待查證）
   - `ElementId` 改成 64 位元（`.Value`）是哪一版，舊的 `IntegerValue` 何時移除（待查證）
   - 參數群組改用 `GroupTypeId` 是哪一版（待查證）
   - Revit 2027 的目標框架（待查證）

把結果填進 [02-tech-stack.md](../02-tech-stack.md) 的「Revit 版本對照表」，並同步更新 [SPEC.md](../../SPEC.md) §4 與 §11 中對應的待驗證項目。

> 為什麼這很重要：.NET Framework 4.8 和 .NET 8 是不同的執行環境，同一個 DLL 不能同時給兩邊使用。Day 2 的多版本 build 就是為了這件事。

## Step 5. 確認 Revit 可以正常啟動

1. 兩個版本都開啟一次，完成首次啟動設定。
2. 各開一個 Revit 內附的範例專案，確認 3D 視圖能正常顯示（測試 GPU 與遠端桌面是否正常）。
3. 記下每個版本的實際版本號（說明 → 關於），填入 [02-tech-stack.md](../02-tech-stack.md)。

---

## 驗收

- [ ] Windows x64 開發機已就緒，可以從 Mac 遠端操作
- [ ] 至少兩個 Revit 版本可以正常開啟範例專案（一個 net48 世代、一個 net8 世代）
- [ ] Windows 上 `dotnet --list-sdks` 列出 8.x；已安裝 .NET Framework 4.8 Developer Pack
- [ ] Mac 上 `dotnet --list-sdks` 列出 8.x
- [ ] [02-tech-stack.md](../02-tech-stack.md) 的環境表與版本對照表已填寫
- [ ] SPEC §11 中與版本相關的待驗證項目已更新
- [ ] 能說明為什麼同一個 add-in DLL 不能同時給 Revit 2024 和 2025 使用

## 常見問題

**Q：可以只裝一個 Revit 版本先開始嗎？**
可以，但 Day 2 的多版本 build 就無法實際驗證。至少在 Day 7 驗收前裝好第二個版本。

**Q：遠端桌面中 Revit 的 3D 視圖是黑的或很慢？**
多半是 GPU 或遠端桌面的顯示設定問題。雲端 VM 要選有 GPU 的規格並安裝顯示驅動。先確認在本機登入時是否正常。

**Q：Mac 上的 `dotnet` 指令找不到？**
用 Homebrew 安裝後可能需要重開 Terminal，或確認 `/usr/local/share/dotnet` 是否在 `PATH` 中。

## 學習筆記

（寫下今天的心得、卡關點、想問的問題）
