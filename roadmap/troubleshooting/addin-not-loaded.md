# 排錯 · Add-in 沒有被 Revit 載入

## 症狀

Revit 啟動後，Add-Ins 頁籤看不到你的按鈕，或 `IExternalApplication.OnStartup` 裡的 log 沒有出現。

## 檢查順序

### 1. Manifest 放對地方了嗎？

Revit 會掃描這兩個位置（`20XX` 是 Revit 版本）：

```
%AppData%\Autodesk\Revit\Addins\20XX\        ← 目前使用者
%ProgramData%\Autodesk\Revit\Addins\20XX\    ← 所有使用者
```

- 版本資料夾要對：Revit 2025 不會讀 `Addins\2024\`。
- 檔案副檔名要是 `.addin`，不是 `.addin.xml`（Windows 預設隱藏副檔名時容易出錯）。

### 2. Manifest 內容正確嗎？

```xml
<?xml version="1.0" encoding="utf-8"?>
<RevitAddIns>
  <AddIn Type="Application">
    <Name>RevitController</Name>
    <Assembly>RevitController.Addin.dll</Assembly>
    <AddInId>（一個固定的 GUID）</AddInId>
    <FullClassName>RevitController.Addin.App</FullClassName>
    <VendorId>（廠商代碼）</VendorId>
  </AddIn>
</RevitAddIns>
```

- `Assembly` 是相對於 manifest 的路徑，或絕對路徑。
- `FullClassName` 要包含 namespace，大小寫要一致。
- `AddInId` 每個 add-in 唯一，**不要每次 build 都換**。

### 3. DLL 的目標框架和 Revit 版本相符嗎？

- 把 .NET 8 build 出來的 DLL 放進 net48 世代的 Revit（或反過來），會載入失敗。
- 確認放進 `Addins\20XX\` 的是對應 configuration 的輸出。

### 4. Revit 有跳出安全提示嗎？

未簽章的 add-in 第一次載入時，Revit 會詢問是否載入。選了「不載入」之後，之後可能都不會再問。到 Revit 的選項或對應的設定檔中重設（位置待查證）。

### 5. 看 Journal 檔

Revit 每次啟動都會寫 journal 檔（位置待查證，通常在 `%LocalAppData%\Autodesk\Revit\Autodesk Revit 20XX\Journals\`）。搜尋你的 add-in 名稱或 `.addin`，通常可以看到載入失敗的原因，例如找不到 DLL 或相依的 DLL。

### 6. 相依的 DLL 找不到或版本衝突

- 你的 add-in 相依的 DLL 要和主 DLL 放在同一個資料夾。
- 如果其他 add-in 先載入了同名但不同版本的 DLL，你的 add-in 可能拿到錯的版本。這是我們讓 add-in 盡量少用第三方套件的原因（見 [01-architecture.md](../01-architecture.md)）。

## 學到的事

（實際遇到後在這裡記錄原因與解法）
