# Day 2 · Solution 結構與多版本 build

## 今日目標

建立 [SPEC.md](../../SPEC.md) §4 的 solution：`RevitController.Core`、`.Addin`、`.Sidecar`、`.Mock` 四個專案，加上兩個測試專案。Add-in 要能依 Revit 版本切換 build configuration（`Release.R22` … `Release.R26`），最後用一支 `scripts/build-all.ps1` 一次產出所有版本的 DLL。

今天只寫空殼，不呼叫任何 Revit API。重點是「專案之間的引用關係」和「同一份原始碼如何產出多個版本」。

## 你會學到

- SDK-style csproj 的結構，`TargetFramework` 與自訂 configuration 的寫法
- 為什麼 Core 用 netstandard2.0，它能被誰引用、少了哪些 API
- 條件編譯符號 `REVIT20XX` / `REVIT20XX_OR_GREATER` 的用法，以及怎麼避免 `#if` 散落各處
- RevitAPI.dll 為什麼要設 `Private=false`
- 怎麼在 Mac 上只 build 不需要 Revit 的專案

---

## Step 1. 建立 solution 與專案

在 repo 根目錄（Mac 或 Windows 都可以）：

```bash
dotnet new sln -n RevitController
dotnet new classlib -n RevitController.Core    -o src/RevitController.Core    -f netstandard2.0
dotnet new classlib -n RevitController.Addin   -o src/RevitController.Addin
dotnet new console  -n RevitController.Sidecar -o src/RevitController.Sidecar -f net8.0
dotnet new classlib -n RevitController.Mock    -o src/RevitController.Mock    -f net8.0
dotnet new xunit    -n RevitController.Core.Tests    -o tests/RevitController.Core.Tests
dotnet new xunit    -n RevitController.Sidecar.Tests -o tests/RevitController.Sidecar.Tests
dotnet sln add src/*/*.csproj tests/*/*.csproj
```

- 測試框架先用 xUnit，[02-tech-stack.md](../02-tech-stack.md) 標示為「待決定」，換掉也只影響 `tests/`。
- `dotnet new classlib -f netstandard2.0` 是否接受這個值依 SDK 版本而定（待查證）；不接受時先建立再手動改 csproj。

專案引用關係：

| 專案 | 目標框架 | 引用 |
| --- | --- | --- |
| `RevitController.Core` | netstandard2.0 | 無（**不可**引用 RevitAPI） |
| `RevitController.Addin` | net48 或 net8.0-windows（依 configuration） | Core、RevitAPI、RevitAPIUI |
| `RevitController.Sidecar` | net8.0 | Core |
| `RevitController.Mock` | net8.0 | 只引用 Core（Sidecar 引用 Mock，所以 Mock 不能反過來引用 Sidecar；共用介面放 Core，見 Day 4） |
| `*.Tests` | net8.0 | 被測的專案 |

```bash
dotnet add src/RevitController.Addin   reference src/RevitController.Core
dotnet add src/RevitController.Sidecar reference src/RevitController.Core src/RevitController.Mock
dotnet add src/RevitController.Mock    reference src/RevitController.Core
```

## Step 2. 共用設定：Directory.Build.props

放在 repo 根目錄，所有專案自動套用：

```xml
<Project>
  <PropertyGroup>
    <LangVersion>latest</LangVersion>
    <Nullable>enable</Nullable>
    <Version>0.1.0</Version>
    <TreatWarningsAsErrors>false</TreatWarningsAsErrors>
  </PropertyGroup>
</Project>
```

> netstandard2.0 與 net48 預設的 C# 版本較舊。`LangVersion latest` 讓語法一致，但有些功能需要執行環境支援（例如 `init`、default interface members），在 Core 和 Addin 中不能用（細節待查證）。

## Step 3. 為什麼 Core 是 netstandard2.0

| 誰引用 Core | 執行環境 | 能不能引用 netstandard2.0 |
| --- | --- | --- |
| net48 版 Add-in（Revit 2024 以前，待查證） | .NET Framework 4.8 | 可以 |
| net8 版 Add-in（Revit 2025 以後） | .NET 8 | 可以 |
| Sidecar / Mock / Tests | .NET 8 | 可以 |

netstandard2.0 是「兩邊都實作的 API 交集」。代價是：

- 沒有 `System.Text.Json`（要另外加 NuGet 套件）、沒有 `Span` 相關的很多 API、沒有 `record` 所需的 `IsExternalInit`（可以自己補一個 internal 類別，待查證）。
- 所以 Core 只放純資料模型與邏輯：DTO、RPC 訊息格式、錯誤碼。序列化細節留到 Day 4 決定。

## Step 4. Add-in 的多版本 configuration

核心想法：**一個 csproj，多個 configuration，每個 configuration 決定 Revit 版本、目標框架、RevitAPI 參考、條件編譯符號。**

`src/RevitController.Addin/RevitController.Addin.csproj` 骨架：

```xml
<PropertyGroup>
  <Configurations>Debug.R24;Release.R24;Debug.R25;Release.R25</Configurations>
  <RevitVersion>$([System.Text.RegularExpressions.Regex]::Match($(Configuration), 'R\d\d').Value.Substring(1))</RevitVersion>
</PropertyGroup>
<PropertyGroup Condition="'$(RevitVersion)' &lt; '25'">
  <TargetFramework>net48</TargetFramework>
</PropertyGroup>
<PropertyGroup Condition="'$(RevitVersion)' &gt;= '25'">
  <TargetFramework>net8.0-windows</TargetFramework>
</PropertyGroup>
```

- 上面用正規表示式從 configuration 名稱取出版本號，只是一種寫法；也可以每個 configuration 各寫一段 `PropertyGroup`，比較囉嗦但比較好讀。MSBuild 字串比較的行為請實測（待查證）。
- R22–R26 全部列出時，configuration 會有 10 個。先做 Day 1 實際安裝的兩個版本，其他版本等有環境再加。
- net48 與 net8 的分界以 Day 1 查證的對照表為準，不要寫死在多個地方。

條件編譯符號：

```xml
<PropertyGroup>
  <DefineConstants>$(DefineConstants);REVIT20$(RevitVersion)</DefineConstants>
  <DefineConstants Condition="'$(RevitVersion)' &gt;= '24'">$(DefineConstants);REVIT2024_OR_GREATER</DefineConstants>
  <DefineConstants Condition="'$(RevitVersion)' &gt;= '25'">$(DefineConstants);REVIT2025_OR_GREATER</DefineConstants>
</PropertyGroup>
```

### 避免 `#if` 散落各處

版本差異（例如 `ElementId.Value` 與 `IntegerValue`，版本界線見 Day 1，待查證）集中在一個 adapter：

```csharp
// src/RevitController.Addin/Compat/ElementIdCompat.cs
internal static class ElementIdCompat
{
    public static long ToLong(ElementId id)
    {
#if REVIT2024_OR_GREATER
        return id.Value;
#else
        return id.IntegerValue;
#endif
    }
}
```

其他程式只呼叫 `ElementIdCompat.ToLong(id)`。規則：**`#if` 只能出現在 `Compat/` 資料夾**，code review 時檢查這一點。

## Step 5. 參考 RevitAPI

兩種做法：

| 做法 | 優點 | 缺點 |
| --- | --- | --- |
| NuGet 套件（例如 `Nice3point.Revit.Api.RevitAPI`，**套件名稱與授權待查證**） | Mac 與 CI 上也拿得到參考組件；版本由套件號碼控制 | 非 Autodesk 官方發佈，需確認授權 |
| 直接參考安裝路徑 `C:\Program Files\Autodesk\Revit 20XX\RevitAPI.dll` | 一定與實際安裝的 Revit 一致 | 只能在裝了該版 Revit 的機器上 build |

不論哪一種，都要設定「不要複製到輸出目錄」：

```xml
<ItemGroup>
  <Reference Include="RevitAPI">
    <HintPath>$(RevitInstallDir)RevitAPI.dll</HintPath>
    <Private>false</Private>
  </Reference>
</ItemGroup>
```

**為什麼 `Private=false`**：RevitAPI.dll 由 Revit 自己載入。如果 add-in 資料夾裡也有一份，可能載入到錯誤的副本，或造成型別不相容。用 NuGet 時對應的設定是 `ExcludeAssets="runtime"`（待查證）。

## Step 6. 在 Mac 上只 build 不需要 Revit 的專案

Addin 專案在 Mac 上無法 build（net48 與 net8.0-windows 都需要 Windows 參考組件）。所以 `dotnet build` / `dotnet test` 不能直接對整個 sln 下。做法擇一：

1. 建立 solution filter `RevitController.Mac.slnf`，只列 Core、Sidecar、Mock 與測試專案（`.slnf` 格式待查證，可以用 `dotnet sln` 或 IDE 產生）。
2. 或在根目錄指令明確指定專案：`dotnet test tests/RevitController.Sidecar.Tests`。

本專案採做法 1；`.slnf` 若有問題再退回做法 2。Addin 加進 `.sln` 之後，Mac 上在根目錄直接打 `dotnet test` 會嘗試 build Addin 而失敗，所以根目錄 README 要寫明確的指令（`dotnet test RevitController.Mac.slnf`）。Day 7 的冷啟動演練「只照 README 做」，這一行就是演練能不能通過的關鍵。

## Step 7. build-all 腳本

`scripts/build-all.ps1`（Windows）：

```powershell
param([string[]]$Versions = @("24", "25"))

dotnet build src/RevitController.Sidecar -c Release
foreach ($v in $Versions) {
    dotnet build src/RevitController.Addin -c "Release.R$v"
    if ($LASTEXITCODE -ne 0) { throw "Build failed for R$v" }
}
```

輸出位置預期是 `src/RevitController.Addin/bin/Release.R25/<TargetFramework>/`。實際路徑以 build 結果為準，Day 7 的 `deploy-addin.ps1` 會用到。

完成後把實際可用的指令回填到 [02-tech-stack.md](../02-tech-stack.md) 的「常用指令」。

---

## 驗收

- [ ] `RevitController.sln` 包含四個 `src/` 專案與兩個 `tests/` 專案
- [ ] Core 沒有任何 RevitAPI 參考，目標框架為 netstandard2.0
- [ ] Windows 上 `./scripts/build-all.ps1` 成功產出兩個版本的 add-in DLL，各自的目標框架正確
- [ ] 輸出資料夾中**沒有** `RevitAPI.dll`、`RevitAPIUI.dll`
- [ ] Mac 上能 build 並 `dotnet test` Core、Sidecar、Mock（不含 Addin）
- [ ] `#if REVIT...` 只出現在 `Compat/` 資料夾
- [ ] 能說明 netstandard2.0 為什麼能同時被 net48 與 net8.0 引用

## 常見問題

**Q：可以讓 Addin 用 `TargetFrameworks`（複數）一次 build net48 與 net8 嗎？**
可以讓同一個 configuration 產出兩個框架，但 RevitAPI 參考與 `REVIT20XX` 符號是跟著 Revit 版本走，不是跟著框架走（例如 R22、R23、R24 都是 net48，API 卻不同）。所以用 configuration 區分版本比較直接。

**Q：Mac 上 `dotnet build` 整個 sln 出現 Addin 的錯誤？**
預期中的行為。改用 Step 6 的 solution filter 或指定專案路徑。

**Q：NuGet 的 RevitAPI 套件和實際安裝的版本有差異怎麼辦？**
以實際安裝的 Revit 為準。套件只用來編譯；執行時一律用 Revit 自己載入的 DLL。若遇到「編譯過但執行時找不到方法」，先檢查套件版本是否與 Revit 版本一致。

**Q：net8 版 add-in 要用 `net8.0` 還是 `net8.0-windows`？**
RevitAPIUI 牽涉 WPF，通常需要 `net8.0-windows` 加 `UseWPF`（待查證）。Week 4 做 Dockable Pane 時一定需要。

## 學習筆記

（寫下今天的心得、卡關點、想問的問題）
