# LADP 和 AD 的關係
LDAP（Lightweight Directory Access Protocol）是一種用於訪問和維護分佈式目錄服務的協議。AD（Active Directory）是一個由微軟開發和推出的目錄服務，它使用LDAP作為其協議之一。

簡單來說，LDAP是一個通用協議，可以用於與各種不同的目錄服務進行通訊，而AD是一個具體的目錄服務，它實現了LDAP協議以及其他協議，例如Kerberos和DNS。因此，可以通過LDAP協議訪問和操作AD目錄服務。LDAP也可以用於與其他目錄服務進行通訊，例如OpenLDAP和Novell eDirectory等。

總之，LDAP是一個協議，而AD是一個實現了LDAP協議的目錄服務。

要驗證AD中的用戶帳戶和密碼，可以使用C#中的System.DirectoryServices命名空間中的類。以下是一個簡單的代碼示例：

````
using System.DirectoryServices;

// 用於驗證使用者認證的方法
// 參數 username: 要驗證的使用者名稱
// 參數 password: 要驗證的使用者密碼
public bool IsAuthenticated(string username, string password)
{
    try
    {
        string domain = "yourdomain.com"; // 修改為你的域名
        string path = "LDAP://" + domain;
        
        // 使用 DirectoryEntry 類別來建立目錄項目，使用提供的使用者名稱和密碼進行驗證
        using(DirectoryEntry entry = new DirectoryEntry(path, username, password)){
            DirectorySearcher searcher = new DirectorySearcher(entry);
            
            // 在目錄中搜尋使用者，如果找到結果，則代表驗證成功
            SearchResult result = searcher.FindOne();
            return true;
        };
    }
    catch (Exception ex)
    {
        // 如果出現異常，代表驗證失敗
        return false;
    }
}
````

IsAuthenticated()為true後，繼續做登入驗證機制，如表單驗證FormsAuthentication、FormsAuthenticationTicket等，做成完整登入機制。
