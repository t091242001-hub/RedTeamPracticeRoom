## 1. 執行摘要 (Executive Summary)

### 測試範圍與目標：
1. 靶機名稱:
2. 難度:
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
sudo nmap -sT -Pn --min-rate 5000 -p- 10.129.230.220
sudo nmap -sT -Pn -sV -sC -O -p99,22,8080 10.129.230.220
sudo nmap --script=vuln -p22,8080 10.129.230.220
```

<img width="614" height="254" alt="螢幕擷取畫面 2026-03-20 214903" src="https://github.com/user-attachments/assets/253d2e2a-3bd2-4879-a652-014948c2d598" />
<img width="1121" height="574" alt="螢幕擷取畫面 2026-03-20 214920" src="https://github.com/user-attachments/assets/5b9093d6-0a84-4320-801f-e5b9a71d0553" />
<img width="804" height="600" alt="螢幕擷取畫面 2026-03-20 220218" src="https://github.com/user-attachments/assets/efec06a3-10a5-478a-8e51-7837f9e84e3f" />

探索8080頁面

<img width="1903" height="894" alt="螢幕擷取畫面 2026-03-20 215104" src="https://github.com/user-attachments/assets/45283f88-5333-4a07-88ec-e0940929cd7c" />

找到用戶名

<img width="1155" height="473" alt="螢幕擷取畫面 2026-03-20 220615" src="https://github.com/user-attachments/assets/52ee3479-8ee5-46f3-b549-0d3d9c3d2c19" />



探索版本公開漏洞

<img width="754" height="740" alt="螢幕擷取畫面 2026-03-20 215636" src="https://github.com/user-attachments/assets/4e67b080-feed-44bd-8d7f-f98473257909" />
<img width="1343" height="778" alt="螢幕擷取畫面 2026-03-20 220341" src="https://github.com/user-attachments/assets/fd11f122-ed3a-4e01-876a-89cc8d751381" />


### 2.2 枚舉(Enumeration)

使用gobuster爆破目錄
```bash
sudo gobuster dir -u http://10.129.230.220:8080 -w /usr/share/wordlists/dirb/common.txt
```

<img width="858" height="742" alt="螢幕擷取畫面 2026-03-20 215850" src="https://github.com/user-attachments/assets/2c95727e-06bc-48c6-9130-fcaae1fd1b41" />



### 2.3 初始存取(Initial Access)

利用公開漏洞，Jenkins使用args4j來解析命令行輸入，並支援透過HTTP、Websocket等協定遠端確定命令列參數。
args4j中使用者可以透過@字元來載入任何文件，這導致攻擊者透過該特性來讀取伺服器上的任何文件。

<img width="1413" height="815" alt="螢幕擷取畫面 2026-03-20 221240" src="https://github.com/user-attachments/assets/f4aef640-26aa-4434-ab0e-6f3686718d32" />

按照PoC先下載/jnlpJars/jenkins-cli.jar

<img width="1684" height="372" alt="螢幕擷取畫面 2026-03-20 221314" src="https://github.com/user-attachments/assets/a737cd5d-7360-4ff5-afbd-5b020fe91c86" />

驗證PoC成功
```bash
java -jar jenkins-cli.jar -s http://10.129.230.220:8080 -http help 1 "@/var/jenkins_home/secret.key"
```

<img width="993" height="149" alt="螢幕擷取畫面 2026-03-20 221910" src="https://github.com/user-attachments/assets/a28c9271-278c-42b8-bced-22480599019b" />

嘗試讀取內部文件，找到只有一個用戶名擁有/bin/bash
```bash
java -jar jenkins-cli.jar -s http://10.129.230.220:8080 -http connect-node "@/etc/passwd"
```

<img width="1352" height="451" alt="螢幕擷取畫面 2026-03-20 222014" src="https://github.com/user-attachments/assets/2119fde5-7926-4994-8ab1-4ef8c9f04e53" />

因我未能找到命令注入的方式，只能期望敏感文件有洩漏憑證，我造訪了Jenkins的配置說明書和其他教學文，查詢敏感目錄

<img width="1273" height="704" alt="螢幕擷取畫面 2026-03-20 223303" src="https://github.com/user-attachments/assets/a3f414b4-1fd0-436d-808b-ff76f7c2fdde" />
<img width="1201" height="234" alt="螢幕擷取畫面 2026-03-20 223608" src="https://github.com/user-attachments/assets/ae4ad4b2-ffe8-4900-b0a0-bb886843250a" />
<img width="1199" height="743" alt="螢幕擷取畫面 2026-03-20 223812" src="https://github.com/user-attachments/assets/27ac2cc8-9eae-44bd-93c7-32f39ff9998e" />
<img width="1072" height="386" alt="螢幕擷取畫面 2026-03-20 223828" src="https://github.com/user-attachments/assets/7ab05e6e-e154-4423-b9de-d86394f302cf" />

探索敏感資訊，如果不存在就會報錯

<img width="1188" height="212" alt="螢幕擷取畫面 2026-03-20 224642" src="https://github.com/user-attachments/assets/94997b1e-b3db-49f8-b908-f8f9ef6da6c8" />

在敏感文件中獲得用戶目錄名

<img width="1193" height="284" alt="螢幕擷取畫面 2026-03-20 224652" src="https://github.com/user-attachments/assets/febda470-1623-4bbf-aea3-a29cbf0a7b4c" />

探索敏感配置文件，並在最末尾發現洩漏憑證

<img width="1647" height="270" alt="螢幕擷取畫面 2026-03-20 224940" src="https://github.com/user-attachments/assets/41b1ebf4-7e67-46bc-b5e7-aacd19314445" />
<img width="1084" height="321" alt="螢幕擷取畫面 2026-03-20 225017" src="https://github.com/user-attachments/assets/efaccac3-c1c6-4b02-ac53-d6a6da796856" />

破解密碼雜湊，先將密碼雜湊儲存成文件，讓hashcat判別雜湊類型，再根據類型使用字典攻擊
(hashcat是GPU運算，所以破解速度會比john使用CPU快，缺點就是不能自動偵測類型，比john多一個步驟。我絕大多數情況都是用john，爆不出來才用hashcat，因為靶機不會出太刁鑽的雜湊類型，不過以防萬一，不能不會hashcat。)

<img width="577" height="141" alt="螢幕擷取畫面 2026-03-20 230101" src="https://github.com/user-attachments/assets/fcd966d8-bf8c-4f29-bffc-37401d7598bd" />
<img width="1370" height="361" alt="螢幕擷取畫面 2026-03-20 230102" src="https://github.com/user-attachments/assets/980fe4ea-143d-4a21-be3a-bba530e174c3" />
<img width="689" height="669" alt="螢幕擷取畫面 2026-03-20 230517" src="https://github.com/user-attachments/assets/382f65a8-8dfc-413b-9b7f-c7522aede809" />

配合資訊蒐集找到的用戶名登入管理頁面

<img width="1914" height="481" alt="螢幕擷取畫面 2026-03-20 230729" src="https://github.com/user-attachments/assets/14c724f4-4a30-4db0-8ec2-c66f3c35a742" />


### 2.4 權限提升(Privilege Escalation)

在內部探索後，了解到管理葉面中可能託管有root的SSH私鑰

<img width="1918" height="944" alt="螢幕擷取畫面 2026-03-20 230730" src="https://github.com/user-attachments/assets/604ecc1d-3fdc-45fa-912a-675b1a8d709d" />

並且了解到管理頁面有關於SSH的插件

<img width="1915" height="775" alt="螢幕擷取畫面 2026-03-20 231833" src="https://github.com/user-attachments/assets/55369b96-9580-4ee8-bcef-56d2651edfa8" />

我去官方網頁了解了SSH插件的寫法

<img width="1876" height="789" alt="螢幕擷取畫面 2026-03-20 232901" src="https://github.com/user-attachments/assets/79dc774a-c34a-438b-9d97-a8dea4c13a57" />

根據官方網頁的說明先創建管道

<img width="1624" height="719" alt="螢幕擷取畫面 2026-03-20 233249" src="https://github.com/user-attachments/assets/96b09e3b-60ce-4927-8efd-518a21135b9a" />
<img width="1395" height="562" alt="螢幕擷取畫面 2026-03-20 233310" src="https://github.com/user-attachments/assets/45a30847-774b-4f2e-8c29-a526713e9d93" />

更改為SSH的寫法，並且更改IP位址以及用戶名代號

<img width="1424" height="667" alt="螢幕擷取畫面 2026-03-20 233558" src="https://github.com/user-attachments/assets/fda20b38-a139-44c3-b7d6-746f170eeff5" />

嘗試部署管道(Build Now)

<img width="386" height="335" alt="螢幕擷取畫面 2026-03-20 233639" src="https://github.com/user-attachments/assets/54135c95-f5a0-4b6f-928c-71bd3bda17df" />

嘗試讀取執行結果(Console Output)

<img width="423" height="442" alt="螢幕擷取畫面 2026-03-20 233715" src="https://github.com/user-attachments/assets/cd6e1fde-8ef1-4910-9609-055093c67b9b" />
<img width="1029" height="609" alt="螢幕擷取畫面 2026-03-20 233745" src="https://github.com/user-attachments/assets/f07d65be-4ba6-4e9f-802a-6266daac1d96" />

既然讀取成功，就可以嘗試取得Shell，這裡應該要想到多種方法，我決定偷渡SSH公鑰

<img width="1408" height="634" alt="螢幕擷取畫面 2026-03-20 233928" src="https://github.com/user-attachments/assets/eed89f64-980d-4081-af1b-5e584bdf781d" />

經過同樣的部屬階段後，我成功讀取了公鑰

<img width="872" height="643" alt="螢幕擷取畫面 2026-03-20 234002" src="https://github.com/user-attachments/assets/a083a78b-ec3e-4472-95cc-d3046a9d207e" />

將公鑰儲存成文件，並給予執行權限600

<img width="703" height="600" alt="螢幕擷取畫面 2026-03-20 234106" src="https://github.com/user-attachments/assets/e88f8199-d584-4381-bfad-f0d832c26762" />
<img width="550" height="124" alt="螢幕擷取畫面 2026-03-20 234237" src="https://github.com/user-attachments/assets/b05b2b89-d108-4765-b779-91e24c5b16d2" />

透過公鑰成功連結SSH


<img width="939" height="690" alt="螢幕擷取畫面 2026-03-20 234248" src="https://github.com/user-attachments/assets/f32ee1ad-37d3-4eb2-b78e-9d31b91c21bc" />



### 2.5 最終成果(Impact)

取得user.txt及root.txt
(在利用Java讀取文件時，其實就可以讀取user.txt，但我並沒有那麼做，因為我並沒有獲得系統控制權，我個人認為未獲得系統控制權取得Flag並不理想，所以我才在最後用root的身分讀取user.txt。)

<img width="422" height="189" alt="螢幕擷取畫面 2026-03-20 234336" src="https://github.com/user-attachments/assets/ab125676-4753-473a-8c57-ef215c579115" />
<img width="1010" height="222" alt="螢幕擷取畫面 2026-03-20 234611" src="https://github.com/user-attachments/assets/29836de9-b374-4f30-a517-f2bef01c9ef1" />




---

## 3. 學習回顧 (Lessons Learned)

### 成功的部分：
### 浪費時間的部分：
### 新知識點：
### 與實戰對應：
