# Web Security Labs

Webアプリケーションの脆弱性診断を目指し、PortSwigger Web Security Academyを使用して取り組んだ学習・検証記録を公開しています。

Webサイトの制作・運用保守で培ったHTML/CSS、JavaScript、CMSなどの知識を土台に、HTTP通信の解析、脆弱性の再現、発生原因の分析、対策の理解に取り組んでいます。

本リポジトリでは、ラボの攻略結果だけでなく、検証過程で確認した事実、技術的な考察、残された課題を記録することを重視しています。

## 1. 経歴・学習背景

### Web関連の実務経験

Web制作・運用保守業務に約8年間従事。

主に以下の業務を経験しています。

- HTML/CSSを使用したWebページの制作・修正
- JavaScript・jQueryを使用した既存機能の修正
- WordPressなどのCMSを利用したサイト運用
- ECサイトやLPの制作・更新
- 既存サイトの保守、不具合調査・対応

これまでのWeb関連の実務経験を活かし、Webアプリケーションの仕組みをセキュリティの観点から理解することを目指しています。

### セキュリティ学習

現在はPortSwigger Web Security Academyを中心に、Webアプリケーションの脆弱性診断技術を学習しています。

Burp Suiteを使用してHTTPリクエスト・レスポンスを観察し、入力値やパラメータの変更によって発生する挙動を検証しています。

特に、以下の点を重視しています。

- 正常なHTTP通信と脆弱性が存在する場合の挙動の違い
- 脆弱性が成立する条件と根本原因
- 検証結果から確認できる影響と、未検証の影響の区別
- 開発・運用側で実施すべき対策
- 検証手順と結果を第三者に説明できる形で記録すること

## 2. 主な検証レポート

これまでに取り組んだ学習内容のうち、代表的なレポートを紹介します。

### SQL Injection

**[WHERE句の条件回避による非公開データの取得](sql-injection/01-where-clause.md)**

SQLクエリの条件式に対する入力値の影響を検証。

正常時と入力変更後のレスポンスの違いを観察し、SQLコメントや条件式によって本来制限されるデータが取得できる仕組みを整理しています。

### Access Control / IDOR

**[IDOR：他人のAPIキー閲覧](access-control/02-idor.md)**

ユーザーを識別するパラメータの変更によって、他ユーザーの情報にアクセスできる問題を検証。

認証と認可の違い、サーバー側でのアクセス制御の必要性、識別子を推測困難にするだけでは根本対策にならない理由を整理しています。

### DOM-based XSS

**[DOM-based XSS：innerHTMLを使用した検索機能の検証](xss/03-dom-based-xss.md)**

JavaScriptによるDOM操作に起因するXSSを検証。

`location.search` から取得した入力値が `innerHTML` に渡される処理を調査し、Source・Sinkの関係と、入力値がHTMLとして解釈される仕組みを整理しています。

## 3. 学習・検証実績

PortSwigger Web Security Academyのラボを使用し、以下の4分野について学習記録を公開しています。

**公開レポート：13件（2026年10月29日時点）**

### SQL Injection（3件）

| 検証内容 | レポート |
|---|---|
| WHERE句の条件回避 | [検証記録](sql-injection/01-where-clause.md) |
| UNION攻撃による列数の特定 | [検証記録](sql-injection/02-union-column-count.md) |
| UNION攻撃で文字列を格納できる列の特定 | [検証記録](sql-injection/03-union-find-text-column.md) |

### Cross-Site Scripting / XSS（3件）

| 検証内容 | レポート |
|---|---|
| Reflected XSS | [検証記録](xss/01-reflected-xss.md) |
| Stored XSS | [検証記録](xss/02-stored-xss.md) |
| DOM-based XSS | [検証記録](xss/03-dom-based-xss.md) |

### Access Control（3件）

| 検証内容 | レポート |
|---|---|
| 保護されていない管理者機能 | [検証記録](access-control/01-unprotected-admin.md) |
| IDORによる他ユーザー情報の閲覧 | [検証記録](access-control/02-idor.md) |
| Cookie改変による管理者権限の偽装 | [検証記録](access-control/03-user-role-cookie.md) |

### Business Logic Vulnerabilities（4件）

| 検証内容 | レポート |
|---|---|
| 価格パラメータ改変による不正購入 | [検証記録](business-logic/01-excessive-trust-client-side.md) |
| 負の数量を利用した購入金額の不正操作 | [検証記録](business-logic/02-high-level-logic-vulnerability.md) |
| メールアドレス変更による管理者機能へのアクセス | [検証記録](business-logic/03-inconsistent-security-controls.md) |
| [クーポンの再利用制限を回避した不正購入](business-logic/04-flawed-enforcement-of-business-rules.md) |

## 4. 学習・検証環境

| 項目 | 使用環境 |
|---|---|
| OS | Parrot Security OS |
| 仮想化環境 | VirtualBox |
| HTTP通信解析 | Burp Suite Community Edition |
| 学習教材 | PortSwigger Web Security Academy |
| 記録管理 | GitHub / Markdown |

## 5. 検証・レポート作成方針

各ラボでは、可能な範囲で以下の項目を整理しています。

1. **検証目的**：何を確認するための検証か
2. **正常時の挙動**：変更前のHTTP通信やアプリケーションの動作
3. **検証手順**：入力値・パラメータなどの変更内容
4. **検証結果**：実際に確認できたレスポンスや画面の変化
5. **発生原因**：脆弱性が成立する技術的な仕組み
6. **影響と対策**：想定されるリスクと推奨される対処方法
7. **振り返り**：理解できたこと、追加学習が必要なこと

なお、検証していない事象については、確認済みの結果と区別して記録する方針です。

レポート作成では、文章の整理や技術理解の補助にAIを利用する場合があります。検証作業を実施した範囲と、実際に確認できた結果を区別することを重視しています。

## 6. 今後の学習課題

現在は以下の技術・知識の習得を課題としています。

- HTTP・TCP/IPなどのネットワーク基礎
- Linux・Webサーバーの動作原理
- 認証・セッション管理・アクセス制御
- Webアプリケーションのバックエンド処理
- 検証結果に基づいた診断報告書の作成
- 未知の脆弱性に対する検証方法の立案

今後は既存の脆弱性の再現学習に加え、検証方針の立案、証拠の収集、技術的な判断をより自立して行えることを目標としています。

## 7. 注意事項

本リポジトリの検証は、PortSwigger Web Security Academyなど、学習・検証が許可された環境で行っています。

第三者のシステムに対する無許可の検証は行いません。

本リポジトリは学習成果をまとめたものであり、商用システムに対する脆弱性診断の実務実績を示すものではありません。
