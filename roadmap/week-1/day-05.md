# Day 5 · Sidecar 的 MCP Server

## 今日目標

在 Sidecar 中架起 MCP server（Streamable HTTP transport），加上 bearer token 認證，註冊第一個工具 `ping`，並把工具呼叫轉成 Day 4 的 RPC 呼叫。每一次工具呼叫都寫入 audit log。最後從 Mac 用 MCP Inspector 或 AI harness 連線，看得到工具清單並能呼叫。

## 你會學到

- MCP 的核心概念：tools、JSON schema、request/response、transport
- 為什麼跨機器要用 Streamable HTTP，而不是 stdio
- Bearer token 與網路暴露範圍的基本安全措施
- 工具描述（description）怎麼寫，AI 才會正確使用
- 用 JSON lines 寫結構化 audit log

---

## Step 1. MCP 快速概念

| 概念 | 說明 | 在本專案 |
| --- | --- | --- |
| Server | 提供工具的一方 | Sidecar |
| Client / Host | 呼叫工具的一方 | AI harness（Claude Code、Agent SDK）、MCP Inspector |
| Tool | 有名稱、描述、輸入 JSON schema 的函式 | `ping`、`get_model_overview`（Day 6） |
| Transport | 訊息怎麼傳 | Streamable HTTP |
| `tools/list` | client 取得工具清單 | SDK 自動處理 |
| `tools/call` | client 呼叫工具，回傳 content | 我們實作的部分 |

MCP 本身也是 JSON-RPC 2.0，但它和 Day 4 的 RPC 是**兩條不同的線**：

```
AI harness ──MCP（HTTP）──▶ Sidecar ──自訂 JSON-RPC（named pipe）──▶ Add-in
```

不要把 MCP 的訊息直接轉送到 pipe。Sidecar 是轉接層：驗證、記錄、轉換參數與單位、精簡結果。

## Step 2. Transport 的選擇

| Transport | 怎麼運作 | 適合 |
| --- | --- | --- |
| stdio | client 啟動 server 程序，用 stdin/stdout 溝通 | 同一台機器 |
| Streamable HTTP | server 監聽 HTTP，client 用 POST 送訊息 | **跨機器**：AI 在 Mac，Sidecar 在 Windows |

AI harness 在 Mac、Revit 在 Windows，所以只能用 HTTP。

## Step 3. 加入 MCP C# SDK

MCP 官方有 C# SDK。套件名稱、版本與 API 都還在變動，以下全部**待查證**，請以 SDK 的 README 與範例為準：

```bash
cd src/RevitController.Sidecar
dotnet add package ModelContextProtocol.AspNetCore --prerelease   # 套件名稱待查證
```

Sidecar 從 console 專案改為 ASP.NET Core（csproj 的 SDK 改成 `Microsoft.NET.Sdk.Web`）。

`Program.cs` 骨架：

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddSingleton<IRevitTransport>(sp => TransportFactory.Create(args)); // pipe 或 mock
builder.Services.AddSingleton<AuditLog>();
builder.Services.AddMcpServer()          // 以下三行 API 名稱待查證
    .WithHttpTransport()
    .WithToolsFromAssembly();

var app = builder.Build();
app.UseMiddleware<BearerTokenMiddleware>();
app.MapMcp("/mcp");                      // 待查證
app.Run();
```

## Step 4. 定義工具

`src/RevitController.Sidecar/Mcp/RevitTools.cs`：

```csharp
[McpServerToolType]                      // attribute 名稱待查證
public sealed class RevitTools(IRevitTransport revit, AuditLog audit)
{
    [McpServerTool(Name = "ping")]
    [Description("檢查 Sidecar 與 Revit add-in 的連線。回傳 Revit 版本。不讀取或修改模型。")]
    public async Task<string> Ping(CancellationToken ct)
        => await audit.RunAsync("ping", null, () => revit.CallAsync("ping", null, ct));
        // RunAsync：計時、寫 audit log，並把 RpcResponse 轉成給 AI 的文字（成功時是 result 的 JSON，失敗時是錯誤說明）
}
```

### Description 的寫法

AI 只看得到工具名稱、描述與 schema。描述要回答：

| 問題 | 範例 |
| --- | --- |
| 這個工具做什麼 | 「回傳目前開啟模型的樓層、連結模型、各類別數量」 |
| 什麼時候該用 | 「開始任何分析前先呼叫，了解模型範圍」 |
| 有沒有副作用 | 「唯讀，不修改模型」 |
| 單位與限制 | 「長度一律為 mm」「結果超過 100 筆時分頁」 |
| 可能的錯誤 | 「Revit 有對話框開著時會回傳 Timeout，請使用者關閉後重試」 |

錯誤要以 AI 看得懂的文字回傳（例如 `NoActiveDocument：Revit 中沒有開啟的專案`），而不是只回數字碼。

## Step 5. Bearer token 認證

開發版的 token 寫在設定檔（見 [01-architecture.md](../01-architecture.md#開發版-vs-正式版)），例如 `appsettings.Development.json` 的 `Auth:Token`，**這個檔案不進版本控制**（加進 `.gitignore`，另外提供 `appsettings.Development.example.json`）。

```csharp
public async Task InvokeAsync(HttpContext ctx)
{
    var header = ctx.Request.Headers.Authorization.ToString();
    if (!FixedTimeEquals(header, $"Bearer {_token}"))
    {
        ctx.Response.StatusCode = 401;
        return;
    }
    await _next(ctx);
}
```

- 比對用固定時間比較（`CryptographicOperations.FixedTimeEquals`），避免時間差攻擊。
- Token 為空或沒設定時，Sidecar 要**拒絕啟動**，不能變成「不驗證」。

### 網路暴露範圍

| 設定 | 說明 |
| --- | --- |
| 監聽位址 | 只綁定區網或 VPN 的介面，不要綁 `0.0.0.0` 後直接暴露到外網 |
| Windows 防火牆 | 只對 Mac 的 IP 或區網開放該 port |
| HTTPS | 區網內先用 HTTP；跨網路前必須加 TLS 或走 VPN（部署方式見 SPEC §12 開放問題 3） |

Port 號碼自行決定並記錄在 [02-tech-stack.md](../02-tech-stack.md)。

## Step 6. Audit log

`src/RevitController.Sidecar/Audit/AuditLog.cs`：每次工具呼叫寫一行 JSON（JSON lines）。

```json
{"ts":"2026-11-03T10:00:00Z","tool":"ping","args":null,"ok":true,"error":null,"ms":42,"client":"192.168.1.20"}
```

| 欄位 | 用途 |
| --- | --- |
| `ts` | UTC 時間 |
| `tool` / `args` | 呼叫什麼、帶什麼參數 |
| `ok` / `error` | 成功或錯誤碼 |
| `ms` | 耗時，用來找慢的工具 |
| `client` | 來源 IP |

- 存放位置先用 `%LocalAppData%\RevitController\audit\audit-yyyyMMdd.jsonl`（保存位置與期限見 SPEC §12 開放問題 5）。
- JSON lines 的好處：可以用 `jq`、PowerShell `ConvertFrom-Json` 或之後匯入 SQLite 查詢。
- 結果內容可能很大，log 只記摘要（例如筆數），不記完整回傳。

## Step 7. 從 Mac 連線

在 Windows 啟動 Sidecar：

```powershell
dotnet run --project src/RevitController.Sidecar
```

在 Mac 用 MCP Inspector（啟動指令待查證，通常是 `npx @modelcontextprotocol/inspector`）：

1. Transport 選 Streamable HTTP，URL 填 `http://<windows-ip>:<port>/mcp`。
2. 加上 header `Authorization: Bearer <token>`。
3. 連線後看到 `ping`，呼叫後回傳 Revit 版本。

也可以把 Sidecar 加進 Claude Code 的 MCP 設定（指令格式待查證），讓 AI 直接呼叫 `ping`。

---

## 驗收

- [ ] Sidecar 以 Streamable HTTP 提供 MCP server
- [ ] 從 Mac 用 MCP Inspector 連線，看得到 `ping` 工具與它的描述
- [ ] 呼叫 `ping` 會經過 pipe 到 add-in，回傳 Revit 版本
- [ ] 不帶 token 或帶錯 token 的請求被拒絕（401）
- [ ] 沒有設定 token 時 Sidecar 拒絕啟動
- [ ] 每次工具呼叫都在 audit log 留下一行 JSON
- [ ] Token 設定檔沒有進版本控制

## 常見問題

**Q：Mac 連不到 Windows 上的 Sidecar？**
依序檢查：Sidecar 監聽的位址是否為 `localhost`（只接受本機）；Windows 防火牆是否開放該 port；Mac 與 Windows 是否在同一個網段。先在 Windows 本機用 `curl` 測試。

**Q：MCP Inspector 顯示連線成功，但工具清單是空的？**
確認工具類別有加上 SDK 要求的 attribute，並且有被註冊（`WithToolsFromAssembly` 或手動註冊，API 待查證）。

**Q：為什麼不讓 AI 直接連 add-in？**
Add-in 在 Revit 程序內，應該盡量小、相依少。認證、log、單位轉換、分頁、之後的 job 管理都放在 Sidecar，Revit 重開也不影響 AI 端的連線。

## 學習筆記

（寫下今天的心得、卡關點、想問的問題）
