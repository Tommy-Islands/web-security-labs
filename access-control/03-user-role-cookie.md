# Broken Access Control：Cookie改変による管理者権限の偽装

## 1. 検証概要

| 項目 | 内容 |
|---|---|
| 学習プラットフォーム | PortSwigger Web Security Academy |
| ラボ名 | User role controlled by request parameter |
| 難易度 | Apprentice |
| 脆弱性 | Broken Access Control（アクセス制御の不備） |
| 分類 | Vertical privilege escalation（垂直方向の権限昇格） |
| 検証環境 | VirtualBox / Parrot Security |
| 使用ツール | Burp Suite Community Edition / Burp専用ブラウザ |
| 使用機能 | Proxy / Request・Response Intercept |
| 検証結果 | Solved |
| 理解度チェック | 5問中4問正解（80%） |

### 検証目的

ログイン時に設定されるCookieを改変することで、一般ユーザーが管理者権限を必要とする機能にアクセスできるか検証する。

本検証では、PortSwigger公式の解法に従い、**ログイン時のHTTPレスポンスをInterceptして `Set-Cookie` ヘッダーを書き換える方法**を使用した。

## 2. 攻撃シナリオと背景

### 2.1 想定システム

あるWebサービスには、一般ユーザー向けのアカウントページと、管理者向けのユーザー管理画面が存在する。

管理者はユーザー一覧の閲覧やアカウントの削除を実行できる。

一般ユーザーは、これらの管理者機能を利用できない設計である。

しかし、システムは管理者かどうかを判定する際に、ブラウザから送信される `Admin` Cookieを使用している。

### 2.2 攻撃者の目的

一般ユーザーとしてログインした状態で、通信内容を改変して管理者として扱われる状態を作り、本来利用できない管理者機能を実行する。

今回のラボでは、ユーザー `carlos` の削除が目標となる。

### 2.3 攻撃シナリオ

1. 攻撃者が一般ユーザーとしてログインする。
2. ログイン時にサーバーから返されるHTTPレスポンスをInterceptする。
3. `Set-Cookie: Admin=false` を `Admin=true` に変更する。
4. 改変したレスポンスをブラウザへ渡す。
5. ブラウザが `Admin=true` を保存し、後続のリクエストで送信する。
6. サーバーがCookieの値を信用する場合、管理者機能へのアクセスが許可される。

## 3. 検証手順と実際の結果

### STEP 1：管理者画面へのアクセス確認

一般ユーザーの状態で、以下のパスにアクセスした。

```http
GET /admin HTTP/2
```

**確認結果**

以下のメッセージが表示された。

```text
Admin interface only available if logged in as an administrator
```

この段階では、通常のブラウザ操作による管理者画面へのアクセスは拒否された。

### STEP 2：ログイン画面への移動

ラボのログイン画面を開いた。

PortSwiggerが提供する学習用アカウントを使用した。

```text
Username: wiener
Password: peter
```

ログインボタンを押す前に、Burp SuiteのInterceptを有効化した。

### STEP 3：ログインリクエストのIntercept

Burp Suiteで以下の画面を開いた。

```text
Proxy → Intercept
```

InterceptをONにした状態でログインボタンをクリックした。

次のリクエストが一時停止した。

```http
POST /login HTTP/2
```

リクエスト本文には、次のようなフォームパラメータが含まれていた。

```text
csrf=[省略]&username=wiener&password=peter
```

ここではログインリクエスト自体は変更していない。

### STEP 4：対象レスポンスのIntercept設定

停止中の `POST /login` リクエストを右クリックし、次の項目を選択した。

```text
Do intercept → Response to this request
```

これは、現在のログインリクエストに対応するサーバーからのレスポンスもInterceptするための操作である。

設定後、`Forward` をクリックしてログインリクエストをサーバーへ送信した。

### STEP 5：ログインレスポンスの書き換え

次のHTTPレスポンスがInterceptされた。

**変更前**

```http
HTTP/2 302 Found
Location: /my-account?id=wiener
Set-Cookie: Admin=false; Secure; HttpOnly
Set-Cookie: session=[省略]; Secure; HttpOnly; SameSite=None
```

ここで確認できたのは、サーバーがログイン時に `Admin=false` というCookieをブラウザへ設定しようとしていることだった。

そこで、`Admin` Cookieの値のみを変更した。

**変更後**

```http
HTTP/2 302 Found
Location: /my-account?id=wiener
Set-Cookie: Admin=true; Secure; HttpOnly
Set-Cookie: session=[省略]; Secure; HttpOnly; SameSite=None
```

変更箇所：

```text
Admin=false → Admin=true
```

`session` の値や、`Secure`・`HttpOnly` 属性は変更していない。

変更後、Burp Suiteの `Forward` をクリックしてレスポンスをブラウザへ渡した。

### STEP 6：Cookie変更の反映確認

レスポンスをForwardした後、次のリクエストがInterceptされた。

```http
GET /my-account?id=wiener HTTP/2
Cookie: Admin=true; session=[省略]
```

**確認結果**

ブラウザから送信されたCookieの値が、`Admin=true` に変化していることを確認した。

この結果から、ログインレスポンスへの改変がブラウザに反映されたことが分かる。

### STEP 7：管理者画面へのアクセス

InterceptをOFFにして、ブラウザでアカウントページを表示した。

ナビゲーションに、通常の一般ユーザー画面では表示されていなかった `Admin panel` のリンクが出現した。

リンクをクリックすると、管理者画面が表示された。

画面には次のユーザー一覧が存在した。

```text
Users

wiener - Delete
carlos - Delete
```

**確認結果**

一般ユーザー `wiener` のログイン状態で、管理者専用のユーザー管理画面にアクセスできた。

### STEP 8：ユーザー削除機能の検証

管理者画面で、`carlos` の右側にある `Delete` をクリックした。

**確認結果**

次のメッセージが表示された。

```text
User deleted successfully!
```

さらに、

- ユーザー一覧から `carlos` が消えた
- `wiener` のみが一覧に残った
- ラボのステータスが `Solved` になった

ことを確認した。

以上より、管理者権限のない利用者がCookieを改変することで、管理者画面へのアクセスとユーザー削除を実行できることを実証した。

## 4. 検証結果と証拠

| 検証項目 | 実際の結果 |
|---|---|
| 通常の `/admin` アクセス | 拒否された |
| `POST /login` のIntercept | 成功 |
| ログインResponseのIntercept | 成功 |
| `Set-Cookie: Admin=false` の確認 | 成功 |
| `Admin=true` への改変 | 成功 |
| 後続Requestの `Cookie: Admin=true` | 確認 |
| Admin panelリンクの表示 | 成功 |
| 管理者画面へのアクセス | 成功 |
| `carlos` の削除 | 成功 |
| ラボのクリア | Solved |

### 証拠として記録する画面

- ログイン時の `POST /login` リクエスト
- `Set-Cookie: Admin=false` を含むレスポンス
- `Set-Cookie: Admin=true` に改変したレスポンス
- 後続リクエストの `Cookie: Admin=true`
- 管理者画面とユーザー一覧
- `User deleted successfully!` および `Solved` の表示

※ スクリーンショットを公開する場合は、セッションIDやCSRFトークンなどの値を伏せる。

## 5. 根本原因と技術的な考察

### 5.1 根本原因

本ラボの根本原因は、**クライアント側で変更可能なCookieの値を、サーバーが管理者権限の判定に信用していたこと**である。

例えば、以下のような処理は危険である。

```javascript
// 問題のある実装の概念例
// 実際のラボのソースコードではない
if (request.cookies.Admin === "true") {
  allowAdminAccess();
}
```

利用者自身が送信する `Admin=true` という申告だけで管理者機能を許可してしまう。

今回、一般ユーザーがCookieを改変して管理者機能を実行できたことから、権限判定が不適切だったことが確認できる。

ただし、サーバー内部のソースコードは確認していないため、実際の実装方法までは断定しない。

### 5.2 垂直方向の権限昇格

今回の脆弱性は、垂直方向のアクセス制御の不備に分類される。

通常の一般ユーザーが、本来は上位権限を持つ管理者だけに許可される操作を実行できたためである。

### 5.3 CookieとSet-Cookieの違い

| ヘッダー | 送信方向 | 役割 |
|---|---|---|
| `Set-Cookie` | サーバー → ブラウザ | Cookieを保存するようブラウザへ指示する |
| `Cookie` | ブラウザ → サーバー | 保存されたCookieをサーバーに送信する |

今回の公式手順では、**Responseの `Set-Cookie` を書き換えた**。

その結果、後続のRequestで `Cookie: Admin=true` が送信されるようになった。

### 5.4 SecureとHttpOnlyの役割

今回のCookieには、次の属性が含まれていた。

```http
Set-Cookie: Admin=false; Secure; HttpOnly
```

**Secure**

Cookieを原則としてHTTPS通信でのみ送信するよう制限する属性。

**HttpOnly**

JavaScriptからCookieを読み書きできないようにする属性。

どちらもCookieの保護に役立つが、権限情報をサーバーが安全に検証する仕組みではない。

したがって、これらの属性が設定されていても、クライアントが変更できるCookieを信用した認可処理の不備は防げない。

## 6. 想定される被害とリスク

### 6.1 今回実証した影響

- 一般ユーザーによる管理者画面へのアクセス
- 管理者向けユーザー一覧の閲覧
- 指定ユーザー `carlos` の削除

### 6.2 実際のサービスで想定される被害

同様の脆弱性が実際のWebサービスに存在する場合、管理者機能の内容に応じて次のような被害が発生する可能性がある。

- 顧客アカウントの不正削除
- 顧客情報や内部情報の不正閲覧
- 管理設定やデータの不正変更
- サービスの利用妨害
- 業務停止や信用低下

### 6.3 検証範囲と限界

今回実証したのは、学習用ラボにおける管理者画面へのアクセスと、指定ユーザーの削除である。

以下は未検証である。

- すべての管理者機能の利用
- データベース全体へのアクセス
- 管理者権限の永続的な付与
- 他のユーザーアカウントの変更
- 実際の企業システムへの影響

また、今回の操作によってサーバー内部のユーザー権限情報そのものが変更されたことは確認していない。

## 7. 推奨対策

### 7.1 クライアント側の権限フラグを信用しない

`Admin=true` のような、ブラウザから変更可能な値だけを管理者権限の根拠にしない。

### 7.2 サーバー側で認可チェックを実装する

認証済みのセッションと、サーバー側で管理する信頼できる権限情報を照合してアクセス可否を判断する。

以下は説明用の擬似コードである。

```javascript
const currentUser = getAuthenticatedUser(request);

if (!currentUser) {
  return 401;
}

const role = getRoleFromTrustedServerData(currentUser.id);

if (role !== "admin") {
  return 403;
}

return executeAdminOperation();
```

### 7.3 重要な操作ごとに権限を確認する

管理者画面の表示だけでなく、ユーザー削除などの操作を実行する際にも認可チェックを行う。

### 7.4 最小権限の原則を適用する

各利用者には、その役割に必要な権限だけを付与する。

### 7.5 Cookie改変を想定したセキュリティテスト

Cookieの値を変更したリクエストを送信しても、利用者の権限が不正に変化しないことを確認する。

## 8. 理解度チェック

本ラボの攻略後、全5問の理解度チェックを実施した。

### 8.1 結果

| 問題 | テーマ | 回答 | 正誤 | 自信度 |
|---|---|---|---|---|
| Q1 | CookieとSet-Cookieの仕組み | B | 正解 | 自信あり |
| Q2 | SecureとHttpOnlyの役割 | B | 不正解（正解C） | 自信あり |
| Q3 | RequestとResponseのIntercept操作 | B | 正解 | 自信なし |
| Q4 | 権限昇格の根本原因 | C | 正解 | 自信あり |
| Q5 | 実証結果と検証範囲の区別 | D | 正解 | 自信なし |

**最終結果：5問中4問正解（80%）**

### 8.2 理解できたこと

- `Set-Cookie` を変更すると、ブラウザが保存するCookieに影響を与えられる
- クライアント側で変更可能な権限フラグを信用することは危険である
- 管理者機能の実行可否はサーバー側で確認する必要がある
- 検証で確認した事実と、想定される被害は区別する必要がある

### 8.3 誤答から判明した課題

**Q2：SecureとHttpOnlyの役割**

`HttpOnly` は管理者用Cookieには適用されないと誤解していた。

実際には、一般ユーザー用・管理者用を問わずCookieに設定でき、JavaScriptからのアクセスを制限する属性である。

また、`Secure` はCookieの送信条件を制限するものであり、アクセス権限の適切な検証を保証しない。

この点は自信ありで誤答したため、優先的に復習する。

### 8.4 正解したが自信がなかった問題

**Q3：Interceptの操作手順**

正しい操作を選択できたが、RequestとResponseをどの順番でInterceptするかについて自信がなかった。

**Q5：検証結果の判断**

確認できた影響と未検証の影響を区別する回答を選べたが、自信がなかった。

脆弱性診断では、結果を正確に説明する能力も必要なので、検証証拠と結論の対応づけを練習する。

### 8.5 総合評価

アクセス制御の根本原因と、Cookie改変による権限昇格の仕組みについては、基礎的な理解を確認できた。

一方、Cookieの属性に関する知識と、Burp SuiteのIntercept操作の再現性には改善の余地がある。

## 9. 今回の学習成果と今後の課題

### 9.1 習得した操作

- Burp Suiteで `POST /login` をInterceptする
- `Do intercept → Response to this request` を使用する
- RequestをForwardして、対応するResponseをInterceptする
- `Set-Cookie` ヘッダーを書き換える
- 改変したResponseをブラウザへ渡す
- 後続のRequestでCookieの変更を確認する
- ブラウザから管理者画面にアクセスする
- 管理者操作の実行結果とラボのSolvedを確認する

### 9.2 今後の課題

1. `Secure`・`HttpOnly`・`SameSite` の役割を整理する。
2. Request・ResponseのInterceptを自力で再現できるようにする。
3. セッションCookieと権限情報の違いを理解する。
4. サーバー側の認証・認可処理を学習する。
5. HTTP通信と画面上の結果を、検証証拠として整理する。

## 10. 参考資料

- [PortSwigger Web Security Academy：User role controlled by request parameter](https://portswigger.net/web-security/access-control/lab-user-role-controlled-by-request-parameter)
- [PortSwigger Web Security Academy：Access control vulnerabilities](https://portswigger.net/web-security/access-control)
- [OWASP Authorization Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authorization_Cheat_Sheet.html)

---

※ 本検証はPortSwiggerが提供する学習用ラボ環境で実施した。実際の第三者システムに対する不正アクセスやアカウント削除は行っていない。
