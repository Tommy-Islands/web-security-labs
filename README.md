# web-security-labs

## 概要

Webアプリケーションの脆弱性診断に関する学習・検証記録をまとめたリポジトリです。

これまでのWebサイト制作・運用経験を土台に、Webアプリケーションの仕組みや脆弱性への理解を深め、脆弱性診断の技術習得を目指しています。

主にPortSwigger Web Security Academyを利用し、脆弱性の再現手順、発生原因、想定される影響、対策を記録しています。

## 学習の経緯

2026年10月、CompTIA Security+（SY0-701）を受験しましたが、不合格となりました。

試験対策を通じてセキュリティ全般の基礎知識を学ぶ一方で、知識を実際の技術として活用する経験が不足していると感じました。

この経験をきっかけに、資格取得中心の学習から、実際に手を動かして脆弱性を検証する実践的な学習へ重点を移しています。

現在はPortSwigger Web Security AcademyとBurp Suiteを利用し、HTTP通信の解析、脆弱性の再現、原因の理解、対策の検討に取り組んでいます。

資格の有無だけでなく、自分が何を理解し、何を検証できるのかを成果物として示していきたいと考えています。

## 学習環境

- OS：Parrot Security OS（VirtualBox）
- ツール：Burp Suite Community Edition
- 学習教材：PortSwigger Web Security Academy

## 学習・検証記録

| 分野 | 学習内容 | 状況 |
|---|---|---|
| SQL Injection | [WHERE句の条件回避による非公開データの取得](sql-injection/01-where-clause.md) | 完了 |
| SQL Injection | [UNION攻撃による列数の特定](sql-injection/02-union-column-count.md) | 完了 |
| Cross-Site Scripting（XSS） | 未着手 |
| 認証・アクセス制御 | 未着手 |

## 学習方針

各ラボでは、以下の項目を中心に記録します。

1. 脆弱性の概要と検証目的
2. HTTPリクエストの観察・変更
3. 検証手順と結果
4. 脆弱性が発生する原因
5. 想定される影響と対策
6. 学習を通じて理解したこと・今後の課題

## 注意事項

本リポジトリの内容は、PortSwigger Web Security Academyなど、学習・検証が許可された環境で実施したものです。

実際の第三者システムに対する無許可の検証は行いません。
