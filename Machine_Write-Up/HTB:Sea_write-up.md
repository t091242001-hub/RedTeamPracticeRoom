## 1. 執行摘要 (Executive Summary)

### 測試範圍與目標：
1. 靶機名稱:Sea
2. 難度:Easy
3. IP:10.129.4.85

### 主要發現：
1. 漏洞 ：跨站腳本XSS(CVE-2023-41425)（風險等級：Medium）
2. 漏洞 ：命令注入Command Injection（風險等級：Medium）

### 攻擊鏈摘要：
根據目錄爆破和原始碼搜索的結果，了解WonderCMS中存在CVE-2023-41425漏洞，這是一個跨站腳本（XSS）漏洞，攻擊者可以利用該漏洞上傳惡意模組，從而獲得系統存取權限。利用系統存取權限，取得未妥善保管的使用者憑證，可用於橫向移動，並發現內部開放端口8080，可被SSH穿隧。穿隧後暴露出以root權限執行的控制面板，可操縱POST請求造成命令注入。

### 潛在影響：
1. 公開漏洞CVE-2023-41425會造成外部攻擊者獲得系統存取權限，可能導致重要資料外洩。
2. 使用者憑證未被妥善保管，若外洩可能造成敏感資料外洩及權限提升。
3. 8080埠控制面板未過濾特殊符號，造成以root權限執行的命令注入漏洞，可能導致系統完全失控、資料大規模洩漏。

### 修復建議：
1. 立即更新WonderCMS版本至最新版。
2. 使用者憑證資料確實設立讀寫權限，禁止低權限身分隨意取得。
3. 8080埠控制面板需過慮敏感字符，並停止以root權限執行。

---

## 2. 技術發現與攻擊路徑詳述 (Findings)

### 2.1 資訊蒐集(Reconnaissance)

使用Nmap掃描開放埠，開放埠僅有22和80，判斷系統為面向使用者的web系統，第一優先攻擊面為80。
```bash
sudo nmap -sT -Pn --min-rate 5000 -p- 10.129.4.85
sudo nmap -sT -Pn -sV -sC -O -p22,80 10.129.4.85
sudo nmap --script=vuln -p22,80 10.129.4.85
```

<img width="809" height="190" alt="螢幕擷取畫面 2026-02-13 164735" src="https://github.com/user-attachments/assets/0ac81bcb-cfd0-4648-80a0-1b914bca7c54" />
<img width="870" height="411" alt="螢幕擷取畫面 2026-02-13 165020" src="https://github.com/user-attachments/assets/be3ff43d-3ea7-4431-95e1-5d4740983c60" />
<img width="614" height="162" alt="螢幕擷取畫面 2026-02-13 171022" src="https://github.com/user-attachments/assets/3fc17f4c-0723-45d0-abdd-e8dfe54ef091" />

### 2.2 枚舉(Enumeration)

使用gobuster進行目錄爆破。
```bash
sudo gobuster dir -u http://10.129.4.85 -w /user/share/dirbuster/wordlists/directory-list-2.3-medium.txt -x php,txt
```
<img width="1236" height="605" alt="螢幕擷取畫面 2026-02-13 171724" src="https://github.com/user-attachments/assets/7f7ebc8a-db62-46ac-81c5-5d94c59e9d4b" />

探索Web頁面。

<img width="1860" height="878" alt="螢幕擷取畫面 2026-02-13 164832" src="https://github.com/user-attachments/assets/899dcd27-edbd-4d74-bb36-55391f6bd047" />
<img width="1374" height="791" alt="螢幕擷取畫面 2026-02-13 165423" src="https://github.com/user-attachments/assets/366c2809-983f-4372-871e-b5da83484d33" />

探索原始碼。

<img width="1096" height="393" alt="螢幕擷取畫面 2026-02-13 171455" src="https://github.com/user-attachments/assets/5ff18b55-d4fa-4ce8-8080-3a91f1279a2f" />
<img width="1674" height="76" alt="螢幕擷取畫面 2026-02-13 171644" src="https://github.com/user-attachments/assets/43127e6c-00d8-4092-bc1c-4764a0d9b659" />

搜尋相關訊息和是否有公開漏洞，使用Google搜尋到WonderCMS和CVE-2023-41425漏洞。


<img width="412" height="189" alt="螢幕擷取畫面 2026-02-13 171859" src="https://github.com/user-attachments/assets/ed738405-3080-4292-bc23-ea22b2f0fd5b" />
<img width="1210" height="827" alt="螢幕擷取畫面 2026-02-13 180754" src="https://github.com/user-attachments/assets/eec4fa9c-8884-43cd-ae29-d3d1197d7322" />
<img width="758" height="490" alt="螢幕擷取畫面 2026-02-13 184243" src="https://github.com/user-attachments/assets/edf62653-a31a-41c4-a085-9f196631a9f8" />
<img width="1836" height="341" alt="螢幕擷取畫面 2026-02-13 182221" src="https://github.com/user-attachments/assets/1171270f-c8b4-4338-83f8-027d674ee47a" />

根據公開漏洞消息，需要尋找到登入頁面的URL，使用Goole搜尋可以輕鬆找到。

<img width="1256" height="642" alt="螢幕擷取畫面 2026-02-13 184435" src="https://github.com/user-attachments/assets/ea7466c5-da2a-4c96-8dea-f4ffea556e94" />
<img width="1321" height="416" alt="螢幕擷取畫面 2026-02-13 184500" src="https://github.com/user-attachments/assets/f24d928e-e622-46bd-8864-d5eedfaf7ace" />


### 2.3 初始存取(Initial Access)

使用Burp Suite確認XSS的注入點，用最小PoC驗證。

<img width="1705" height="830" alt="螢幕擷取畫面 2026-02-13 185745" src="https://github.com/user-attachments/assets/b67f08c3-b441-4b6f-bcf9-ecf7f3c34c7d" />
<img width="1277" height="695" alt="螢幕擷取畫面 2026-02-13 185842" src="https://github.com/user-attachments/assets/54dc8376-b7ba-4e19-8e67-698c5495e06b" />

也可使用curl手動驗證

### 2.4 橫向移動(Lateral Movement)
### 2.5 權限提升(Privilege Escalation)
### 2.6 最終成果(Impact)

---

## 3. 學習回顧 (Lessons Learned)

### 成功的部分：
### 浪費時間的部分：
### 新知識點：
### 與實戰對應：
