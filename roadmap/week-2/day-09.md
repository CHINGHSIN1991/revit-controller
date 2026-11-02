# Day 9 · 元素收集與幾何擷取

## 今日目標

給定一條 `ClashRule`，在 add-in 中收集 A 組與 B 組元素（依類別、參數篩選、主模型或連結模型），並擷取每個元素的 solid。建立幾何快取，元素被修改時讓快取失效。

今天的程式都在 add-in 內，要在 Windows 上用 Revit 驗證。連結模型的座標轉換留到 Day 10，今天只處理「從哪個 `Document` 收集」。

## 你會學到

- `FilteredElementCollector` 搭配多類別與參數篩選
- `Element.get_Geometry(Options)` 與 `Options` 各欄位的影響
- 幾何樹的結構：`GeometryElement` → `Solid` / `GeometryInstance` → `GetInstanceGeometry()`
- 為什麼要排除空 solid（`Volume == 0`）
- 用 `DocumentChanged` 事件讓快取失效
- mm 與 Revit 內部單位（英尺）的換算放在哪裡

---

## Step 1. 單位換算放一個地方

SPEC §5.2：對外一律 mm，add-in 內部轉成英尺。換算只寫一次：

```csharp
// src/RevitController.Core/Units/Mm.cs（Core 不引用 RevitAPI，只用常數）
public static class Mm
{
    public const double PerFoot = 304.8;
    public static double ToFeet(double mm) => mm / PerFoot;
    public static double FromFeet(double ft) => ft * PerFoot;
    public static double Mm3FromFeet3(double ft3) => ft3 * PerFoot * PerFoot * PerFoot;
}
```

Revit 也有 `UnitUtils.ConvertToInternalUnits(value, UnitTypeId.Millimeters)`，但單位 API 在不同版本間有變動（`DisplayUnitType` → `ForgeTypeId`，見 Day 1 查證結果，待查證）。長度與體積的換算係數固定，在 Core 用常數就好，可以在 Mac 上測。

## Step 2. 決定要從哪些 `Document` 收集

`ClashElementSet.Source` 有三種值：

| `source` | 收集對象 |
| --- | --- |
| `host` | 主模型 `doc` |
| `link:<name>` | 名稱符合的連結模型的 `GetLinkDocument()`，每個 instance 各一份 |
| `any` | 主模型加上所有已載入的連結 |

先定義一個中介型別，表示「這批元素來自哪裡」。Transform 今天先用 `Transform.Identity`，Day 10 再補上：

```csharp
// src/RevitController.Addin/Clash/ElementCollector.cs
internal sealed class SourceScope
{
    public Document Doc { get; set; }              // 主模型或連結文件
    public RevitLinkInstance? LinkInstance { get; set; }   // 主模型時為 null
    public Transform ToHost { get; set; } = Transform.Identity;
}
```

`<name>` 要對應到什麼（連結的檔名、`RevitLinkType` 名稱，或使用者自訂的別名，例如 `STR`）先決定一種並寫進 SPEC。建議用 `RevitLinkType` 名稱去掉副檔名（待決定）。

## Step 3. 依類別與參數收集

```csharp
internal static IList<Element> Collect(SourceScope scope, ClashElementSet set)
{
    var cats = set.Categories
        .Select(c => (BuiltInCategory)Enum.Parse(typeof(BuiltInCategory), c))
        .ToList();

    var collector = new FilteredElementCollector(scope.Doc)
        .WhereElementIsNotElementType()
        .WherePasses(new ElementMulticategoryFilter(cats));

    if (set.Filter != null)
        collector = collector.WherePasses(BuildParameterFilter(scope.Doc, set.Filter));

    return collector.ToElements();
}
```

| 概念 | 說明 |
| --- | --- |
| Quick filter | 只看元素的基本資料，不用展開元素，例如類別篩選。先放 |
| Slow filter | 要展開元素才能判斷，例如參數篩選。後放 |
| `WhereElementIsNotElementType()` | 排除族群類型，只留實際放置的元素 |
| `ElementMulticategoryFilter` | 一次篩多個類別，比 `OfCategory` 串多次方便 |

參數篩選用 `ParameterFilterRuleFactory` 建立 `FilterRule`，再包成 `ElementParameterFilter`：

```csharp
static ElementFilter BuildParameterFilter(Document doc, ParameterFilterSpec spec)
{
    // 1. 由參數名稱找到 ElementId（BuiltInParameter 或共用參數）
    // 2. var rule = ParameterFilterRuleFactory.CreateEqualsRule(paramId, value ...);
    //    字串版的多載在不同版本有差異（是否有 caseSensitive 參數）（待查證）
    // 3. return new ElementParameterFilter(rule);
    throw new NotImplementedException();
}
```

`ParameterFilterSpec` 的 JSON 格式先只支援「參數名稱 + 等於 / 包含 + 值」，夠用再擴充。

## Step 4. 擷取 solid

```csharp
// src/RevitController.Addin/Clash/GeometryExtractor.cs
static readonly Options Opt = new Options
{
    DetailLevel = ViewDetailLevel.Fine,
    IncludeNonVisibleObjects = false,
    ComputeReferences = false,
};

internal static List<Solid> GetSolids(Element e)
{
    var result = new List<Solid>();
    Walk(e.get_Geometry(Opt), result);
    return result;
}

static void Walk(GeometryElement? geo, List<Solid> result)
{
    if (geo == null) return;
    foreach (GeometryObject obj in geo)
    {
        if (obj is Solid s && s.Volume > 0) result.Add(s);           // 空 solid 排除
        else if (obj is GeometryInstance gi) Walk(gi.GetInstanceGeometry(), result);
    }
}
```

| 項目 | 影響 |
| --- | --- |
| `DetailLevel` | 族群在不同詳細等級可能有不同幾何。碰撞用 `Fine`；Navisworks 匯出時的設定可能不同，Day 14 比對時要記得 |
| `IncludeNonVisibleObjects` | `true` 會包含隱藏的輔助幾何，碰撞時通常不要 |
| `GetInstanceGeometry()` | 回傳已套用 instance transform 的幾何（對應的 `GetSymbolGeometry()` 則是族群本身座標）。兩者差異請查 API 文件確認（待查證） |
| `Volume == 0` | 線、面、空 solid 也會出現在幾何樹中，不排除會造成後面的 boolean 運算失敗 |

不需要 `View` 的 `Options` 代表取得的是模型幾何，不受視圖可見性影響。

## Step 5. 幾何快取與失效

同一個元素在多條規則中會被重複使用，幾何擷取很花時間，所以要快取：

```csharp
// src/RevitController.Addin/Clash/GeometryCache.cs
internal sealed class GeometryCache
{
    // key：(文件識別, ElementId)。文件識別可以用 Document.PathName 或 Title（待決定）
    readonly Dictionary<(string doc, long id), List<Solid>> _map = new();

    public void OnDocumentChanged(object sender, DocumentChangedEventArgs e)
    {
        // e.GetModifiedElementIds()、e.GetDeletedElementIds() → 從 _map 移除
        // 被連結的模型重新載入時，整份連結的快取都要清掉（如何偵測待查證）
    }
}
```

在 `App.OnStartup` 註冊：`application.ControlledApplication.DocumentChanged += cache.OnDocumentChanged;`，在 `OnShutdown` 取消註冊。

注意：

- `ElementId` 的整數值在 2024 以前是 `IntegerValue`，之後是 `Value`（64 位元，待查證版本），用 Day 2 建好的 adapter 處理，不要在這裡寫 `#if`。
- 連結文件的修改應該不會觸發主模型的 `DocumentChanged`，連結要重新載入後才更新；重新載入時主模型會收到哪些事件（待查證）。
- 快取只在 UI thread 上讀寫，不需要鎖。

## Step 6. 臨時的除錯方法

在 Day 11 的 job 完成前，先加一個 RPC 方法 `clash.debugCollect(rule)`，回傳 A / B 組的元素數量與 solid 數量。從 Sidecar 呼叫，和 Revit 畫面上的數量比對。這個方法驗收後可以留著當診斷工具，但不要開放成 MCP 工具。

---

## 驗收

- [ ] `Mm` 換算有單元測試（Mac 上跑）
- [ ] 對基準模型的一條規則，`clash.debugCollect` 回傳的元素數量與 Revit 中手動篩選的數量一致
- [ ] 族群（例如風管配件、梁）能取到 solid；`Volume == 0` 的物件被排除
- [ ] 參數篩選至少支援一種條件，並有測試案例
- [ ] 修改一個元素後，該元素的快取被清掉（用 log 確認）
- [ ] 兩個 Revit 版本都 build 成功

## 常見問題

**Q：某些元素 `get_Geometry` 回傳 null 或沒有 solid？**
可能是沒有 3D 幾何的元素（例如某些註解類），或幾何只在特定詳細等級出現。記錄這類元素的類別與數量，回傳給使用者，不要默默丟掉。

**Q：牆、樓板也有 `GeometryInstance` 嗎？**
系統族群通常直接回傳 `Solid`；可載入族群（風管配件、家具、結構框架等）才常見 `GeometryInstance`。兩種情況都要處理。

**Q：幾何能不能在背景執行緒處理？**
`Solid` 是 Revit API 物件，只能在 API context 中使用。SPEC §5.3 提到之後可以在 UI thread 上轉成 mesh，再交給背景執行緒計算，本週不做。

## 學習筆記

（寫下今天的心得、卡關點、想問的問題）
