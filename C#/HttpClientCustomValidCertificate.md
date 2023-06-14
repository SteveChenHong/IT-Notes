在 .NET 中使用 HttpClient 進行 HTTP 請求時，預設情況下，它會驗證對方的 HTTPS 憑證是否合法。這稱為 SSL/TLS 驗證。

HttpClient 使用 .NET 的 SSL/TLS 庫來進行憑證驗證，並根據憑證的有效性和信任狀態來判斷連線是否安全。如果對方的憑證是由信任的憑證授權中心（CA）簽署的，且未過期且與要請求的主機名稱相符，則 HttpClient 會視其為合法憑證。

然而，有時您可能需要更細緻的控制和自定義驗證邏輯。在這種情況下，您可以使用 HttpClientHandler 類別來設定憑證驗證選項。以下是一個示例：  
```
using System;
using System.Net.Http;

class Program
{
    static void Main()
    {
        // 建立 HttpClientHandler 實例
        HttpClientHandler handler = new HttpClientHandler();

        // 設定憑證驗證選項
        handler.ServerCertificateCustomValidationCallback = (sender, cert, chain, sslPolicyErrors) =>
        {
            // 自定義驗證邏輯
            // 在此處您可以檢查 cert 和 chain 的相關資訊，並決定是否接受或拒絕憑證

            // 回傳 true 表示接受憑證，回傳 false 表示拒絕憑證
            return true;
        };

        // 建立 HttpClient 並指定 HttpClientHandler
        HttpClient client = new HttpClient(handler);

        // 發送 HTTP 請求
        HttpResponseMessage response = client.GetAsync("https://example.com").Result;

        // 處理回應
        Console.WriteLine(response.StatusCode);

        // 確保正確關閉 HttpClient
        client.Dispose();
    }
}

```

在 .NET 6 中，HttpClientFactory 取代了之前版本中的 HttpClient，成為首選的 HTTP 請求管理方式。HttpClientFactory 是一個用於建立和管理 HttpClient 實例的工廠類別，提供更好的效能和資源管理。

在使用 HttpClientFactory 時，預設情況下，它會自動驗證對方的 HTTPS 憑證是否合法，使用與前面提到的 HttpClient 相同的憑證驗證機制。

以下是使用 .NET 6 中的 HttpClientFactory 的示例：

```
using System;
using System.Net.Http;
using System.Net.Security;
using System.Security.Cryptography.X509Certificates;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Http;

class Program
{
    static void Main()
    {
        // 建立服務集合
        var services = new ServiceCollection();

        // 設定 HttpClientFactory
        services.AddHttpClient("custom", httpClient =>
        {
            httpClient.ConfigurePrimaryHttpMessageHandler(() =>
            {
                // 建立自定義的 HttpMessageHandler
                var handler = new HttpClientHandler();
                
                // 設定憑證驗證回呼函式
                handler.ServerCertificateCustomValidationCallback = ValidateCertificate;
                
                return handler;
            });
        });

        // 建立服務提供者
        var serviceProvider = services.BuildServiceProvider();

        // 從服務提供者中取得 HttpClientFactory
        var httpClientFactory = serviceProvider.GetRequiredService<IHttpClientFactory>();

        // 使用自定義的 HttpClientFactory 建立 HttpClient
        var client = httpClientFactory.CreateClient("custom");

        // 發送 HTTP 請求
        var response = client.GetAsync("https://example.com").Result;

        // 處理回應
        Console.WriteLine(response.StatusCode);
    }

    static bool ValidateCertificate(HttpRequestMessage request, X509Certificate2 certificate, X509Chain chain, SslPolicyErrors errors)
    {
        // 自定義憑證驗證邏輯
        // 在此處您可以檢查憑證的相關資訊，並根據需求返回驗證結果

        // 回傳 true 表示接受憑證，回傳 false 表示拒絕憑證
        return true;
    }
}
```
