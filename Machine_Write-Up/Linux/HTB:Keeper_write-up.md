## 1. 執行摘要 (Executive Summary)

### 測試範圍與目標：
1. 靶機名稱:Keeper
2. 難度:Easy
3. IP:10.129.8.249

### 主要發現：
1. 漏洞 ：透過預設管理員憑證獲得內部文件中儲存的SSH憑證及系統存取權（風險等級：Critical）
2. 漏洞 ：透過舊版本的KeePass存在的公開漏洞(CVE-2023-32784)從記憶體轉儲（Memory Dump）中還原出明文主密碼（風險等級：High）

### 攻擊鏈摘要：

透過未修改的預設密碼獲得管理員權限後，針對管理面板進行枚舉，獲得內部使用者的SSH憑證，取得系統控制權，並使用舊版本的KeePass存在的公開漏洞(CVE-2023-32784)解析內部文件，獲得root明文密碼及權限。

### 潛在影響：

1. 網站預設憑證未更改且具有管理員權限，攻擊者可直接控制設備、竊取敏感資料，甚至將設備作為跳板，滲透進企業內部網路。
2. 舊版本的KeePass存在的公開漏洞(CVE-2023-32784)將洩漏root的明文密碼，造成攻擊者獲得系統完全控制權、重大資料外洩。

### 修復建議：

1. 立即更改預設憑證且開啟多因素驗證 (MFA)。
2. 立即更新KeePass至最新版本，立即更換資料庫主密碼並清理殘留。

---

## 2. 技術發現與攻擊路徑詳述 (Findings)

### 2.1 資訊蒐集(Reconnaissance)

使用Nmap掃描開放埠
```bash
sudo nmap -sT -Pn --min-rate 5000 -p- 10.129.8.249
sudo nmap -sT -Pn -sV -sC -O -p22,80 10.129.8.249
sudo nmap --script=vuln -p22,80 10.129.8.249
```
<img width="820" height="289" alt="螢幕擷取畫面 2026-02-26 200016" src="https://github.com/user-attachments/assets/6bceac86-b207-49cc-8132-5731e756f21c" />
<img width="924" height="491" alt="螢幕擷取畫面 2026-02-26 200052" src="https://github.com/user-attachments/assets/2f14d24e-3080-47f4-bf27-ff18290e3807" />
<img width="807" height="536" alt="螢幕擷取畫面 2026-02-26 201910" src="https://github.com/user-attachments/assets/95ed345a-c9b5-4f05-9bc1-9105bc83c5e0" />

探索80主頁

<img width="878" height="406" alt="螢幕擷取畫面 2026-02-26 200110" src="https://github.com/user-attachments/assets/8b459f4c-4abd-407a-85a2-b0a53e654768" />

將獲得的URL加入/etc/hosts

<img width="503" height="225" alt="螢幕擷取畫面 2026-02-26 200613" src="https://github.com/user-attachments/assets/2c27c392-2edc-4871-8c1b-0bbcb7f188b2" />

前往連結

<img width="1668" height="814" alt="螢幕擷取畫面 2026-02-26 200633" src="https://github.com/user-attachments/assets/c023a60f-fe58-4378-bd1f-89a86566e863" />


### 2.2 枚舉(Enumeration)

Google搜尋是否有預設憑證

<img width="984" height="854" alt="螢幕擷取畫面 2026-02-26 200801" src="https://github.com/user-attachments/assets/19874f0e-374c-41b8-80ba-384cdedb13fd" />

成功登入

<img width="1874" height="837" alt="螢幕擷取畫面 2026-02-26 200812" src="https://github.com/user-attachments/assets/90114dc5-ae2b-47ea-bddf-b98089e8ef55" />


### 2.3 初始存取(Initial Access)

探索網站內容後取得用戶憑證

<img width="1870" height="608" alt="螢幕擷取畫面 2026-02-26 201209" src="https://github.com/user-attachments/assets/b17e6c4e-abd4-4771-9931-dc0112b5bc23" />
<img width="1859" height="763" alt="螢幕擷取畫面 2026-02-26 201832" src="https://github.com/user-attachments/assets/fd24f717-4e58-471b-8d1f-72d2bf5da396" />

利用獲得的憑證登入SSH

<img width="719" height="395" alt="螢幕擷取畫面 2026-02-26 202027" src="https://github.com/user-attachments/assets/658547c7-e84f-4c14-a820-ef375eb86463" />

獲得控制權

<img width="404" height="155" alt="螢幕擷取畫面 2026-02-26 202123" src="https://github.com/user-attachments/assets/0f44e446-dcd1-47e3-a1d5-62e901e7952f" />


### 2.4 權限提升(Privilege Escalation)

解壓縮很可疑的zip檔

<img width="565" height="158" alt="螢幕擷取畫面 2026-02-26 202854" src="https://github.com/user-attachments/assets/bfcde412-0b2a-4778-bfe0-b067de5d276c" />

因我未見過這種副檔名，Google搜索，獲得公開漏洞和PoC腳本，我會在私底下常是手動實現。

<img width="816" height="819" alt="螢幕擷取畫面 2026-02-26 203009" src="https://github.com/user-attachments/assets/7776b892-7a17-4987-92fa-fe356557a064" />
<img width="1743" height="778" alt="螢幕擷取畫面 2026-02-26 203326" src="https://github.com/user-attachments/assets/21bce891-fa6e-49ca-88b1-846b951cdc29" />

下載PoC腳本並賦予使用權限

<img width="1202" height="456" alt="螢幕擷取畫面 2026-02-26 203518" src="https://github.com/user-attachments/assets/0a88c4ee-4bbb-45e0-87f9-6c9ddb189855" />
<img width="1866" height="266" alt="螢幕擷取畫面 2026-02-26 203534" src="https://github.com/user-attachments/assets/ea6c3952-5e9b-40e4-b385-4a78627a1f93" />
<img width="673" height="157" alt="螢幕擷取畫面 2026-02-26 203631" src="https://github.com/user-attachments/assets/04511aa6-5b8c-4262-b3c3-1244959c2c23" />

使用腳本爆破密碼

<img width="586" height="292" alt="螢幕擷取畫面 2026-02-26 203939" src="https://github.com/user-attachments/assets/90dd16a8-7608-4da8-93b5-789ee2c01f0a" />

將帶有亂碼的密碼帶入Google搜尋，找到可疑語詞

<img width="819" height="692" alt="螢幕擷取畫面 2026-02-26 204006" src="https://github.com/user-attachments/assets/a7b36070-1e1d-4aaa-b8f5-5a03a237c8a4" />

安裝KeepPass2

<img width="832" height="460" alt="螢幕擷取畫面 2026-02-26 204642" src="https://github.com/user-attachments/assets/8f779468-8df1-4388-9258-d004a71c746b" />

使用sshpass下載.kdbx檔

<img width="836" height="125" alt="螢幕擷取畫面 2026-02-26 204857" src="https://github.com/user-attachments/assets/aeae4d65-e132-4c42-adcc-e602dd61c89d" />

嘗試搜索到的密碼後成功

<img width="942" height="788" alt="螢幕擷取畫面 2026-02-26 205000" src="https://github.com/user-attachments/assets/8600d188-ff64-4e95-bac1-c8f9b82d3eda" />
<img width="913" height="783" alt="螢幕擷取畫面 2026-02-26 205139" src="https://github.com/user-attachments/assets/3e24a308-f5d3-4ad9-a5f9-b0218c50dde9" />

將獲得的秘文存成.ppk檔

<img width="714" height="607" alt="螢幕擷取畫面 2026-02-26 205721" src="https://github.com/user-attachments/assets/c68470aa-0ac2-49de-9913-48962c39aa38" />

使用puttygen轉成.pem檔

<img width="631" height="129" alt="螢幕擷取畫面 2026-02-26 205826" src="https://github.com/user-attachments/assets/1ea28f70-753a-4b42-975b-bd1db80cab3f" />

嘗試SSH連線

<img width="1097" height="229" alt="螢幕擷取畫面 2026-02-26 205837" src="https://github.com/user-attachments/assets/c1309d42-fc6a-422f-ae36-4c3ffe33e9f1" />


### 2.5 最終成果(Impact)

獲得root權限和root.txt

<img width="980" height="209" alt="螢幕擷取畫面 2026-02-26 205924" src="https://github.com/user-attachments/assets/9ae0cb44-2344-42b0-9aeb-0264938085ff" />

---

## 3. 學習回顧 (Lessons Learned)

### 成功的部分：

1. 透過簡單的Google搜尋就可以蒐集到所有需要的東西，包括預設憑證、PoC以及各種知識。
2. 權限提升的部分我沒遇過，是邊學邊打。

### 浪費時間的部分：

1. 搜索管理員頁面時我嘗試優先尋找上傳頁面，但應該先整體看完再打下去，避免跳進兔子洞。

### 新知識點：

1. 關於KeepPass2是初次使用。
2. 關於.dmp的爆破方法。
3. 關於.ppk轉成.pem可用於SSH密鑰。

### 與實戰對應：

1. 遇到不熟的名詞應妥善運用Google。
2. 任何登入頁都應該先搜尋是否有未清理的預設憑證。
