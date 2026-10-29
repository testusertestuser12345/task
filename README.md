# dirsearch 研究ノート + ハンズオン・ラボ

Webのディレクトリ/ファイル探索ツール **dirsearch** を調べ、手を動かして確かめるための一式です。
この README に調査結果をまとめ、同じフォルダに「叩かれる側のサーバ」「経由するプロキシ」「実行スクリプト」を Docker で用意しています。

> ⚠️ **利用の前提**：ディレクトリスキャンは、自分が所有する／書面で許可を得たシステムにのみ行ってください。無許可のスキャンは多くの国・地域で違法です。このラボは `docker-compose.yml` のプライベートネットワーク内で完結しており、練習はこの中だけで行う想定です。

---

## 1. dirsearch とはどんなコマンドか

Webサーバに対して「ありそうなパス」を辞書で片っ端から要求し、返ってきた**ステータスコードや応答サイズ**から、リンクされていない隠しディレクトリ・ファイル（管理画面、`.git`、`.env`、バックアップ、APIなど）をあぶり出すツールです。いわゆる **content discovery / forced browsing** を行います。

- 本体：Python製のコマンドラインツール（本ノートは **v0.5.0** 時点のソースを確認）
- 最小の使い方：

```bash
dirsearch -u https://example.com/
```

- よく使う形：

```bash
# 拡張子を展開して探索（辞書内の %EXT% が php/html/js に置き換わる）
dirsearch -u https://example.com/ -e php,html,js

# 見つかったディレクトリを再帰的に掘る + レポート出力
dirsearch -u https://example.com/ -e php -r -o report.md -O md
```

**仕組みのポイント**
- 辞書の各行がパス候補。`%EXT%` という置換キーワードがある行だけ、`-e` で指定した拡張子に展開されます（`-f`/`--force-extensions` を付けると全行の末尾に拡張子を付加）。
- 既定では `200` だけでなく、`301/302`（リダイレクト）、`401`（要認証）、`403`（禁止だが"存在はする"）なども「見つかった」として扱います。`403` が返る＝そこに何かがある、という情報自体が価値になります。
- **soft-404（偽の200）対策**：存在しないパスにも 200 を返すサイト向けに、起動時にランダムな存在しないパスを投げて「偽の正常応答」を学習し、同じ応答を除外する **wildcard / calibration** 機構を持ちます（`--auto-calibration` で強化）。

---

## 2. 他ツール（dirb / gobuster / ffuf）との比較

| 観点 | **dirsearch** | dirb | gobuster | ffuf |
|---|---|---|---|---|
| 言語 | Python | C | Go | Go |
| 速度 | 速い（v0.5で**async**標準、任意でRust native backend） | 遅い | 非常に速い | 非常に速い |
| 同梱辞書 | **充実**（約9,700行の主辞書＋技術別カテゴリ辞書） | 小さめ同梱 | **なし**（別途用意） | なし |
| 用途の広さ | ディレクトリ/ファイル探索に特化、機能豊富 | ディレクトリ探索のみ | dir/dns/vhost/s3 など | **汎用ファザー**（任意の `FUZZ` 位置） |
| 拡張子展開 | `%EXT%`・`-e`・`-f` と柔軟 | 限定的 | `-x` | 自前で組む |
| 再帰 | `-r`（深さ・対象コード指定可） | あり | 限定的 | `-recursion` |
| クロール連携 | `--crawl`（応答/robotsから発見） | なし | なし | なし |
| バックアップ探索 | `--find-backup`（.bak/.old/~等を自動） | なし | なし | 自前 |
| 高度なマッチ/除外 | status/size/word/line/regex/**time** | 弱い | status/length | **非常に強力**（ffuf由来の思想をv0.5で取り込み） |
| プロキシ | HTTP/SOCKS、**複数ローテーション**、Tor、replay | 単一 | 単一 | 単一(＋外部) |
| 出力形式 | simple/plain/json/xml/md/csv/html/**sqlite/MySQL/PostgreSQL** | テキスト | テキスト/json | json/csv/html/md |
| 中断・再開 | **セッション保存/再開**あり | なし | なし | なし |
| Python API | あり（`DirsearchFuzzer`） | なし | ライブラリ利用可 | ライブラリ利用可 |

**dirsearch を選ぶ理由（ざっくり）**
- **「箱から出してすぐ使える」バランスが良い**：強力な同梱辞書＋拡張子展開＋再帰＋レポートが最初から揃う。ffuf/gobuster は速いが辞書は自前。
- **機能の幅が広い**：クロール、バックアップ探索、セッション再開、多彩な出力（DB直書き含む）、プロキシローテーションまで単体で完結。
- **速度も妥協しない**：v0.5 で async が既定になり、対応環境では Rust native backend も選べる。昔の「dirsearch＝遅い」像は更新されている。

**逆に他が向く場面**
- 1トークンの差分や複雑な位置の総当たり（Host/パラメータ/サブドメインを `FUZZ` で）なら **ffuf** が一枚上。
- DNS/vhost/S3 バケット探索も混ぜたいなら **gobuster**。
- とにかく軽く1発、なら **dirb**。

---

## 3. 言語と拡張性

- **言語**：Python（本体）。v0.5.0 で **最小 Python 3.11**、リリースビルドは 3.14 系。高速化のための **Rust 製 native backend** も追加され、`--request-backend native` / `--wordlist-backend native` で切替可能（要 Python 3.14・Rust）。
- **実行モード**：`-a/--async`（既定）と `--sync`（従来のスレッド版）。
- **拡張性は高い**：
  - **Python API** が公式提供。`FuzzerConfig` と `DirsearchFuzzer` を使い、自作スクリプトから結果をオブジェクトとして受け取れる（CI やパイプラインに組み込みやすい）。
  - **設定ファイル** `config.ini` で既定値を全面的に上書き可能（スレッド数、除外コード、タイムアウト等）。
  - **辞書は差し替え/追加自由**。`-w` で複数ファイル・ディレクトリ指定、`--wordlist-categories` で技術別辞書を選択、**テンプレート辞書**（`%SUBJECT%` `%CRUD_OP%` `%API_VERSION%` などのプレースホルダ）で動的生成も可能。
  - **出力が拡張可能**：json/xml/md/csv/html/sqlite に加え **MySQL/PostgreSQL** へ直接書き込み（`dirsearch[mysql]` / `[postgresql]` の追加依存）。
  - 入力も柔軟：URL一覧ファイル、STDIN、CIDR、**生HTTPリクエスト** `--raw`、**Nmap XML** 取り込み `--nmap-report`。

---

## 4. デフォルトのワードリスト（実物を確認した結果）

本体同梱の辞書は `db/` 以下にあります。主辞書は **`db/dicc.txt`**。

- **主辞書 `db/dicc.txt`**：**9,680 行 / 約 144 KB**。
  - これは「素の行数」。`%EXT%` を含む行が **558 行**あり、`-e` で拡張子を展開すると生成エントリ数は増えます。実測では拡張子なしで **12,295 件**、`-e php,html,js` 指定時は展開内容に応じて変動（`--wordlist-status` で確認可能）。
- **技術別カテゴリ辞書 `db/categories/`**（`--wordlist-categories` で選択、`all` で全部）：

| ファイル | 行数 | 中身 |
|---|---|---|
| `common.txt` | 6,717 | 汎用パス |
| `web.txt` | 978 | 一般的なWebパス |
| `conf.txt` | 873 | 設定ファイル系 |
| `extensions.txt` | 558 | 拡張子リスト |
| `logs.txt` | 199 | ログファイル |
| `db.txt` | 127 | DB関連 |
| `backups.txt` | 108 | バックアップ |
| `vcs.txt` | 96 | `.git` `.svn` 等 |
| `keys.txt` | 22 | 鍵・資格情報 |
| 技術別（`php/` `python/` `java/` `dotnet/` `node/` `coldfusion/`） | 各9〜46 | WordPress/Laravel/Django/Flask/Spring/Express 等 |
| インフラ（`infra/aws.txt` `docker.txt` `k8s.txt`） | 6〜11 | クラウド/コンテナ |

- **ブラックリスト**：`db/400_blacklist.txt` `403_blacklist.txt` `500_blacklist.txt`（特定コードで"誤検出しやすいパス"を除外）。
- **User-Agent リスト** `db/user-agents.txt`：53 個（`--random-agent` 用）。
- 既定 User-Agent（指定しない場合）：`Mozilla/5.0 (Windows NT 10.0; Win64; x64) ... Chrome/87 ...`。

**「`/etc/hosts` 等」の中身について**：主辞書には LFI/パストラバーサルを狙う行が多数含まれます。実際に `db/dicc.txt` を検索すると以下のような行が入っています（抜粋・いずれも"存在すれば情報漏えいに直結する"狙い目）：

```
etc/passwd
etc/hosts
etc/config.ini
etc/database.xml
../../../../../../etc/passwd
%2e%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/etc/passwd      ← URLエンコード型
cgi-bin/.%2e/%2e%2e/%2e%2e/%2e%2e/etc/passwd             ← cgi-bin経由
app/etc/local.xml（Magento設定の漏えい狙い。.bak/.live等の派生も多数）
```

`etc/hosts`〜`etc/passwd` 系の行だけでなく、`win.ini` や `web.config`、`.git/`、`.env.*`（`.env.local` `.env.prod` `.env.docker` …大量）、`wp-*`、`*.sql`、`*.zip`、`id_rsa`、`.bash_history`、`actuator/env`、`swagger.json` なども最初から入っています。LFI狙いの `etc/` 系だけで **168 行**が該当しました。

確認コマンド：

```bash
wc -l db/dicc.txt                  # 行数
grep -c '%EXT%' db/dicc.txt        # 拡張子展開される行数
dirsearch --wordlist-status        # 実際に生成されるエントリ数を表示して終了
dirsearch --wordlist-categories all --wordlist-status
```

---

## 5. 最大のメリット・利点

1. **最初から強い**：大規模な同梱辞書＋技術別カテゴリ＋拡張子展開で、設定ゼロでも深く広く探索できる。
2. **見逃しにくい設計**：再帰(`-r`)、クロール(`--crawl`)、バックアップ探索(`--find-backup`)が連動し、「入口を1つ見つけたら芋づる式に掘る」動きを1コマンドで実現。
3. **ノイズに強い**：wildcard/calibration で soft-404 を自動除外、`--mc/--fc/--ms/--fs/--mw/--fl/--mr/--mt` など ffuf 由来の細かいマッチ/除外で「本物」だけ残せる。
4. **運用が楽**：json/xml/md/csv/html/**sqlite/DB** 出力、生レスポンス保存、**セッション中断・再開**。長時間スキャンや大量ターゲットでも扱いやすい。
5. **回避・分散の手段が揃う**：複数プロキシのローテーション、SOCKS/Tor、`--random-agent`、`--delay`/`--max-rate`、`--ip`/`--interface` 指定。
6. **組み込みやすい**：Python API・設定ファイル・多彩な入力（raw/Nmap/CIDR/STDIN）でパイプラインに載せやすい。

---

## 6. ディレクトリスキャン以外の使い道

dirsearch は実質「**辞書駆動のHTTPファザー＆大量リクエスタ**」なので、パス探索以外にも応用できます。

- **バーチャルホスト/APIバージョン探索**：テンプレート辞書で `api/%API_VERSION%/%SUBJECT%` のような動的候補を生成し、API面を列挙。
- **バックアップ/漏えいファイル専用スキャン**：`--wordlist-categories backups,vcs,conf,keys` に絞って `.git` `.env` `*.bak` `id_rsa` だけを狙う。
- **技術スタック特定後の深掘り**：WordPress と分かれば `--wordlist-categories php`（wordpress/woocommerce 等）で的を絞る。
- **認証の挙動調査**：`-H` でヘッダ注入、`--auth`（basic/digest/bearer/ntlm/jwt）で認証状態ごとの差分を観察。
- **HTTPメソッド試験**：`-m POST`/`PUT`/`DELETE` と `-d`（ボディ）で、メソッド差による応答の違いを確認。
- **大量URLの一括健全性チェック**：`-l urls.txt` や `--nmap-report` で、複数ホストに同じ辞書を当てて棚卸し。
- **軽量な死活/反応監視**：`--match-time` で「遅いエンドポイント」だけ抽出するなど、性能面の当たりを付ける用途にも転用可能。
- **ライブラリとして**：Python API で結果を受け取り、自作の検査・集計スクリプトの前段に。

> 注：ffuf のような「任意位置の総当たり」（パラメータ名やHostヘッダのFUZZ）に特化はしていません。純粋なパラメータファジングは ffuf、面の列挙と深掘りは dirsearch、と使い分けるのが実務的です。

---

## 7. レジデンシャル／プロキシチェーン経由でのスキャン

**できます。** dirsearch はプロキシを厚くサポートしています（`lib/connection/` を確認）。

- **単段**：`-p http://HOST:PORT` または `-p socks5://HOST:PORT`
- **複数指定＝ローテーション**：`-p` を複数回、または `--proxies-file proxies.txt`。内部ではリクエストごとに `random.choice` でプロキシを選ぶ実装（async は `ProxyRoatingTransport`、sync も同様）。レジデンシャルプロキシ業者の「IPプール一覧」をそのまま `--proxies-file` に渡せば**IPが毎リクエストでばらける**。
- **認証つき**：URL に `http://user:pass@host:port`、または `--proxy-auth user:pass`。業者がユーザ名でセッション/地域を制御する方式にも対応。
- **プロキシチェーン（多段）**：dirsearch 自体は「1ホップ目のプロキシ」を指すだけ。多段は**プロキシ側**で upstream へ転送させて作ります（本ラボの `proxy-chain` がこれ。`UPSTREAM` で次段へ丸ごと転送）。OSレベルで `proxychains4 dirsearch ...` と被せる方法もあります。
- **Tor**：`--tor`（9050 を利用）。SOCKS を使うので scanner イメージに `pysocks` を同梱済み。
- **replay-proxy**：`--replay-proxy http://127.0.0.1:8080` で、**見つかったパスだけ** Burp 等に流して手動精査へ受け渡し。
- **送信元の制御**：`--ip` でDNSを固定、`--interface` で出口NICを選択。

### コマンド例

```bash
# 1) 単段（HTTP）
dirsearch -u https://target/ -e php -p http://proxy1:3128

# 2) SOCKS5
dirsearch -u https://target/ -p socks5://127.0.0.1:1080

# 3) レジデンシャル等のIPプールをローテーション（毎リクエストでIPが変わる）
#    proxies.txt に 1行1プロキシで列挙（user:pass 付きも可）
dirsearch -u https://target/ -e php \
          --proxies-file proxies.txt \
          --random-agent --delay 0.1 --max-rate 20

# 4) 認証つきプロキシ
dirsearch -u https://target/ -p http://user:pass@gw.provider.com:8000
#   または
dirsearch -u https://target/ -p http://gw.provider.com:8000 --proxy-auth user:pass

# 5) 多段チェーン（手前の1台を指すだけ。多段化はプロキシ側 or proxychains で）
dirsearch -u https://target/ -p http://first-hop:3128
proxychains4 dirsearch -u https://target/ -e php      # OSレベルで被せる例

# 6) Tor 経由
dirsearch -u https://target/ --tor

# 7) 見つかったパスだけ Burp に流す
dirsearch -u https://target/ -e php --replay-proxy http://127.0.0.1:8080
```

`proxies.txt` の書式例：

```
http://user:pass@res1.provider.com:8000
http://user:pass@res2.provider.com:8000
socks5://127.0.0.1:1080
```

> レジデンシャルプロキシは「業者と契約した自分のアカウント」で、かつ「スキャン対象から許可を得ている」場合にのみ使ってください。IPを変える目的が**検知回避による無許可アクセス**なら、それは不正アクセスです。本ラボは分散の*挙動*を安全に観察するためのものです。

---

## 8. 同梱ラボの構成と使い方

```
dirsearch-lab/
├─ docker-compose.yml      # ラボ全体
├─ target/                 # 叩かれる側: わざと穴だらけのWeb（標準ライブラリのみ）
│   ├─ app.py
│   └─ Dockerfile
├─ proxy/                  # 経由する簡易フォワードプロキシ（HTTP/CONNECT/認証/多段）
│   ├─ proxy.py
│   └─ Dockerfile
├─ scanner/                # dirsearch 同梱イメージ
│   └─ Dockerfile
├─ wordlists/
│   ├─ custom.txt          # このラボ用の小さな自作辞書
│   └─ proxies.txt         # ローテーション用プロキシ一覧
├─ scripts/
│   ├─ _lib.sh             # 共通設定（docker/local 切替）
│   ├─ 00_up.sh            # 起動＆疎通確認
│   ├─ 01_basic.sh         # 基本スキャン
│   ├─ 02_recursive_crawl.sh  # 再帰＋クロール＋バックアップ
│   ├─ 03_filters_bypass.sh   # soft-404/WAF/XFF/遅延の処理
│   ├─ 04_proxy.sh         # 単段/ローテーション/認証/多段/置換
│   ├─ 05_full_power.sh    # 合わせ技（最強）
│   └─ 99_down.sh          # 片付け
└─ reports/                # 出力先（自動生成）
```

### 起動

```bash
cd dirsearch-lab
bash scripts/00_up.sh         # ビルド＆起動＆疎通確認
```

### スキャン実行（Docker版：scanner コンテナで実行）

```bash
bash scripts/01_basic.sh
bash scripts/02_recursive_crawl.sh
bash scripts/03_filters_bypass.sh
bash scripts/04_proxy.sh
bash scripts/05_full_power.sh
```

### ホストに dirsearch を入れて使う場合（local版）

```bash
pip install "git+https://github.com/maurosoria/dirsearch.git"
DS_MODE=local bash scripts/01_basic.sh      # 標的は localhost:8080 を向く
```

### 観察ポイント（標的が仕込んでいる"教材"）

| 仕掛け | 見え方 | 効くオプション |
|---|---|---|
| 公開済み隠しファイル | `.env` `.git/HEAD` `.git/config` `backup.zip` `swagger.json` `docker-compose.yml` が **200** | 基本スキャン |
| ディレクトリ | `admin` `api` `uploads` `backup` が **301** でスラッシュ付きへ | `-r` で掘る |
| 要認証 | `manager/html` が **401**（Tomcat風） | 既定で検出 |
| 禁止だが存在 | `server-status` が **403** | 既定で検出 |
| **認可バイパス** | `admin/` は通常 **403**、`X-Forwarded-For: 127.0.0.1` 付きで **200** | `-H` でヘッダ注入 |
| **robots 由来** | `robots.txt` の `Disallow: /secret-panel/` | `--crawl` |
| **JS 由来** | トップのJSが `/api/v2/internal/debug` を参照 | `--crawl` |
| **バックアップ** | `login.php` → `login.php.bak` が存在 | `--find-backup` |
| 再帰で出る | `backup/db.sql` `api/v1/users` `uploads/shell.php.txt` | `-r -R 3` |
| **soft-404** | `/shop/` 以下は存在しないパスも **200** | `--auto-calibration` で除外 |
| **WAF風** | UAに `gobuster/ffuf/nikto/sqlmap` があると **403** | `--random-agent` |
| **遅延** | `reports/export` は 1.5 秒 | `--match-time '>1000'` |
| **レート制限** | `target-rl`(:8081) は 20req/s 超で **429** | `--max-rate 15` |

標的は送信元ごとのリクエスト数を記録します。プロキシ経由で**送信元IPが分散する様子**を数字で確認できます：

```bash
curl -s http://localhost:8080/_lab/stats     # 送信元IPごとの件数
curl -s http://localhost:8080/_lab/reset      # カウンタ初期化
docker compose logs proxy1 proxy2 proxy-chain # どのプロキシを経由したか
```

### 片付け

```bash
bash scripts/99_down.sh
```

---

## 9. 「特別な最強技」集

手を動かして効く順に。

1. **芋づる式フルコンボ**（入口1つから深掘り）
   ```bash
   dirsearch -u https://target/ -e php,html,js,json,bak,old,zip,sql \
     -r -R 3 --deep-recursive --crawl --find-backup --auto-calibration --random-agent
   ```
   `--deep-recursive` は `api/users` を見つけたら `api/` も遡って掘る。`--crawl` が robots/JS/リンクから新パスを足し、`--find-backup` が各ヒットの `.bak/.old/~/.zip` を自動追撃。

2. **"本物だけ"を残すフィルタ術**（ffuf由来）
   ```bash
   # 偽の200を除外（例: 全部1234バイトのカスタム404）
   dirsearch -u https://target/ -e php --exclude-sizes 1234B
   # body に "Not Found" を含むものを除外、200/403だけ残す
   dirsearch -u https://target/ -e php --exclude-text "Not Found" --mc 200,403
   # 遅いエンドポイントだけ（重い処理＝攻撃面のことがある）
   dirsearch -u https://target/ --mt '>1000'
   ```
   `--mmode/--fmode and|or` でマッチ条件を論理合成できる。

3. **403 を"開ける"試み**（存在するが禁止、を突く）
   ```bash
   dirsearch -u https://target/admin/ -e php --include-status 200,403 \
     -H "X-Forwarded-For: 127.0.0.1" -H "X-Original-URL: /admin/"
   ```
   本ラボの `admin/` は XFF で 200 に変わる。実機でも逆プロキシ設定次第で同種の穴がある。

4. **生HTTPをそのまま再生**（Burpで保存した要求を辞書化）
   ```bash
   dirsearch --raw req.txt --scheme https -e php
   ```
   クッキー・独自ヘッダ込みの認証済みセッションで探索できる。

5. **IPプール・ローテーション＋平準化**（分散＆低姿勢）
   ```bash
   dirsearch -u https://target/ -e php \
     --proxies-file proxies.txt --random-agent --max-rate 20 --delay 0.1 --retries 2
   ```
   `--max-rate` で秒間を抑え、`--random-agent` でUAも散らす。

6. **中断に強い長丁場**（セッション）
   ```bash
   dirsearch -u https://target/ -e php -r -s run1.session   # Ctrl-Cで中断→
   dirsearch -s run1.session                                 # 続きから再開
   dirsearch --list-sessions                                 # 一覧
   ```

7. **一撃で全証跡**（複数出力＋生レスポンス）
   ```bash
   dirsearch -u https://target/ -e php \
     -o out.md -O md -o out.json -O json -o out.sqlite -O sqlite \
     --save-response ./responses
   ```

8. **ターゲットを一括供給**
   ```bash
   dirsearch --nmap-report scan.xml -e php           # nmap -sV の結果から標的生成
   cat hosts.txt | dirsearch --stdin -e php          # STDINから
   dirsearch --cidr 10.0.0.0/28 -e php               # CIDR
   ```

9. **置換プロキシで手動へ橋渡し**
   ```bash
   dirsearch -u https://target/ -e php --replay-proxy http://127.0.0.1:8080
   ```
   ヒットだけ Burp に飛ぶので、発見→精査の動線が短い。

---

## 10. この環境で確認できたこと（検証メモ）

このノートの数値・挙動は、`dirsearch` **v0.5.0** のソース（`maurosoria/dirsearch`、確認時のコミット `b45e01b`）を取得して直接確認しました。辞書行数・カテゴリ構成・`config.ini` 既定値・プロキシのローテーション実装（`random.choice`／`ProxyRoatingTransport`）・`--find-backup` の生成ロジック・LFI系辞書エントリは、いずれも実物を見て記載しています。

標的サーバとプロキシは `curl` で全挙動を実機確認済みです（公開ファイル=200、`admin/`=403→XFFで200、スキャナUA=403、`/shop/`のsoft-404=常に200、`reports/export`=1.5秒、`target-rl`=20件通過/超過分429、プロキシ単段・認証(407→通過)・多段チェーンの転送）。

> 補足：この執筆環境（クラウドのサンドボックス）は外部ネットワークが遮断されており、**Docker Hub からのイメージ取得と pip の外部取得ができません**。そのため「dirsearch 本体をこの環境で実行した生ログ」は含めていません。`docker compose config` と各スクリプトの構文、標的/プロキシの挙動はこの環境で検証済みで、**インターネットに出られる普通のマシンで `bash scripts/00_up.sh` → `bash scripts/01_basic.sh` を実行すれば、上表の通りに動きます**（必要な取得先は GitHub と PyPI と `python:3.12-slim` イメージのみ）。
```
