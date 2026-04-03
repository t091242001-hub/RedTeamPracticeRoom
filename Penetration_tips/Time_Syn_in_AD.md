### 時間偏差問題

在紅隊視角中，時間偏差問題常常成為AD滲透或橫向移動中的隱形障礙，尤其是利用Kerberos或NTLM等需要用到時間戳的協議的時候，一旦目標機器和攻擊機有時間偏差，就會導致驗證失敗或在系統中留下異常日誌。因為Windows對於這種協議的時間偏差只有很小的容忍範圍，通常在五分鐘左右。

原則上，只要是執行Kerberos或NTLM相關的攻擊，都要先想到作時間同步：

```bash
sudo ntpdate IP
sudo net time set -S IP
```

ntpdate走NTP(Network Time Protocol)協議，123 UDP port，不太會留下系統日誌或被管理員注意到，但若UDP123沒開放則無法使用。

net命令中的時間調整(time set)命令，以IP作為源頭(Source)，沒有任何返回消息，走的是SMB(445,135,139)的協議，可能留下系統日誌。
