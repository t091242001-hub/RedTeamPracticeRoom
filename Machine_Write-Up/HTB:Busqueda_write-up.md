## 1. 執行摘要 (Executive Summary)

### 測試範圍與目標：
1. 靶機名稱:Busqueda
2. 難度:Eazy
3. IP:10.129.228.217

### 主要發現：
1. 漏洞 ：命令注入漏洞Command Injection (CVE-2023-43364)（風險等級：Critical）
2. 漏洞 ：路徑劫持PATH Hijacking（風險等級：Critical）

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

```




### 2.3 初始存取(Initial Access)
### 2.4 橫向移動(Lateral Movement)
### 2.5 權限提升(Privilege Escalation)
### 2.6 最終成果(Impact)

---

## 3. 學習回顧 (Lessons Learned)

### 成功的部分：
### 浪費時間的部分：
### 新知識點：
### 與實戰對應：
