### windows netsata 指令(可看作net網路stat狀態)
可以查詢本機網路和外界網路連線的指令，可以透過這個指令的查詢得知有沒有奇怪的連線在你的機器中連線，  
也可透過此指令瞭解電腦連線的狀況。

### 1.命令提示字元輸入以下(或用PowerShell)

```C:\WINDOWS\system32>netstat -ano```
  
狀態說明：  
  
－LISTENING：偵聽（LISTENING）狀態，等待連接。

－ESTABLISHED：ESTABLISHED的意思是建立連接。表示兩臺機器正在通信。  

－CLOSE_WAIT：  
　對方主動關閉連接或者網絡異常導致連接中斷，這時我方的狀態會變成CLOSE_WAIT  
　此時我方要調用close()來使得連接正確關閉  
 
－TIME_WAIT：我方主動調用close()斷開連接，收到對方確認後狀態變爲TIME_WAIT。  

－SYN_SENT：連線初始時，發送封包的狀態。  

－SYN_RECV：接收到一個要求連線的主動連線封包。  

－FIN_WAIT1：該插槽服務(socket)已中斷，該連線正在斷線當中。  

－FIN_WAIT2：該連線已掛斷，但正在等待對方主機回應斷線確認的封包。  

顯示範例：  

使用中連線
|協定|本機位址|外部位址|狀態|PID|
|:-|:-|:-|:-|:-|
|TCP|192.168.0.100:50867|140.82.114.25:443|ESTABLISHED|10224|  

表示本機port:50867目前與207.46.125.33:443主機連線已建立完成，PID是10224  

「工作管理員」 -> 「詳細資料」頁籤中，可查詢處理程序PID

### 2.列出每個屬性說明
```C:\WINDOWS\system32>netstat /?```

顯示通訊協定統計資料與目前的 TCP/IP 網路連線  
-a 顯示所有連線和接聽的連接埠  
-b 顯示在建立各個連線或接聽連接埠時會用到的可執行檔  
-n 以數字格式顯示位址和連接埠號碼  
-o 顯示與各連線相關的擁有流程識別碼  
-p proto 顯示由 proto 指定之通訊協定的連線; proto  
　　  　&nbsp;&nbsp;可以是以下任一項: TCP、UDP、TCPv6 或 UDPv6。若搭配 -s  
　　　  &nbsp;&nbsp;選項使用來顯示各通訊協定的統計資料，proto 可以是以下任一項:  
　　　　IP、IPv6、ICMP、ICMPv6、TCP、TCPv6、UDP 或 UDPv6。  
.....

### 3.其它實用用法

想監控特定連線的來源或 Port，一些小技巧，用例子：  

```netstat -na | find "特定IP"```  
```netstat -na | find "特定Port"```  
```netstat -na | find "特定IP:特定Port"```  
像是：  
```netstat -na | find "192.168.0.100"```  
```netstat -na | find "5353"```  
```netstat -na | find "192.168.0.100:5353"```  

```netstat –na 1 | find "特定IP"```  
顯示特定 IP 之連線，每隔一秒更新畫面一次 (適用於像是你已鎖定可疑對象，但不知他何時會連過來)  
-a 代表列出所有連線  
-n 代表僅列出 IP 及 Port，不解析為 hostname 及 service name，速度會快很多  

```netstat –nao 1 | find "特定IP"```  
加上 -o 參數可顯示觸發該連線之 process ID，  
欲知 process name 則可以透過內建的 tasklist 這程式  

```netstat –na 1 | find "4444" | find "ESTABLISHED"```  
也可以針對特定 Port，不分對象的進行監控，  
再透過 find "ESTABLISHED" 篩選  

```netstat -a -p tcp | find "7218"```    
只列出IPv4的監聽口

```netstat -a | find /c "7218"```  
取得 7218 port 的筆數
