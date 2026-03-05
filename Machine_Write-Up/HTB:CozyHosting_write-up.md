## 1. 執行摘要 (Executive Summary)

### 測試範圍與目標：
1. 靶機名稱:CozyHosting
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
### 2.2 枚舉(Enumeration)
### 2.3 初始存取(Initial Access)
note.空格改成${IFS} base64編碼 |解碼 執行 全部再URL編碼
### 2.4 橫向移動(Lateral Movement)
### 2.5 權限提升(Privilege Escalation)

ssh有一個功能ProxyCommand，用於通過跳板機代理到遠程主機，用參數-o為引導，指定到遠程主機，但因提權使用，並不想連接到哪裡，所以最後補一個x不讓它空著

提權邏輯來自bash 0<&2 1>&2，但單純輸入這樣，ssh在開始密鑰交換時會因為我們做的重定向而無法與主機通訊導致錯誤，所以前面先給SSH個空命令並用;隔開，讓它先成功，然後再讓ssh執行系統命令提權邏輯

### 2.6 最終成果(Impact)

---

## 3. 學習回顧 (Lessons Learned)

### 成功的部分：
### 浪費時間的部分：
### 新知識點：
### 與實戰對應：
