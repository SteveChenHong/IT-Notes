文章一：  

Nginx 伺服器憑證的設計可以分為三種主要情況

- 不需要使用憑證：這種情況下，伺服器不需要憑證，通信以普通的 HTTP 協議進行。這種情況下的通信是不加密的，缺乏安全性。

- 單向驗證，只有伺服器有憑證：這種情況下，伺服器配置了憑證，用戶端需要驗證伺服器的憑證。通信使用 HTTPS 協議，但只有伺服器被驗證。

- 雙向驗證，伺服器和客戶端都有憑證：這種情況下，伺服器配置了憑證，並且客戶端也需要提供自己的憑證。通信使用 HTTPS 協議，雙方都進行了驗證，提供了更高的安全性。

根據您的需求和安全要求，您可以從這三種情況中選擇適合的設計。通常情況下，為了確保通信的安全性，至少應該選擇第 2 種情況，即使用伺服器憑證實現單向驗證。如果需要更高的安全性和雙方互相驗證，可以選擇第 3 種情況，實現雙向驗證。

```
#單向憑證設定檔(只有Server)

server {
    listen 443 ssl http2;
    server_name your-domain.com;

    ssl_certificate /path/to/your_ssl_certificate.crt;  # SSL 憑證的路徑
    ssl_certificate_key /path/to/your_private_key.key;  # 私鑰的路徑

    # 可選：添加其他 SSL 相關設定
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers 'TLS_AES_128_GCM_SHA256:TLS_AES_256_GCM_SHA384:ECDHE-RSA-AES128-GCM-SHA256...';
    
    # 添加其他伺服器配置...
}
```


```
#雙向憑證設定檔(Server有且要求Client也要帶入憑證)

server {
    listen 443 ssl;
    server_name your-domain.com;

    ssl_certificate /path/to/your_ssl_certificate.crt;    # 伺服器 SSL 憑證的路徑
    ssl_certificate_key /path/to/your_private_key.key;    # 伺服器私鑰的路徑

    ssl_client_certificate /path/to/client_cert_bundle.crt;  # 客戶端憑證捆綁檔案的路徑
    ssl_verify_client on;  # 啟用客戶端驗證

    # 可選：添加其他 SSL 相關設定
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers 'TLS_AES_128_GCM_SHA256:TLS_AES_256_GCM_SHA384:ECDHE-RSA-AES128-GCM-SHA256...';
    
    # 添加其他伺服器配置...
}
```
