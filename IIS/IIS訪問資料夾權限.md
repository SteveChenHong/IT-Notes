# 📂 資料夾權限設定：IIS 常見群組與使用者名稱

在 Windows 中，您可以透過以下步驟設定 IIS 站台資料夾的權限：

1. 右鍵點選資料夾，選擇「內容」。
2. 切換至「安全性」標籤頁，點選「編輯」按鈕。
3. 在「權限」視窗中，點選「新增」以加入使用者或群組。
4. 在「從這個位置」欄位中，選擇「本機電腦」，然後在「輸入物件名稱」欄位中輸入以下使用者或群組名稱，並點選「檢查名稱」以確認。

---

## 1. IIS App Pool\站台的應用程式集區（權限範圍最小）

- **說明**：此帳戶對應於特定應用程式池，預設為 `ApplicationPoolIdentity`，例如 `IIS AppPool\MyAppPool`。
- **用途**：僅授予應用程式所需的最低權限，建議僅給予「讀取」與「執行」權限，以確保應用程式能正常運行，並減少安全風險。
- **注意事項**：若在「輸入物件名稱」中找不到此帳戶，請確保「從這個位置」欄位選擇的是「本機電腦」，而非網域名稱。([stackoverflow.com](https://stackoverflow.com/questions/7334216/iis7-permissions-overview-applicationpoolidentity?utm_source=chatgpt.com))

---

## 2. IIS_IUSRS（權限範圍中等）

- **說明**：`IIS_IUSRS` 是一個內建的 Windows 群組，包含所有應用程式池的身份。
- **用途**：通常用於對 IIS 所有 Web 站點的安全性設置，而不僅僅是特定應用程式池。
- **權限建議**：僅賦予「讀取」權限，以確保 IIS 可以提供網頁和資源。
- **注意事項**：不建議為 `IIS_IUSRS` 群組賦予寫入權限，除非有特定需求，因為這可能會增加安全風險。

---

## 3. Authenticated Users（權限範圍最大）

- **說明**：此群組包含所有已通過身份驗證的使用者帳戶。
- **用途**：表示 Windows 系統中所有使用者名稱、密碼登錄並通過身份驗證的帳戶。
- **權限建議**：僅在需要所有經過身份驗證的使用者都能存取資料夾時，才賦予適當的權限。
- **注意事項**：賦予此群組過多權限可能會增加安全風險，建議僅在必要時使用。

---

## ✅ 設定建議總結

- **最小權限原則**：僅為應用程式池身份（`IIS AppPool\MyAppPool`）賦予必要的權限，避免過度授權。
- **避免過度授權**：避免為 `IIS_IUSRS` 或 `Authenticated Users` 群組賦予寫入權限，除非有特定需求。
- **定期檢查權限**：定期檢查資料夾的權限設定，確保符合安全性最佳實踐。([stackoverflow.com](https://stackoverflow.com/questions/7334216/iis7-permissions-overview-applicationpoolidentity?utm_source=chatgpt.com))

---

## 📝 附註

- **IIS App Pool Identity**：自 Windows Server 2008 以來，IIS 預設使用應用程式池身份（`ApplicationPoolIdentity`）來運行應用程式池。([learn.microsoft.com](https://learn.microsoft.com/en-us/iis/manage/configuring-security/application-pool-identities?utm_source=chatgpt.com))
- **權限設定工具**：可以使用 `icacls` 命令行工具來設定資料夾的權限，例如：
  
  ```powershell
  icacls "C:\inetpub\wwwroot" /grant "IIS AppPool\MyAppPool:(OI)(CI)(RX)"
