# 02 · 技術規格與開發環境

## 技術選型

| 層 | 技術 | 為什麼 |
| --- | --- | --- |
| Revit Add-in | C#，依 Revit 版本 target .NET Framework 4.8 或 .NET 8 | Revit API 只提供 .NET 介面 |
| 共用程式庫 | C#，netstandard2.0 | 同時被 net48 與 net8.0 引用 |
| Sidecar | C#，.NET 8 | 與 Core 共用型別；跨平台，可在 Mac 開發 |
| MCP Server | MCP 官方 C# SDK（待查證套件名稱與版本） | 支援 Streamable HTTP transport |
| Add-in ↔ Sidecar | JSON-RPC 2.0 over Named Pipe | 本機、低延遲、可限制存取權限 |
| 資料庫 | SQLite（Microsoft.Data.Sqlite） | 單檔、零設定 |
| 測試 | xUnit（待決定） | Core 與 Sidecar 可在 Mac 與 CI 執行 |
| Excel | ClosedXML（待確認授權） | Day 13 |
| UI（Week 4） | WPF（Dockable Pane） | Revit 內嵌 UI 的標準做法 |

## 開發環境

撰寫時的實際狀況：Mac 上**沒有** .NET SDK，也還沒有 Windows 機器。下表的版本欄請在 Day 1 實際安裝後填入。

### Mac（AI 端、Sidecar 開發）

| 工具 | 需求 | 本機版本 | 狀態 |
| --- | --- | --- | --- |
| .NET SDK | 8.x | 待填 | ❌ 未安裝 |
| Git | 任一近期版本 | 待填 | ✅ |
| MCP client | Claude Code 或 MCP Inspector | 待填 | — |
| IDE | VS Code（C# Dev Kit）或 Rider | 待填 | — |

### Windows（Revit 端）

| 工具 | 需求 | 本機版本 | 狀態 |
| --- | --- | --- | --- |
| Windows | x64（不要用 Apple Silicon 上的 Windows ARM 當主要環境） | 待填 | ❌ 尚未準備 |
| Revit | 至少兩個版本：一個 net48 世代、一個 net8 世代 | 待填 | ❌ |
| .NET SDK | 8.x | 待填 | ❌ |
| .NET Framework 4.8 Developer Pack | 給 net48 版 add-in | 待填 | ❌ |
| IDE | Visual Studio 或 Rider（要能 attach 到 Revit.exe 除錯） | 待填 | ❌ |
| Navisworks Manage | Day 14 比對用 | 待填 | ❌ |

## Revit 版本對照表（Day 1 查證後填寫）

| Revit | .NET | 已安裝在開發機 | 備註 |
| --- | --- | --- | --- |
| 2022 | .NET Framework 4.8（待查證） | | |
| 2023 | .NET Framework 4.8（待查證） | | |
| 2024 | .NET Framework 4.8（待查證） | | `ElementId` 64 位元（待查證） |
| 2025 | .NET 8（待查證） | | |
| 2026 | .NET 8（待查證） | | |
| 2027 | 待查證 | | |

## 目錄結構（Week 1 完成時）

```
revit-controller/
├── RevitController.sln
├── Directory.Build.props          ← 共用設定：LangVersion、Nullable、版本號
├── src/
│   ├── RevitController.Core/
│   │   ├── Rpc/                   ← JSON-RPC 訊息、錯誤碼、IRevitTransport 介面
│   │   └── Models/                ← ModelOverview 等 DTO
│   ├── RevitController.Addin/
│   │   ├── App.cs                 ← IExternalApplication
│   │   ├── Execution/             ← RequestQueue、RevitEventHandler
│   │   ├── Rpc/                   ← PipeServer、方法分派
│   │   ├── Handlers/              ← model.overview 等實作
│   │   └── RevitController.addin  ← manifest
│   ├── RevitController.Sidecar/
│   │   ├── Program.cs
│   │   ├── Mcp/                   ← MCP tools
│   │   ├── Revit/                 ← PipeTransport（介面在 Core，避免與 Mock 循環引用）
│   │   └── Audit/
│   └── RevitController.Mock/      ← MockTransport + fixtures
├── tests/
│   ├── RevitController.Core.Tests/
│   └── RevitController.Sidecar.Tests/
├── fixtures/
│   └── mock/                      ← Mock transport 用的 JSON
├── installer/
├── RevitController.Mac.slnf       ← 只含 Mac 上能 build 的專案
└── scripts/                       ← build-all、deploy-addin
```

## 常用指令（Day 2 建好後回填實際指令）

```bash
# 在 Mac 或 Windows：建置並測試 Core + Sidecar
dotnet build src/RevitController.Sidecar
dotnet test RevitController.Mac.slnf

# 在 Windows：建置所有 Revit 版本的 add-in
dotnet build src/RevitController.Addin -c Release.R25

# 用 Mock 啟動 sidecar（Mac 開發用）
dotnet run --project src/RevitController.Sidecar -- --transport mock
```
