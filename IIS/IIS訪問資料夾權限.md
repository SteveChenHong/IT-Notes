在 IIS 要訪問的資料夾右鍵 > 內容 > 安全性 > 編輯 > 新增群組或使用者名稱，常見如下三種群組

- IIS App Pool\站台的應用程式集區 (權限範圍最小)：

這是與特定應用程式池關聯的用戶或用戶群體。
此帳戶用於運行您的 Web 應用程序，通常對應用程序的應用程式池有特定的權限。
通常，您會將此帳戶授予對應用程序所需的最低權限，以確保應用程序能夠正常運行，但不會擁有不必要的權限。
這通常包括對應用程序所需的文件和資料夾的讀取和執行權限。

 - IIS_IUSRS (權限範圍中等)：

IIS_IUSRS（Internet Information Services（IIS）用戶組）是一個預定義的 Windows 用戶組。
此用戶組包含了可以訪問 IIS Web 站點內容的用戶。
通常，IIS_IUSRS 用於對 IIS 所有 Web 站點的安全性設置，而不僅僅是特定應用程序池。
IIS_IUSRS 用戶組通常被賦予對 IIS Web 站點內容的讀取權限，以便 IIS 可以提供網頁和資源。  

- Authenticated Users (權限範圍最大)：

表示 Windows 系統中所有使用用户名、密碼登錄並通過身份驗證的帳戶。

- 總結來說，差異在於範圍和用途：

IIS App Pool\站台的應用程式集區是特定於應用程式池的，它用於運行您的 Web 應用程序，並具有較緊密的關聯性。您通常需要將其配置以滿足特定應用程序的需求。

IIS_IUSRS 是一個更廣泛的用戶組，它用於訪問整個 IIS 伺服器上的 Web 站點內容，而不僅僅是單個應用程序池。它通常僅被賦予讀取權限，以確保 IIS 可以提供內容。

Authenticated Users 又比 IIS_IUSRS 更廣泛。

安全性設置的具體需求應根據您的應用程序和環境來確定。通常，應盡量限制權限，以減少潛在的安全風險，並確保應用程序可以正常運行。
