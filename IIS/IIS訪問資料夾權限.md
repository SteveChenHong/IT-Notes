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

### 🔎 延伸補充（來自 IIS 7.5 文章）
- `IIS_IUSRS` 是 **IIS 7.5 的預設程式集區群組**，其中包含 `DefaultAppPool`。  
- 該群組成員的 **ApplicationPoolIdentity** 是預設的標識。  
- `ApplicationPoolIdentity` 並不是單一帳戶，而是所有應用程式集區的預設身分總稱，且與各自的應用程式集區一一對應，例如：  
  - `DefaultAppPool` → `IIS AppPool\DefaultAppPool`  
- 有了 `IIS_IUSRS` 群組，管理應用程式集區標識就更簡單，不需要針對不同的應用程式池分別設定額外帳戶權限。  

---

## 3. IUSR（匿名帳戶，權限範圍小到中）

- **說明**：`IUSR` 是 **IIS 7.5 的匿名帳戶**，取代了早期的 `IUSR_MachineName`。  
- **用途**：網站若啟用「匿名驗證」，訪客瀏覽網站時 IIS 會以 `IUSR` 身份存取資源，不需輸入帳號密碼。  
- **權限範圍**：通常僅需「讀取」與「執行」權限，用於提供匿名訪客存取靜態檔案（HTML、圖片、CSS、JS 等）。  
- **建議做法**：  
  - 若網站只用 Application Pool Identity（`IIS AppPool\MyAppPool`）執行程式，通常不需要額外賦予 `IUSR` 權限。  
  - 只有在啟用匿名驗證、且訪客需要直接存取資料夾檔案時，才需要授權。  
  - 建議僅限「讀取 / 執行」，避免給「寫入」權限，否則匿名訪客可能修改或上傳檔案，存在安全風險。  

---

## 4. Authenticated Users（權限範圍最大）

- **說明**：此群組包含所有已通過身份驗證的使用者帳戶。  
- **用途**：表示 Windows 系統中所有使用者名稱、密碼登錄並通過身份驗證的帳戶。  
- **權限建議**：僅在需要所有經過身份驗證的使用者都能存取資料夾時，才賦予適當的權限。  
- **注意事項**：賦予此群組過多權限可能會增加安全風險，建議僅在必要時使用。  

---

## ✅ 設定建議總結

- **最小權限原則**：僅為應用程式池身份（`IIS AppPool\MyAppPool`）賦予必要的權限，避免過度授權。  
- **避免過度授權**：避免為 `IIS_IUSRS`、`IUSR` 或 `Authenticated Users` 群組賦予寫入權限，除非有特定需求。  
- **定期檢查權限**：定期檢查資料夾的權限設定，確保符合安全性最佳實踐。([stackoverflow.com](https://stackoverflow.com/questions/7334216/iis7-permissions-overview-applicationpoolidentity?utm_source=chatgpt.com))

---

## 📝 附註

- **IIS App Pool Identity**：自 Windows Server 2008 以來，IIS 預設使用應用程式池身份（`ApplicationPoolIdentity`）來運行應用程式池。([learn.microsoft.com](https://learn.microsoft.com/en-us/iis/manage/configuring-security/application-pool-identities?utm_source=chatgpt.com))  
- **權限設定工具**：可以使用 `icacls` 命令行工具來設定資料夾的權限，例如：  

  ```powershell
  icacls "C:\inetpub\wwwroot" /grant "IIS AppPool\MyAppPool:(OI)(CI)(RX)"

---
## 比較表

| 帳戶 / 群組名稱                 | 用途核心                                                         | 是否需密碼       | 權限範圍 (精準)                                          | 建議授權 / 核心重點                                            |
| ------------------------- | ------------------------------------------------------------ | ----------- | -------------------------------------------------- | ------------------------------------------------------ |
| **IIS AppPool\AppPool名稱** | 應用程式集區專用虛擬帳戶，用於運行特定網站或應用程式。                                  | 否（自動生成虛擬帳戶） | 僅限該 **AppPool** 對應的網站與其配置，安全性最小化（最少權限原則）。          | 🔑 讀取 / 執行權限。建議僅對需要的目錄或資源授權，避免授權過廣。                    |
| **IIS_IUSRS**             | IIS 內建群組，包含所有應用程式集區虛擬帳戶 (e.g. `IIS APPPOOL\DefaultAppPool`)。 | 否（群組本身無密碼）  | 中等範圍。用於 **多個應用程式集區共用資源**（如上傳目錄、共用設定檔）。             | 建議只給 **讀取** 權限，避免成為過度授權的通道。                            |
| **IUSR**                  | IIS 內建匿名存取帳戶，用於處理 **匿名訪客請求**（例如：靜態 HTML、圖片）。                 | 否（內建帳戶，無密碼） | 僅限匿名訪客能存取的公開內容目錄。                                  | 建議僅授予 **讀取** 權限，避免寫入或執行權限。若不需匿名訪客，建議停用。                |
| **Authenticated Users**   | 系統中所有已驗證（登入 Windows）的使用者帳戶集合。                                | 是（使用個人帳戶密碼） | 權限極廣，包含 **所有已登入使用者**。對 IIS 來說，通常等同於 Windows 使用者群體。 | ⚠️ 很少用於 IIS 授權，因為範圍太大。僅在 **需要依 Windows 登入控制資源存取** 時使用。 |


