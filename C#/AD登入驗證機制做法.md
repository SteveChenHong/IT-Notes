# LADP 和 AD 的關係
LDAP（Lightweight Directory Access Protocol）是一種用於訪問和維護分佈式目錄服務的協議。AD（Active Directory）是一個由微軟開發和推出的目錄服務，它使用LDAP作為其協議之一。

簡單來說，LDAP是一個通用協議，可以用於與各種不同的目錄服務進行通訊，而AD是一個具體的目錄服務，它實現了LDAP協議以及其他協議，例如Kerberos和DNS。因此，可以通過LDAP協議訪問和操作AD目錄服務。LDAP也可以用於與其他目錄服務進行通訊，例如OpenLDAP和Novell eDirectory等。

總之，LDAP是一個協議，而AD是一個實現了LDAP協議的目錄服務。

要驗證AD中的用戶帳戶和密碼，可以使用C#中的System.DirectoryServices命名空間中的類。以下是一個簡單的代碼示例：

````
using System.DirectoryServices;

public bool IsAuthenticated(string username, string password)
{
    try
    {
        string domain = "yourdomain.com"; // 修改為你的域名
        string path = "LDAP://" + domain;
        using(DirectoryEntry entry = new DirectoryEntry(path, username, password)){
            DirectorySearcher searcher = new DirectorySearcher(entry);
            SearchResult result = searcher.FindOne();
            return true;
        };
    }
    catch (Exception ex)
    {
        return false;
    }
}
````

IsAuthenticated()為true後，繼續做登入驗證機制，如表單驗證FormsAuthentication、FormsAuthenticationTicket等，做成完整登入機制。
