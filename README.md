FAIL2BAN
--------------------

Fail2Ban 可以用來防護 Linux Server 上的 SSH、vsftp、dovecot...等服務免於遭駭客使用暴力密碼入侵。<br>

### Centos 7 安裝
1. 先安裝 EPLE
```shell
sudo yum install epel-release
```
2. 安裝fail2ban
```shell
sudo yum install fail2ban
```
3. 啟動fail2ban
```shell
sudo systemctl enable fail2ban
sudo systemctl start fail2ban
```

fail2ban 的設定會在 /etc/fail2ban裡面。<br>
裡面原本有 jail.conf 這個檔案，裡面有預設配置。
可以直接修改 jail.conf ，也可以另外加一個客製檔案
編輯一個檔案 jail.local
```markdown
[DEFAULT]
ignoreip = 127.0.0.1/8
bantime  = 300 # 封鎖的時間，單位:秒，300=5分鐘，改為 -1 表示「永久」封鎖
findtime = 600 # 在多久的時間內，單位:秒，600=10分鐘
maxretry = 3 # 登入失敗幾次封鎖

[sshd]
enabled  = true
# port     = ssh
filter   = sshd
# logpath  = /var/log/fail2ban.log
backend = systemd
# maxretry = 5
# findtime = 600
# bantime  = 1200
action   = iptables[name=SSH, port=ssh, protocol=tcp]
```
在採用 systemd 的系統上， 請把 logpath 註解掉， 改用 backend = systemd。<br>
儲存後重啟fail2ban
```shell
systemctl restart fail2ban
```

### 測試
另外開一個ssh進行連線，輸入三次錯誤。
回到伺服器上
有兩個地方可以查看被封鎖的IP
1. fail2ban-client status sshd
```shell
[root@localhost ~]# fail2ban-client status sshd
Status for the jail: sshd
|- Filter
|  |- Currently failed: 1
|  |- Total failed:     4
|  `- Journal matches:  _SYSTEMD_UNIT=sshd.service + _COMM=sshd
`- Actions
   |- Currently banned: 1
   |- Total banned:     1
   `- Banned IP list:   192.168.0.197

```
2. 查看log
```shell
2019-12-19 04:35:17,924 fail2ban.filter         [32315]: INFO    [sshd] Found 192.168.43.81 - 2019-12-19 04:35:17
2019-12-19 04:35:29,289 fail2ban.filter         [32315]: INFO    [sshd] Found 192.168.43.81 - 2019-12-19 04:35:29
2019-12-19 04:35:53,855 fail2ban.filter         [32315]: INFO    [sshd] Found 192.168.43.81 - 2019-12-19 04:35:53
2019-12-19 04:35:54,248 fail2ban.actions        [32315]: NOTICE  [sshd] Ban 192.168.43.81
2019-12-19 04:40:55,310 fail2ban.actions        [32315]: NOTICE  [sshd] Unban 192.168.43.81
```

手動解除IP鎖定狀態
```shell
fail2ban-client set sshd unbanip yourip
```

```shell
[root@localhost ~]# fail2ban-client status sshd
Status for the jail: sshd
|- Filter
|  |- Currently failed: 0
|  |- Total failed:     25
|  `- Journal matches:  _SYSTEMD_UNIT=sshd.service + _COMM=sshd
`- Actions
   |- Currently banned: 1
   |- Total banned:     2
   `- Banned IP list:   192.168.0.197
[root@localhost ~]# fail2ban-client set sshd unbanip 192.168.0.197
192.168.0.197
[root@localhost ~]# fail2ban-client status sshd
Status for the jail: sshd
|- Filter
|  |- Currently failed: 0
|  |- Total failed:     25
|  `- Journal matches:  _SYSTEMD_UNIT=sshd.service + _COMM=sshd
`- Actions
   |- Currently banned: 0
   |- Total banned:     2
   `- Banned IP list:
```
fail2ban.log 裡面可以看到
```markdown
2019-12-05 10:53:35,313 fail2ban.actions        [1985]: NOTICE  [sshd] Ban 192.168.0.197
2019-12-05 10:53:35,350 fail2ban.actions        [1985]: NOTICE  [sshd] 192.168.0.197 already banned
2019-12-05 10:57:03,422 fail2ban.actions        [1985]: NOTICE  [sshd] Unban 192.168.0.197
```

fail2ban 保護 phpMyadmin
--------------------
**主要問題**<br>
fail2ban 在過往的版本有已經設定好的config可以直接使用，來進行phpMyadmin的安全保護，<br>
但是新版本此config被刪除，必須自己寫config以及log給fail2ban使用。<br>

fail2ban實際運作方式是：
1. 查看指定的log，根據設定的log格式找尋是否符合要求
2. 如果新的log符合格式，記住IP，並且次數+1，並開始計算時間

#### 處理方式
1. 找出是否有可以代表phpmyadmin登入失敗的log內容
2. 寫config，判斷內容
3. 在jail.local呼叫config和log並進行偵測
<br>
1. phpmyadmin沒有自己的log，但是可以去apache的access_log找<br>
連續2次登入失敗
```markdown
192.168.43.81 - - [19/Dec/2019:05:43:21 +0800] "POST /phpmyadmin/index.php HTTP/1.1" 200 3679 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/79.0.3945.79 Safari/537.36"
192.168.43.81 - - [19/Dec/2019:05:43:37 +0800] "POST /phpmyadmin/index.php HTTP/1.1" 200 3679 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/79.0.3945.79 Safari/537.36"
```
登入成功
```markdown
192.168.43.81 - - [19/Dec/2019:05:43:49 +0800] "POST /phpmyadmin/index.php HTTP/1.1" 302 20 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/79.0.3945.79 Safari/537.36"
192.168.43.81 - - [19/Dec/2019:05:43:49 +0800] "GET /phpmyadmin/index.php HTTP/1.1" 200 14065 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/79.0.3945.79 Safari/537.36"
192.168.43.81 - - [19/Dec/2019:05:43:50 +0800] "GET /phpmyadmin/phpmyadmin.css.php?nocache=4725525373ltr&server=1 HTTP/1.1" 200 20802 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/79.0.3945.79 Safari/537.36"
192.168.43.81 - - [19/Dec/2019:05:43:50 +0800] "POST /phpmyadmin/ajax.php HTTP/1.1" 200 1497 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/79.0.3945.79 Safari/537.36"
192.168.43.81 - - [19/Dec/2019:05:43:50 +0800] "POST /phpmyadmin/ajax.php HTTP/1.1" 200 1592 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/79.0.3945.79 Safari/537.36"
192.168.43.81 - - [19/Dec/2019:05:43:50 +0800] "POST /phpmyadmin/ajax.php HTTP/1.1" 200 1489 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/79.0.3945.79 Safari/537.36"
192.168.43.81 - - [19/Dec/2019:05:43:50 +0800] "POST /phpmyadmin/version_check.php HTTP/1.1" 200 64 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/79.0.3945.79 Safari/537.36"
192.168.43.81 - - [19/Dec/2019:05:59:33 +0800] "GET /phpmyadmin/themes/pmahomme/img/s_unlink.png HTTP/1.1" 200 589 "http://192.168.43.109/phpmyadmin/phpmyadmin.css.php?nocache=4725525373ltr&server=1" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/79.0.3945.79 Safari/537.36"
```
登出
```markdown
192.168.43.81 - - [19/Dec/2019:05:59:34 +0800] "POST /phpmyadmin/logout.php HTTP/1.1" 302 8761 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/79.0.3945.79 Safari/537.36"
192.168.43.81 - - [19/Dec/2019:05:59:34 +0800] "GET /phpmyadmin/index.php HTTP/1.1" 200 3633 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/79.0.3945.79 Safari/537.36"
```
<br>
可以看出一些規則，中間有一些內容是登入失敗才會出現的地方<br>
... "POST /phpmyadmin/index.php HTTP/1.1" 200 ...<br>
可以把這個當作判斷依據<br><br>

2. 寫一個專用的config，我命名為 phpmyadmin-9skin.conf<br>
最主要的部份是設定failregex，其中的動點是要抓出IP，是使用<HOST>的標籤來處理
```markdown
[Definition]

# Option:  failregex
# Notes.:  regex to match the password failure messages in the logfile. The
#          host must be matched by a group named "host". The tag "<HOST>" can
#          be used for standard IP/hostname matching and is only an alias for
#          (?:::f{4,6}:)?(?P<host>\S+)
# Values:  TEXT
#
failregex = <HOST> .* "POST /phpmyadmin/index.php HTTP/1.1" 200 .*

# Option:  ignoreregex
# Notes.:  regex to ignore. If this regex matches, the line is ignored.
# Values:  TEXT
#
ignoreregex =
```
把phpmyadmin-9skin.conf放在/etc/fail2ban/filter.d 裡。
<br><br>
3. 把之前寫的jail.local更新
```markdown
[DEFAULT]
ignoreip = 127.0.0.1/8
bantime  = 300 # 封鎖的時間，單位:秒，600=10分鐘，改為 -1 表示「永久」封鎖
findtime = 600 # 在多久的時間內，單位:秒，600=10分鐘
maxretry = 3 # 登入失敗幾次封鎖

[sshd]
enabled  = true
# port     = ssh
filter   = sshd
# logpath  = /var/log/fail2ban.log
backend = systemd
# maxretry = 5
# findtime = 600
# bantime  = 1200
action   = iptables[name=SSH, port=ssh, protocol=tcp]

[phpmyadmin-9skin]
enabled  = true
filter   = phpmyadmin-9skin
action   = iptables[name=HTTP, port=http, protocol=tcp]
logpath  = /var/log/httpd/access_log
# maxretry = 3
# bantime  = 600
```
完成上面的步驟，重啟fail2ban。<br>
最後再來測試ssh和phpmyadmin是否都有擋住。<br>
<br>

讓fail2ban自動發送e-mail提示訊息
--------------------
總不可能一直守在終端機前用指令去看有沒有IP被BAN，fail2ban能夠在ban某個IP的同時自動發送e-mail，提醒用戶。<br>
他是既有的config，在 /etc/fail2ban/action.d/ 裡的 sendmail-whois.conf <br>
config內容就不多說，直接看如何調用這個conf<br>
在 jail.local 裡面 action 標籤加入 sendmail-whois[name=SSH, dest=收件者EMail, sender=顯示的寄件者EMail] <br>
sender可以不寫。<br>
<br>
jail.local 檔案到這邊已經算是全部配置完成了，
```markdown
[DEFAULT]
ignoreip = 127.0.0.1/8	# 白名單IP
bantime  = 300	# 封鎖的時間，單位:秒，300=5分鐘，改為 -1 表示「永久」封鎖
findtime = 600	# 在多久的時間內，單位:秒，600=10分鐘
maxretry = 3	# 登入失敗幾次封鎖

[sshd]
enabled  = true
# port     = ssh
filter   = sshd
# logpath  = /var/log/fail2ban.log
backend = systemd
# maxretry = 5
# findtime = 600
# bantime  = 1200
action   = iptables[name=SSH, port=ssh, protocol=tcp]
           sendmail-whois[name=SSH, dest=paul@9skin.com]

[phpmyadmin-9skin]
enabled  = true
filter   = phpmyadmin-9skin
action   = iptables[name=HTTP, port=http, protocol=tcp]
           sendmail-whois[name=SSH, dest=paul@9skin.com]
logpath  = /var/log/httpd/access_log
# maxretry = 3
# bantime  = 600
```
最後記得<br>
重新啟動 fail2ban