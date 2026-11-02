# Day 8 · 基準模型與資料模型

## 今日目標

準備一組建築、結構、MEP 分檔並互相連結的**基準模型**，人工標註已知的碰撞，作為之後所有比對與 regression 測試的正確答案。同時在 Core 實作 `ClashRule` 與 `ClashRecord`，在 Sidecar 建好 SQLite Result Store 與 schema migration，並加上規則管理的 MCP 工具。

除了準備基準模型，今天的程式碼都可以在 Mac 上完成與測試。

## 你會學到

- 碰撞檢討的業務背景：各專業之間常見的碰撞類型
- 基準資料集（ground truth）怎麼設計：真碰撞、可接受的碰撞、邊界情況
- `ClashRule` / `ClashRecord` 與 SPEC §5.2、§5.4 的對應
- 穩定 ID 的設計：用雜湊組合規則與元素參照
- Microsoft.Data.Sqlite 的基本用法與簡單的 schema migration

---

## Step 1. 了解常見的碰撞類型

碰撞檢討的目的是在施工前找出「圖上畫得下、現場做不出來」的地方。常見組合：

| A 組 | B 組 | 典型問題 | 通常的判讀 |
| --- | --- | --- | --- |
| 風管、水管 | 結構梁 | 管線穿梁 | 真碰撞，除非有預留開口 |
| 管線 | 天花板 | 管線低於天花板 | 真碰撞 |
| 管線 | 牆 | 管線穿牆 | 多半可接受（會開孔），但防火牆要注意 |
| 電纜架 | 風管 | 機電互撞 | 真碰撞 |
| 結構柱 | 建築柱 | 重複建模 | 屬於 Duplicate，Week 3 處理 |
| 門 | 結構牆 | 門嵌在牆裡 | 不是碰撞（hosted），Week 3 的忽略規則處理 |

這張表也是基準模型要涵蓋的範圍。

## Step 2. 準備基準模型

需求：

1. 至少三個檔案：建築（ARC）、結構（STR）、機電（MEP）。
2. 一個主模型（例如 MEP）連結另外兩個；**至少一個連結被放置兩次**（Day 10 要測）。
3. 建議另外準備一個「未載入的連結」，用來測 `LinkNotLoaded`。
4. 模型不要太大：先以一到兩層樓、數百個元素為目標，方便人工標註。

來源優先順序：公司過去的真實專案（注意保密，只截取一部分）→ Revit 內附範例 → 自己畫。

> 模型檔案通常很大，而且可能含客戶資料，**不要直接放進 Git**。在 `fixtures/baseline/README.md` 記錄檔名、存放位置（例如公司 NAS）、連結方式與 Revit 版本即可。

## Step 3. 人工標註正確答案

`fixtures/baseline/expected-clashes.json` 記錄每一筆已知的碰撞：

```jsonc
{
  "model_set": "baseline-v1",
  "rules": ["mep-vs-struct-beam", "duct-vs-ceiling"],
  "expected": [
    {
      "rule_id": "mep-vs-struct-beam",
      "a": { "file": "MEP", "unique_id": "..." },
      "b": { "file": "STR", "link_instance": "STR#1", "unique_id": "..." },
      "kind": "true_clash",          // true_clash | acceptable | edge_case
      "note": "風管穿梁，梁沒有開口"
    }
  ]
}
```

設計原則：

| 類型 | 為什麼要放 | 例子 |
| --- | --- | --- |
| 真碰撞 | 驗證 recall（有沒有漏抓） | 管線穿梁 |
| 可接受的碰撞 | Week 3 忽略規則與 Week 4 AI 判讀的評估資料 | 套管穿牆 |
| 邊界情況 | 驗證容差與幾何精度 | 剛好貼齊、相交 1 mm、族群內有巢狀族群 |
| 不該碰撞 | 驗證 precision（有沒有誤報） | 管線離梁 5 mm |

UniqueId 可以在 Revit 中選取元素後用 Revit Lookup 之類的工具查（待查證工具在目前版本的名稱與安裝方式）。

## Step 4. Core：`ClashRule` 與 `ClashRecord`

欄位**逐一對照** SPEC §5.2 與 §5.4，不要自己加減。JSON 用 snake_case，C# 用 PascalCase，在序列化設定統一轉換。

```csharp
// src/RevitController.Core/Models/ClashRule.cs
public sealed class ClashRule
{
    public string Id { get; set; } = "";
    public string Name { get; set; } = "";
    public ClashType Type { get; set; }          // Hard | Clearance | Duplicate
    public double ToleranceMm { get; set; }
    public ClashElementSet SetA { get; set; } = new();
    public ClashElementSet SetB { get; set; } = new();
    public List<object> Ignore { get; set; } = new();   // Week 3 再定義型別
}

public sealed class ClashElementSet
{
    public string Source { get; set; } = "any";  // "host" | "link:<name>" | "any"
    public List<string> Categories { get; set; } = new();  // "OST_DuctCurves"
    public ParameterFilterSpec? Filter { get; set; }
}
```

```csharp
// src/RevitController.Core/Models/ClashRecord.cs（節錄）
public sealed class ClashRecord
{
    public string ClashId { get; set; } = "";
    public string RuleId { get; set; } = "";
    public ClashType Type { get; set; }
    public List<ClashElementRef> Elements { get; set; } = new();  // 固定兩個
    public double[] Point { get; set; } = new double[3];           // mm，專案座標
    public double? VolumeMm3 { get; set; }
    public string? GridRef { get; set; }
    public ClashStatus Status { get; set; } = ClashStatus.New;
    // group_id、assignee、comments、ai_assessment、viewpoint、
    // snapshot_path、external_refs：照 SPEC §5.4 補齊，本週不一定會填
}
```

命名注意：Revit API 本身有 `Autodesk.Revit.DB.ElementSet` 類別，所以這裡用 `ClashElementSet`，避免 add-in 中名稱衝突。

注意：Core 是 netstandard2.0，`init`、`record` 等語法要看 LangVersion 與 polyfill（待查證 Day 2 的 `Directory.Build.props` 設定）。

## Step 5. 穩定 ID

`clash_id = hash(rule_id, ref_a, ref_b)`，其中元素參照是 `(link_instance_uid, unique_id)`（SPEC §6）。

```csharp
public static class ClashId
{
    public static string Create(string ruleId, ClashElementRef a, ClashElementRef b)
    {
        // 1. 把 a、b 依字串排序，讓 (a,b) 與 (b,a) 得到同一個 ID
        // 2. 組成 "ruleId|linkA/uidA|linkB/uidB"
        // 3. SHA-256 取前 16 bytes，轉成 hex
        throw new NotImplementedException();
    }
}
```

| 設計問題 | 決定 |
| --- | --- |
| 為什麼不用 `ElementId`？ | `ElementId` 在不同檔案、甚至同一檔案重新開啟後不保證不變；`UniqueId` 比較穩定 |
| 為什麼要排序？ | A / B 組可能有重疊類別，順序不同時不能變成兩筆 |
| 為什麼要包含 link instance？ | 同一個連結放兩次，裡面同一個元素會出現在兩個位置，是兩筆不同的碰撞 |

先寫單元測試：同樣輸入得到同樣 ID、交換 a/b 得到同樣 ID、不同 link instance 得到不同 ID。

## Step 6. Sidecar：SQLite Result Store 與 migration

使用 `Microsoft.Data.Sqlite`。Migration 用最簡單的做法：一個 `schema_version` 表，加上依序編號的 SQL 檔。

```sql
-- src/RevitController.Sidecar/Store/Migrations/001_init.sql
CREATE TABLE clash_rules (id TEXT PRIMARY KEY, json TEXT NOT NULL, updated_at TEXT NOT NULL);
CREATE TABLE clash_runs  (id TEXT PRIMARY KEY, rule_ids TEXT NOT NULL, started_at TEXT, finished_at TEXT, status TEXT);
CREATE TABLE clashes (
  clash_id TEXT NOT NULL, run_id TEXT NOT NULL, rule_id TEXT NOT NULL,
  status TEXT NOT NULL, level TEXT, json TEXT NOT NULL,
  PRIMARY KEY (clash_id, run_id)
);
```

```csharp
public sealed class ResultStore
{
    public ResultStore(string dbPath) { /* new SqliteConnection($"Data Source={dbPath}") */ }
    public void Migrate() { /* 讀 schema_version → 依序執行還沒跑過的 .sql，包在 transaction 中 */ }
    public void UpsertRule(ClashRule rule) { }
    public IReadOnlyList<ClashRule> ListRules() => throw new NotImplementedException();
}
```

| 做法 | 說明 |
| --- | --- |
| 整筆存 JSON + 幾個常用查詢欄位 | 欄位還在變動時最省事；需要篩選的欄位（status、level、rule_id）另外開欄位 |
| 完全正規化 | 查詢彈性大，但 schema 一變就要寫 migration |

目前先用前者。SQLite 的 JSON 函式是否可用要看 Microsoft.Data.Sqlite 內建的 SQLite 版本（待查證），先不要依賴。

## Step 7. 規則管理的 MCP 工具

在 `Mcp/RuleTools.cs` 加上 `create_clash_rule`、`list_clash_rules`、`delete_clash_rule`，寫法比照 Day 5 的工具註冊方式。`create_clash_rule` 要驗證：`id` 不重複、`categories` 是合法的 `BuiltInCategory` 名稱字串、`tolerance_mm >= 0`、`source` 格式正確。

這三個工具只動 SQLite，不需要 Revit，可以全部在 Mac 上用 Mock 模式測。

---

## 驗收

- [ ] 基準模型（ARC / STR / MEP）互相連結，其中一個連結放置兩次
- [ ] `fixtures/baseline/README.md` 與 `expected-clashes.json` 已建立，至少 10 筆標註，涵蓋四種類型
- [ ] `ClashRule`、`ClashRecord` 的欄位與 SPEC §5.2、§5.4 一致，JSON 序列化是 snake_case
- [ ] `ClashId` 的三個單元測試通過
- [ ] Sidecar 啟動時自動執行 migration，重複啟動不會重複執行
- [ ] 從 MCP client 呼叫 `create_clash_rule` → `list_clash_rules` → `delete_clash_rule` 成功

## 常見問題

**Q：沒有真實專案可以用怎麼辦？**
自己畫一個小模型就夠了。重點是**刻意**放入各種邊界情況，而不是模型多真實。真實專案留到 Week 3 做效能基準。

**Q：標註要做到多精確？**
至少記錄兩個元素的 UniqueId 與預期判讀。交集體積這類數字不用標，Day 14 會拿 Navisworks 結果互相對照。

**Q：Core 是 netstandard2.0，可以用 `System.Text.Json` 嗎？**
可以透過 NuGet 引用，但 add-in 端引用越多套件越容易和其他 add-in 衝突（見 [01-architecture.md](../01-architecture.md)）。Core 的模型類別盡量只用屬性，序列化的設定放在 Sidecar 與 Add-in 各自處理。

## 學習筆記

（寫下今天的心得、卡關點、想問的問題）
