# gobuster 辞書ファイル一覧

Kali Linux にデフォルトで入っている wordlist の一覧と使用シチュエーション。

---

## `/usr/share/wordlists/dirb/`

| ファイル名 | 語数 | 使うシチュエーション |
|---|---|---|
| `common.txt` | 4,614語 | 最初の軽い列挙。速度優先。CTFの簡単なマシンはこれで十分 |
| `big.txt` | 20,469語 | common.txt で見つからないとき。中程度の深さの列挙 |
| `small.txt` | 959語 | 時間が極端にない時・応答が遅いサーバー向け |
| `extensions_common.txt` | 拡張子リスト | `-x` オプションと組み合わせて拡張子ブルートフォース |
| `vulns/` 配下各種 | 脆弱性特化 | Apache / IIS / CGI など特定サーバー向けの既知パス |

---

## `/usr/share/seclists/Discovery/Web-Content/`（メイン戦場）

| ファイル名 | 語数 | 使うシチュエーション |
|---|---|---|
| `directory-list-2.3-small.txt` | 87,664語 | 軽めの列挙。HTB の Easy マシンはほぼこれで足りる |
| `directory-list-2.3-medium.txt` | 220,560語 | **最もよく使う。** OSCP のデフォルト選択肢 |
| `directory-list-2.3-big.txt` | 1,273,833語 | medium で見つからない時。時間がかかるので終盤に使う |
| `raft-small-directories.txt` | 17,770語 | 実在するディレクトリ名ベースで精度高め。medium の代替候補 |
| `raft-medium-directories.txt` | 30,000語 | raft の中間。directory-list-2.3-medium の代替として使いやすい |
| `raft-large-directories.txt` | 62,284語 | raft で深堀りするとき |
| `raft-small-files.txt` | 17,452語 | **ファイル名を探すとき**（config, backup 等）に特化 |
| `raft-medium-files.txt` | 35,263語 | ファイル探索の中程度 |
| `common.txt` | 4,724語 | dirb の common.txt の代替。ffuf でもよく使われる |
| `quickhits.txt` | 2,369語 | 既知の危険パス（.git, .env, phpinfo.php 等）を素早くチェック |
| `apache.txt` | 小規模 | Apache 固有のパス限定で探すとき |
| `nginx.txt` | 小規模 | nginx 固有パス |
| `IIS.txt` | 小規模 | IIS / Windows 環境の Web 列挙 |
| `CMS/wordpress.txt` など | CMS ごと | WordPress や Drupal 等が確定したあとの深堀り |

---

## `/usr/share/seclists/Discovery/DNS/`（サブドメイン列挙）

gobuster の `dns` モードで使う。

| ファイル名 | 使うシチュエーション |
|---|---|
| `subdomains-top1million-5000.txt` | 最初に試す軽量リスト。応答が遅いサーバー向け |
| `subdomains-top1million-20000.txt` | 標準的なサブドメイン列挙。バランスが良い |
| `subdomains-top1million-110000.txt` | 深堀り用。見つからない時の最終手段 |
| `bitquark-subdomains-top100000.txt` | 別ソース由来。上記と組み合わせて精度を上げる |

---

## `/usr/share/seclists/Discovery/Web-Content/` — vhost 列挙

gobuster の `vhost` モードで使う。

| ファイル名 | 使うシチュエーション |
|---|---|
| `subdomains-top1million-5000.txt` | vhost を素早く確認したいとき |
| `subdomains-top1million-20000.txt` | 標準的な vhost 列挙 |

---

## 使い分けフローチャート

```
目的は何か？
│
├─ ディレクトリ列挙（dir モード）
│   ├─ まず試す        → directory-list-2.3-medium.txt
│   ├─ 時間を節約したい → directory-list-2.3-small.txt
│   ├─ 深堀りしたい    → directory-list-2.3-big.txt
│   ├─ ファイルを探す  → raft-medium-files.txt + -x php,txt,html
│   └─ 危険パス確認   → quickhits.txt
│
├─ サブドメイン列挙（dns モード）
│   ├─ まず試す        → subdomains-top1million-20000.txt
│   └─ 深堀り          → subdomains-top1million-110000.txt
│
└─ vhost 列挙（vhost モード）
    └─ まず試す        → subdomains-top1million-5000.txt
```

---

## よく使うコマンド例

```bash
# 基本的なディレクトリ列挙
gobuster dir -u http://TARGET -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt

# 拡張子指定あり
gobuster dir -u http://TARGET -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -x php,txt,html,bak

# 危険パスの素早いチェック
gobuster dir -u http://TARGET -w /usr/share/seclists/Discovery/Web-Content/quickhits.txt

# サブドメイン列挙
gobuster dns -d TARGET.com -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt

# vhost 列挙
gobuster vhost -u http://TARGET -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt --append-domain
```

---

## 補足

- **SecLists が入っていない場合**: `sudo apt install seclists` でインストール
- **TJnull の OSCP 推奨**: `directory-list-2.3-medium.txt` が試験環境でも最もよく使われる
- **スレッド数**: `-t 50` 程度が安定。サーバーが落ちそうな場合は `-t 10` に下げる
- **ステータスコード除外**: `-b 403,404` で不要なレスポンスを除外するとノイズが減る
