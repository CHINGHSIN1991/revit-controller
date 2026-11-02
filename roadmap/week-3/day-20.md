# Day 20 · BCF 連接器與外部工具驗證

## 今日目標

1. 把 Day 19 的轉換邏輯包成 `IClashConnector` 實作（`Id = "bcf21"`），輸出 `.bcfzip`，並在 `external_refs` 記錄 Topic GUID。
2. 在**至少兩個外部工具**中開啟匯出檔，確認相機位置與元素定位正確。
3. 實際查證各工具的 BCF 支援程度（版本、匯入或匯出），更新 SPEC §11。

各外部工具支援哪個 BCF 版本、能匯入還是只能匯出，**撰寫本文件時都還沒有查證**。今天就是查證的那一天，結果以實測為準。

## 你會學到

- 連接器模式的第二個實作：和 Excel 連接器比，多了 `Viewpoints` 能力
- ZIP 打包（`System.IO.Compression`）
- 外部工具如何用 IFC GUID 對應元素，以及為什麼它需要載入同一份 IFC
- Revit IFC 匯出設定對 GUID 與座標的影響
- 重複匯出時沿用 Topic GUID，讓外部工具視為更新

---

## Step 1. 實作 `Bcf21Connector`

```csharp
// Sidecar/Connectors/Bcf21/Bcf21Connector.cs（骨架）
public sealed class Bcf21Connector : IClashConnector
{
    public string Id => "bcf21";
    public string DisplayName => "BCF 2.1";
    public ConnectorCapabilities Capabilities =>
        ConnectorCapabilities.Export | ConnectorCapabilities.Snapshots | ConnectorCapabilities.Viewpoints;
        // Import 之後再加（SPEC §8.2）

    public async Task<ExportResult> ExportAsync(IReadOnlyList<ClashRecord> clashes, ExportOptions options, CancellationToken ct)
    {
        var topics = clashes.Select(c => _mapper.ToTopic(c, options)).ToList();  // 沿用或產生 Topic GUID
        await using var fs = File.Create(options.OutputPath);
        using var zip = new ZipArchive(fs, ZipArchiveMode.Create);
        WriteEntry(zip, "bcf.version", _writer.WriteVersion());
        foreach (var t in topics)
        {
            ct.ThrowIfCancellationRequested();
            WriteEntry(zip, $"{t.Guid}/markup.bcf", _writer.WriteMarkup(t));
            // viewpoint.bcfv、snapshot.png
        }
        return new ExportResult(/* 每筆 clash_id → Topic GUID，供寫回 external_refs */);
    }

    public Task<ImportResult> ImportAsync(ImportSource s, CancellationToken ct) => throw new NotSupportedException();
    public Task<SyncResult> SyncStatusAsync(IReadOnlyList<ClashRecord> c, CancellationToken ct) => throw new NotSupportedException();
}
```

- ZIP 內的路徑分隔符要用 `/`，不要用 `Path.Combine`（Windows 上會變成 `\`）。
- 匯出成功後，在**同一個 SQLite transaction** 裡把 Topic GUID 寫進 `external_refs["bcf21"]`。匯出失敗就不寫。
- 依 SPEC §8.1，連接器只讀 `ClashRecord`，不修改核心資料模型；`external_refs` 由 `export_clashes` 的呼叫端寫回。
- 在 sidecar 註冊後，`list_connectors` 應該列出 `bcf21` 與它的能力。

## Step 2. 截圖還沒產生的碰撞

`snapshot_path` 為 null 或已過期（Day 18）時：

| 選擇 | 說明 |
| --- | --- |
| 匯出前先呼叫 `show_clash` 產生 | 截圖最完整，但要走 Revit，匯出變成長時間 job |
| 沒有截圖就不附 | 匯出快，外部工具仍有相機可以定位 |

建議：預設不附，`options.generate_missing_snapshots = true` 時改成 job 執行。

## Step 3. 準備驗證環境

外部工具要能「定位元素」，必須載入**與 BCF 的 IFC GUID 相同來源**的模型：

1. 從基準模型的每個專業模型（主模型與各連結）分別匯出 IFC。記下 Revit IFC 匯出的設定：座標基準、是否「儲存 IFC GUID 到參數」等（選項名稱待查證）。
2. 匯出 BCF 時，`ExportOptions` 的座標基準要和第 1 點一致（Day 19 Step 5）。
3. 挑 5 筆有代表性的碰撞：主模型對連結、連結對連結、hard、clearance、duplicate 各至少一筆。
4. 在 Revit 中記下這 5 筆碰撞兩個元素的 IFC GUID（可從 IFC 檔或參數中查），先和 `ClashRecord.ifc_guid` 比對一次。**這是 SPEC §11「IFC GUID 計算方式」的第一次實測。**

## Step 4. 在外部工具中開啟

候選工具（ROADMAP 列出的例子，實際能取得哪些依授權而定）：BIMcollab Zoom、Solibri、Navisworks（可能需要外掛），以及 Trimble Connect、Revit 本身的 BCF 外掛等。

每個工具的操作順序：

1. 開啟第 3 步匯出的 IFC。
2. 匯入 `.bcfzip`。
3. 對 5 筆碰撞逐一檢查。

驗證表（每個工具一份，填實測結果，不要先填預期值）：

| 檢查項目 | 工具 1：_____ | 工具 2：_____ |
| --- | --- | --- |
| 工具版本 | | |
| 能匯入 BCF 2.1？ | | |
| 能匯出 BCF？哪個版本？ | | |
| 支援 BCF 3.0？ | | |
| Topic 清單、標題、狀態正確 | | |
| 截圖顯示正常 | | |
| 相機位置正確（看向碰撞點） | | |
| 兩個元素都被選取 | | |
| 連結模型的元素能被選取 | | |
| Clipping planes（section box）有效 | | |
| 中文標題與留言正常 | | |
| 重新匯入（Step 5）時視為更新 | | |
| 備註 | | |

相機或選取不正確時的排查順序：

| 症狀 | 先檢查 |
| --- | --- |
| 相機偏移一個固定距離 | 座標基準（Day 19 Step 5） |
| 相機偏移且角度不對 | 正北角旋轉 |
| 相機位置差 1000 倍或 0.3 倍左右 | 單位（mm / m / ft） |
| 相機正確但沒有選到元素 | IFC GUID 不一致、或對方沒有載入對應的 IFC |
| 工具拒絕開啟檔案 | 先用 XSD 驗證（Day 19），再檢查 ZIP 結構與 `bcf.version` |

## Step 5. 重複匯出：沿用 Topic GUID

1. 第一次匯出，在外部工具中匯入。
2. 在本平台更新其中一筆的狀態與留言（`update_clash`）。
3. 再次匯出。因為 `external_refs["bcf21"]` 已有值，Topic GUID 相同。
4. 在外部工具中再匯入，觀察它是**更新原 Topic**還是**新增一筆**。

不同工具的行為可能不同（待查證），把結果填進 Step 4 的表格。若工具會建立重複的 Topic，在文件中提示使用者「每次匯出為新檔，匯入前先清除舊的」。

## Step 6. 更新 SPEC §11

把今天的實測結果寫回 [SPEC.md](../../SPEC.md)：

- §11「各工具對 BCF 的支援程度」：每個測過的工具一行，寫版本、方向、限制。尤其記下 Navisworks 的結果，它決定 Day 27 的互通方式。
- §11「Revit IFC GUID 的計算方式」：第 3 步的比對結果（一致、不一致、在哪些情況不一致）。
- 若 BCF 的座標或單位與 Day 19 的假設不同，回頭修正 Day 19 文件中標示「待查證」的地方。

驗證時的截圖與工具版本存進 `docs/verification/bcf21/`，方便之後換工具版本時重測。

---

## 驗收

- [ ] `list_connectors` 列出 `bcf21`，能力為 Export、Snapshots、Viewpoints
- [ ] `export_clashes("bcf21", …)` 產出 `.bcfzip`，`external_refs["bcf21"]` 有寫入 Topic GUID
- [ ] 至少兩個外部工具能開啟，5 筆碰撞的相機與元素定位都正確（或差異已找到原因並記錄）
- [ ] Step 4 的驗證表已填寫
- [ ] 重複匯出時 Topic GUID 不變；各工具的處理方式已記錄
- [ ] SPEC §11 中 BCF 支援程度與 IFC GUID 兩項已更新

## 常見問題

**Q：手邊只有一個工具可以測？**
M2 驗收要求至少兩個。可以找有免費版或試用版的工具（哪些工具提供免費版待查證），或請合作的顧問公司協助開啟。

**Q：連結模型的元素選不到？**
外部工具要同時載入那個連結模型匯出的 IFC。另外確認 `ifc_guid` 是用連結文件中的元素計算的，不是主模型。

**Q：Navisworks 能直接讀 BCF 嗎？**
不確定，這正是 SPEC §11 的待驗證事項。實測結果決定 Day 27 要走 BCF、XML 交換，還是寫 Navisworks 外掛。

## 學習筆記

（寫下今天的心得、卡關點、想問的問題）
