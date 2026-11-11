# CSRF：対策のないメールアドレス変更機能へのクロスサイトリクエスト

## 1. 検証概要

| 項目 | 内容 |
|---|---|
| 学習プラットフォーム | PortSwigger Web Security Academy |
| ラボ名 | CSRF vulnerability with no defenses |
| 難易度 | Apprentice |
| 分野 | Cross-Site Request Forgery（CSRF） |
| 検証日 | 2026年11月11日 |
| 対象機能 | メールアドレス変更（`POST /my-account/change-email`） |
| 使用ツール | Burp Suite、Exploit Server、ブラウザ |
| 検証結果 | LAB Solved |
| 理解度チェック | 5/5問正解（100%、全問「自信あり」） |

### 検証目的

ログイン済みの利用者が操作できるメールアドレス変更機能にCSRF対策がない場合、第三者が用意したWebページからリクエストを自動送信させ、本人の意図しない変更を実行できるか検証する。

あわせて、HTMLフォームの`value`属性と`application/x-www-form-urlencoded`のエンコード処理の違いを確認する。

**対象は許可されたPortSwiggerの学習用ラボのみ。**

---

## 2. 実務上の攻撃シナリオ

### 2.1 想定するシステムと攻撃者

会員制サービスで、ログイン中のユーザーが`Update email`から登録メールアドレスを変更できると想定する。

攻撃者は対象サービスのアカウントやパスワードを知らなくても、悪意のあるHTMLページを用意して被害者に閲覧させることができる。

### 2.2 攻撃から被害までの流れ

1. 攻撃者はメールアドレス変更先`POST /my-account/change-email`と必要なパラメータを把握する。
2. 対象サービスはセッションに基づいて操作を受け付ける一方、CSRFトークン等でリクエストの正当性を検証していない。
3. 攻撃者が、攻撃者の指定したメールアドレスを送信するフォームを別サイトに設置する。
4. 被害者が対象サービスにログイン中にそのページを開き、ブラウザが有効な認証Cookieをクロスサイトのリクエストへ送る条件が揃うと、スクリプトでフォームが自動送信される。
5. サーバーがリクエストの出所を適切に検証しなければ、本人の意図しないメールアドレス変更が成立する。
6. 実務では、変更後メールアドレスがアカウント復旧の送信先になる場合などに、アカウント乗っ取りへ発展するおそれがある。

**この攻撃用HTMLでは、被害者がページを開いた後、フォームの送信ボタンを押す必要はない。** ただし、攻撃ページを開く契機やCookieの送信条件は必要である。

### 2.3 想定被害と根本原因

| 観点 | 内容 |
|---|---|
| 想定被害 | 登録メールアドレスの不正変更、通知・パスワード再設定先の乗っ取り、状況によってはアカウントの不正利用 |
| 根本原因 | 重要な状態変更リクエストをサーバーが受け付ける際、適切なCSRF防御を実施していない |
| 成立条件 | 被害者の認証状態が有効で、ブラウザから対象サイトへ認証情報が送信されること |
| 防止策 | CSRFトークンの発行・検証、Origin/Refererの検証、SameSite Cookieの適切な設定 |
| 検知策 | 重要操作の監査ログ、異常なOrigin/Refererや短時間の設定変更の監視 |

**上記の被害は実務上の想定であり、今回のラボでパスワード再設定やアカウント乗っ取りまで実証したものではない。**

---

## 3. 技術的背景

### 3.1 CSRFの仕組み

CSRF（Cross-Site Request Forgery）は、被害者のブラウザが持つ認証状態を利用し、第三者サイトから本人の意図しない状態変更リクエストを送信させる問題。

「誰のアカウントか」をセッションCookieで識別するだけでは、「その操作を本人が意図して行ったか」を保証できない。

### 3.2 変更先リクエストの問題点

今回観測した`POST /my-account/change-email`のリクエスト本文には、`email`パラメータのみがあり、CSRFトークン等の対策用パラメータが存在しなかった。

ただし、**本文にトークンがないことだけであらゆる環境のCSRF脆弱性を断定できるわけではない**。HTTPヘッダー、Cookie設定、サーバー側のOrigin検証など他の防御も考慮する必要がある。今回は、実際に第三者ページから変更を成立させSolvedを確認したことが重要な実証結果となる。

### 3.3 二重エンコードの原因

HTMLフォームの`value`属性には、通常、エンコード前の値を記述する。ブラウザがフォームを`application/x-www-form-urlencoded`形式で送信する際に適切にエンコードする。

最初に`hacked%40evil-user.net`をHTML属性へ直接記述すると、`%`が`%25`としてエンコードされる。

```text
HTMLのvalue             : hacked%40evil-user.net
送信されたパラメータ値 : hacked%2540evil-user.net
通常のフォーム復号後   : hacked%40evil-user.net
```

その結果、メールアドレスの`@`として扱われず変更が失敗した。

修正後は`value="hacked@evil-user.net"`を指定。送信時にはブラウザが`@`を`%40`へエンコードするため、サーバー側で元のメールアドレスとして解釈できる。

---

## 4. 検証手順

### STEP 1：通常のメールアドレス変更を確認

`wiener` / `peter` でログインし、`Update email`フォームを一度送信した。

Burp Suiteで`POST /my-account/change-email`を確認。リクエスト本文は`email=<値>`のみで、CSRFトークンなどの対策パラメータがないことを確認した。

### STEP 2：Exploit Serverに攻撃用フォームを設置

Exploit Serverに、対象の変更先へPOSTを送信するHTMLフォームを作成した。

隠し入力`email`に変更先メールアドレスを設定し、JavaScriptの`document.forms[0].submit()`を使用してページ読み込み時に自動送信するようにした。

### STEP 3：最初の送信失敗と原因の特定

初回はHTMLフォームの`value`属性へエンコード済みの`hacked%40evil-user.net`を直接記述していた。

ブラウザのフォーム送信処理でさらにエンコードされ、`email=hacked%2540evil-user.net`となったため、メールアドレス変更が成立しなかった。

### STEP 4：フォームの修正

`value`属性をエンコード前の`hacked@evil-user.net`に修正した。

これにより、フォーム送信時の値が正常に扱われるようになった。

### STEP 5：自分のアカウントで確認

Exploit Serverの`View exploit`で自分自身に対して動作を確認し、メールアドレスが変更されたことを確認した。

### STEP 6：被害者に配信しSolvedを確認

攻撃フォーム内のメールアドレスを自分の検証用とは別の値に変更し、`Deliver to victim`を実行した。

**LAB Solvedを確認した。** 被害者側の実際の画面表示や最終メールアドレスの値は提供されていないため、ここでは断定しない。

---

## 5. 実証HTTPと攻撃用HTML

**重要：以下のHTTPリクエストとHTMLは、共有された手順・パラメータ・観測結果をもとにした説明用の再構成例であり、生のBurpリクエスト全文またはExploit Server上のHTML原本の転載ではない。** 実際のドメイン、Cookie、未確認ヘッダー、送信先の最終的なメールアドレスは掲載しない。

### 5.1 メールアドレス変更リクエストの説明例

```http
POST /my-account/change-email HTTP/1.1
Host: <lab-id>.web-security-academy.net
Content-Type: application/x-www-form-urlencoded

email=hacked%40evil-user.net
```

`email=hacked%40evil-user.net` は、フォームへ生の`@`を記載したときのURLエンコード済み送信例。`HTTP/1.1`および`Content-Type`は説明用の表記で、キャプチャした値ではない。

**実測から確認できる点：** 本文の入力パラメータは`email`のみで、CSRF対策パラメータがなく、修正後の攻撃HTMLからメールアドレス変更が成立した。

### 5.2 初回失敗時の送信値（観測結果）

HTMLフォーム内で次のように記述していた。

```html
<input type="hidden" name="email" value="hacked%40evil-user.net">
```

この場合に観測された送信値：

```text
email=hacked%2540evil-user.net
```

**結果：メールアドレス変更は失敗。**

### 5.3 修正後の攻撃用HTML（再構成例）

```html
<!DOCTYPE html>
<html lang="ja">
<head>
  <meta charset="UTF-8">
  <title>CSRF lab demonstration</title>
</head>
<body>
  <form method="POST"
        action="https://<lab-id>.web-security-academy.net/my-account/change-email">
    <input type="hidden" name="email" value="hacked@evil-user.net">
  </form>
  <script>
    document.forms[0].submit();
  </script>
</body>
</html>
```

※ `<lab-id>` は伏せ字。HTMLは検証内容を説明するために再構成したもの。`hacked@evil-user.net`は修正の説明用であり、最後の`Deliver to victim`時には別の値に変更している。

**結果：** `View exploit` で自分のアカウントのメールアドレス変更が成立。被害者配信後にLAB Solved。

### 5.4 レスポンスについて

HTTPステータスコード、レスポンスヘッダー、レスポンス本文、Cookieの属性は生の記録が提供されていないため記載しない。画面上の変更確認とSolvedを結果の証拠として扱う。

---

## 6. 検証結果と根本原因

| 検証項目 | 確認した事実 | 意味 |
|---|---|---|
| 通常の変更 | `Update email`から変更リクエストを送信 | 対象機能を特定 |
| リクエスト本文 | `email`のみ、CSRFトークン等のパラメータなし | 本文によるトークン検証がない |
| 初回HTML | `%40`を直接指定し`%2540`で送信 | フォームの二重エンコードを確認 |
| 修正後HTML | 生の`@`を指定し`View exploit`を実行 | 自分のメールアドレス変更に成功 |
| 被害者への配信 | 別のメールアドレス値で`Deliver to victim` | LAB Solved |

**根本原因：** 重要な状態変更を受け付ける際に、正規操作を証明するCSRFトークンや同等の防御が適切に機能していないこと。結果として、第三者サイトからのフォームPOSTをログイン中のユーザーの操作として実行できた。

---

## 7. 推奨防止策と検知策

### 7.1 CSRFトークンの発行と検証

推測困難なCSRFトークンをフォーム等へ埋め込み、サーバー側でセッションとの対応を検証する。不一致・欠落したリクエストを拒否する。

### 7.2 Origin / Refererの検証

重要操作では`Origin`や、必要な場合に`Referer`を適切に検証し、許可していないオリジンからのリクエストを拒否する。ヘッダーが欠落した場合の安全な扱いも設計する。

### 7.3 SameSite Cookieと追加防御

セッションCookieへ適切な`SameSite`属性を設定し、クロスサイトのリクエストへのCookie送信を制限する。これは重要な補助防御だが、認証状態や脅威モデルに応じてCSRFトークン等と併用する。

必要に応じてFetch Metadataヘッダーの検証や、メールアドレス変更前の再認証・変更確認を組み合わせる。

### 7.4 監視・検知

メールアドレス変更の監査ログを記録し、変更時刻、ユーザー、送信元、異常な`Origin`、短期間の連続変更などを監視する。重要設定の変更通知も有効。

---

## 8. 実務上の想定被害

メールアドレスが本人の意思に反して変更されると、通知の受信先やアカウント復旧の送信先が攻撃者へ移る可能性がある。

サービスの仕様次第ではアカウント乗っ取りにつながるが、**今回のラボで実証したのはメールアドレス変更とLAB Solvedまで**。他の不正操作は未検証である。

同様の問題は、パスワード変更、送金先登録・送金、配送先変更、権限・セキュリティ設定の変更など、ユーザーの状態を変更する重要操作にも起こり得る。

---

## 9. 未検証事項と制約

- 生のHTTPリクエスト・レスポンス全文、Cookie属性、ヘッダー、実際のHTTPバージョン
- サーバー側のCSRF判定処理、Origin/Refererの扱い
- SameSite Cookieの有無とクロスサイト送信条件の詳細
- `Deliver to victim`後の被害者ブラウザ画面、最終メールアドレスの具体値
- メールアドレス変更時の通知・再認証・確認メールの有無
- パスワード変更や送金等の別機能でのCSRF成立
- 実在するサービスでの影響

※ 検証で確認できた事実、説明用に再構成したHTTP/HTML、実務上の想定を分けて記録した。

---

## 10. 理解度チェック

| 問題 | テーマ | 回答 | 正解 | 判定 | 自信度 |
|---|---|---|---|---|---|
| Q1 | CSRFの根本原因 | C | C（CSRF検証不備） | 正解 | 自信あり |
| Q2 | 攻撃成立時の被害者側の条件・クリックの要否 | B | B（ログイン中、必要なCookie送信、自動フォームなので送信ボタン不要） | 正解 | 自信あり |
| Q3 | `%40` の二重エンコード | B | B（`%` が`%25`となり`%2540`になる） | 正解 | 自信あり |
| Q4 | CSRFの代表的な対策2つ以上 | C | C（トークン検証、Origin/Referer検証、SameSite Cookie） | 正解 | 自信あり |
| Q5 | CSRFのリスクが高い機能 | C | C（メール・パスワード変更、送金、設定変更等） | 正解 | 自信あり |

**5/5問正解（100%）。全問「自信あり」。**

### 理解の整理

- CSRFはブラウザが持つ認証状態を利用する。被害者のパスワードを攻撃者が知る必要はない。
- 自動送信フォームは、攻撃ページを開くことで送信される。明示的な送信ボタンのクリックは不要。
- HTMLの`value`には通常エンコード前のメールアドレスを記述し、フォーム送信時のエンコードはブラウザに任せる。
- CSRFトークンなどの適切なサーバー側検証とCookie制御を組み合わせる。

---

## 11. 振り返り

今回のラボでは、Burp Suiteで対象のPOSTリクエストを確認したうえで、Exploit Serverに攻撃フォームを作成した。

最初はエンコード済みの`%40`をHTML属性に入力したため`%2540`に変化して失敗したが、送信処理の原因を特定して生の`@`に修正し、`View exploit`で成功を確認できた。最後に`Deliver to victim`でLAB Solvedとなった。

**今後の課題：** 生のHTTP通信、認証Cookieの`SameSite`属性、`Origin`・`Referer`の有無を記録し、成立条件をより正確に評価する。

---

## 12. 参考資料

- [PortSwigger：CSRF vulnerability with no defenses](https://portswigger.net/web-security/csrf/lab-no-defenses)
- [PortSwigger：Cross-site request forgery](https://portswigger.net/web-security/csrf)
- [OWASP：Cross-Site Request Forgery Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html)

※ 許可された学習用ラボでの検証記録であり、実環境に対する無許可の攻撃や診断実績を示すものではない。