# Day 32 · ACC 連接器（一）：認證與專案

> 本文件依計畫推估，開始前請依 Day 21 與 Day 28 回顧結果修訂。

## 今日目標

讓 sidecar 能連到客戶的 Autodesk Construction Cloud（ACC）專案：註冊 APS 應用程式、實作 OAuth 認證、安全保存 token，並取得 hub、project 與 Issues 容器。同時查證 ACC Issues API 的權限與限制，更新 SPEC §11。

ACC 連接器只有客戶有使用 ACC 時才需要（SPEC §8.2）。如果 Day 28 的使用者回饋顯示沒有客戶使用 ACC，今天改做查證與設計，實作延後。

> 本文件中所有 APS 的 endpoint 路徑、scope 名稱、欄位名稱都**沒有經過驗證**，一律以 APS 官方文件為準。

## 你會學到

- APS（Autodesk Platform Services）應用程式註冊的概念
- OAuth 2-legged 與 3-legged 的差異，以及要選哪一種
- Token 的保存與更新；機密資訊為什麼不能進版本控制
- ACC 的資料結構：hub → project → container → issues

---

## Step 1. 註冊 APS 應用程式

在 APS 開發者入口建立應用程式（流程與畫面待查證，以 APS 官方文件為準）：

1. 建立應用程式，取得 client ID 與 client secret。
2. 設定 callback URL（3-legged 需要，例如 sidecar 本機的 `http://localhost:<port>/acc/callback`）。
3. 勾選需要的 API（Issues 相關，待查證名稱）。
4. 請客戶的 ACC 管理員把這個應用程式加入他們帳號的整合清單（是否需要、怎麼加，待查證）。

**Client secret 一取得就放進安全儲存，不要貼到任何文件、issue、聊天紀錄。**

## Step 2. 2-legged vs 3-legged

| | 2-legged（client credentials） | 3-legged（authorization code） |
| --- | --- | --- |
| 代表誰 | 應用程式本身 | 登入的使用者 |
| 需要使用者登入 | 不需要 | 需要，在瀏覽器中登入 Autodesk 帳號 |
| Issue 的建立者 | 應用程式（或無法建立，待查證） | 該使用者 |
| 權限 | 依應用程式在帳號中的設定 | 依使用者在專案中的角色 |
| 適合 | 背景同步 | 互動式使用 |

ACC Issues API 是否支援 2-legged 存取，是今天最重要的查證項目（待查證，以 APS 官方文件為準）。本專案以互動式為主（SPEC §2），預設採 **3-legged**：Issue 的建立者是實際的人，權限也跟著使用者走，比較符合稽核需求。

3-legged 的流程（概念）：

```
使用者在 Pane 按「連接 ACC」
  → Sidecar 開啟瀏覽器到 APS 授權頁面（帶 client_id、scope、redirect_uri、state）
  → 使用者登入並同意
  → 瀏覽器導回 sidecar 的 callback，帶 authorization code
  → Sidecar 用 code 換 access token 與 refresh token
```

- `state` 參數要隨機產生並在 callback 時比對，防止 CSRF。
- 是否要用 PKCE（待查證 APS 是否支援或要求）。
- Callback 只監聽 `localhost`，用完即關。

## Step 3. Token 的保存與更新

| 項目 | 做法 |
| --- | --- |
| Client ID | 設定檔（不算機密，但也不要寫死在程式碼中） |
| Client secret | 環境變數或 Windows 的安全儲存（例如 DPAPI / Credential Manager，待決定） |
| Access token | 只放記憶體 |
| Refresh token | 加密後存到本機（DPAPI，以目前 Windows 使用者為範圍） |

```csharp
// src/RevitController.Sidecar/Connectors/Acc/AccAuth.cs（骨架）
public sealed class AccAuth
{
    private readonly ISecretStore _secrets;       // Windows 上用 DPAPI 實作，Mac 上用記憶體假實作
    private string? _accessToken;
    private DateTimeOffset _expiresAt;

    public async Task<string> GetAccessTokenAsync(CancellationToken ct)
    {
        if (_accessToken != null && DateTimeOffset.UtcNow < _expiresAt.AddMinutes(-2))
            return _accessToken;
        var refresh = await _secrets.GetAsync("acc.refresh_token", ct)
                      ?? throw new RcException("AccNotConnected");
        var t = await RefreshAsync(refresh, ct);  // POST 到 APS token endpoint（待查證，以 APS 官方文件為準）
        await _secrets.SetAsync("acc.refresh_token", t.RefreshToken, ct);  // refresh token 可能會輪替（待查證）
        (_accessToken, _expiresAt) = (t.AccessToken, DateTimeOffset.UtcNow.AddSeconds(t.ExpiresIn));
        return _accessToken;
    }
}
```

Token 的有效期限、refresh token 是否每次輪替，都以 APS 官方文件為準（待查證）。

`.gitignore` 檢查：任何 `*.secrets.json`、`appsettings.*.local.json` 都不能進版本控制。Day 34 再用掃描工具檢查一次 git 歷史。

## Step 4. ACC 的資料結構

```
hub（客戶的 Autodesk 帳號）
 └─ project（一個建案）
     └─ container（Issues 等服務的資料容器，ID 可能與 project ID 不同，待查證）
         └─ issues
```

取得的步驟（endpoint 路徑全部待查證，以 APS 官方文件為準）：

| 步驟 | 取得 | 備註 |
| --- | --- | --- |
| 1 | 使用者可存取的 hub 清單 | 讓使用者選擇 |
| 2 | Hub 下的 project 清單 | 讓使用者選擇 |
| 3 | Project 對應的 Issues container ID | 是否需要額外查詢，待查證 |
| 4 | Issue 類型、子類型、自訂欄位 | Day 33 建立 Issue 時需要 |

選好的 hub、project、container 存到 sidecar 設定（不是機密），之後每次使用。

```csharp
public interface IAccClient
{
    Task<IReadOnlyList<AccHub>> GetHubsAsync(CancellationToken ct);
    Task<IReadOnlyList<AccProject>> GetProjectsAsync(string hubId, CancellationToken ct);
    Task<string> GetIssuesContainerIdAsync(string hubId, string projectId, CancellationToken ct);
}
```

用 `HttpClient` 自己呼叫，或使用官方 / 社群的 .NET SDK（套件名稱與維護狀態待查證）。先用介面隔開，換實作不影響連接器。

## Step 5. 查證 ACC Issues API 的權限與限制

把結果寫進 SPEC §11 的「ACC Issues API 的權限、限制，以及是否能建立帶有視角的 Issue」：

| 問題 | 結果 |
| --- | --- |
| 支援 2-legged 還是只支援 3-legged | 待查證 |
| 需要的 scope | 待查證 |
| 使用者需要什麼專案角色才能建立 / 修改 Issue | 待查證 |
| 能否附加圖片（截圖） | 待查證 |
| 能否帶位置或視角、能否連結到模型中的元素 | 待查證 |
| 留言是否可以透過 API 讀寫 | 待查證 |
| Rate limit | 待查證 |
| 是否有 webhook 可通知 Issue 變更 | 待查證 |

查不到明確答案的，用測試專案實測，並記下實測日期。

## Step 6. MCP 工具與測試

新增兩個輔助工具（或併入 `list_connectors` 的回傳）：

| 工具 | 用途 |
| --- | --- |
| `acc_status` | 是否已連線、目前選擇的 hub / project（不回傳任何 token） |
| `acc_connect` | 回傳需要使用者在瀏覽器完成的登入說明；AI 不能代替使用者登入 |

測試：用 stub 的 `HttpMessageHandler` 模擬 APS 回應，在 Mac 上測 token 更新、過期、refresh 失敗的處理。

---

## 驗收

- [ ] APS 應用程式已註冊，client secret 只存在安全儲存中
- [ ] 3-legged 登入流程可以完成，`state` 有比對
- [ ] Access token 過期後能自動用 refresh token 更新
- [ ] 能列出 hub、project，並取得 Issues container
- [ ] `acc_status` 不會回傳任何 token 或 secret
- [ ] SPEC §11 的 ACC 項目已更新查證結果
- [ ] Repo 中搜尋不到任何 secret 或 token

## 常見問題

**Q：可以先用 2-legged 開發，比較簡單嗎？**
可以先用來驗證 API 能不能通，但要先確認 Issues API 是否支援（待查證）。正式版以 3-legged 為主，Issue 的建立者才會是真人。

**Q：Sidecar 在 Windows 上，使用者在 Mac 上用 AI，誰來登入？**
登入在 Revit 那台 Windows 上完成（從 Pane 觸發）。Token 保存在 Windows，以 Revit 使用者為範圍。

**Q：客戶有多個 ACC 帳號（hub）？**
v1 一次只連一個 project，切換時重新選擇。多專案同時同步留到之後。

## 學習筆記

（寫下今天的心得、卡關點、想問的問題）
