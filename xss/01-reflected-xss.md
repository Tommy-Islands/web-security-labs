# Reflected XSS：検索機能におけるJavaScript実行の検証

## 1. 検証概要

| 項目 | 内容 |
|---|---|
| 学習プラットフォーム | PortSwigger Web Security Academy |
| ラボ名 | Reflected XSS into HTML context with nothing encoded |
| 難易度 | Apprentice |
| 使用ツール | Burp Suite Community Edition |
| 対象機能 | ブログサイトの検索機能 |
| 対象パラメータ | `search` |
| 脆弱性の種類 | Reflected XSS（反射型クロスサイトスクリプティング） |
| 検証結果 | Solved |

### 検証目的

検索欄に入力した文字列がHTTPレスポンスにどのように反映されるか調査し、HTMLタグやJavaScriptが解釈・実行されるか検証する。

## 2. 攻撃シナリオと背景

### 想定するシステム

記事検索機能を備えたブログサイトを想定する。

利用者が検索欄にキーワードを入力すると、サーバーは検索結果を表示し、入力したキーワードも検索結果ページに表示する。

今回の対象では、検索パラメータに含まれる入力値がHTML内に反映される。

### 攻撃者の目的

攻撃者は、検索機能に存在するReflected XSSを悪用し、細工したURLを開いた利用者のブラウザで任意のJavaScriptを実行させることを狙う。

実際のシステムでは、以下の被害につながる可能性がある。

- 偽のログインフォームなどを利用したフィッシング
- 利用者の権限で実行可能なサイト内操作の悪用
- ページ内容の改ざんや偽情報の表示
- JavaScriptからアクセス可能な情報の不正取得

ただし、これらは想定されるリスクであり、今回のラボで実行したものではない。

### 想定する攻撃の流れ

**第1段階：入力箇所の発見**

攻撃者は、ブログの検索欄が入力値を検索結果ページに表示することを確認する。

**第2段階：HTMLとして解釈されるか調査**

HTMLタグを含む文字列を入力し、ブラウザが単なる文字列ではなくHTML要素として解釈するか確認する。

**第3段階：JavaScript実行の検証**

JavaScriptを含むHTMLを入力し、ブラウザでスクリプトが実行されるか確認する。

**第4段階：第三者への攻撃（未実施）**

実際の攻撃では、悪意ある検索パラメータを含むURLを第三者に開かせることで、その第三者のブラウザ上でJavaScriptを実行させることが考えられる。

今回のラボでは、第三者へのURL送信や情報窃取は行っていない。

## 3. 検証環境

- OS：Parrot Security（VirtualBox上の仮想環境）
- 使用ツール：Burp Suite Community Edition
- 使用機能：Proxy / HTTP history
- 対象：PortSwigger Web Security Academyが提供する学習用ラボ

## 4. 検証手順と結果

### 4.1 検索文字列の反映確認

まず、通常の文字列を検索欄に入力した。

**入力値**

```text
xsstest123
```

**ブラウザ上の表示**

```text
0 search results for 'xsstest123'
```

さらに、Burp SuiteのHTTP historyでレスポンスを確認した。

**確認したHTML**

```html
<section class=blog-header>
    <h1>0 search results for 'xsstest123'</h1>
    <hr>
</section>
```

**結果：検索キーワードがHTML内に反映されていることを確認した。**

この段階では、入力した文字列が表示されただけなので、XSSが成立するとまでは判断できない。

### 4.2 HTMLタグの解釈を確認

次に、HTMLタグを含む文字列を検索欄へ入力した。

**入力値**

```html
<b>XSSTEST</b>
```

**結果：`XSSTEST` が太字で表示された。**

HTMLの `<b>` タグが文字列としてではなく、HTML要素として解釈されたことを確認した。

この結果から、検索文字列を使ってHTMLの構造に影響を与えられる状態であると判断した。

ただし、HTMLタグが解釈されたことだけでは、JavaScriptが実行可能であることまでは証明できない。

### 4.3 JavaScript実行の確認

最後に、JavaScriptを含む文字列を検索欄へ入力した。

**入力値**

```html
<script>alert(1)</script>
```

**結果**

- ブラウザに「1」と表示されるダイアログが出現した
- PortSwiggerのラボがSolvedになった

`alert(1)` は、ブラウザでJavaScriptが実行されたことを確認するための動作確認用コードである。

実際の情報窃取や不正なサイト操作は行っていない。

### 4.4 検証結果のまとめ

| 検証内容 | 入力値 | 確認した結果 |
|---|---|---|
| 通常文字列の反映 | `xsstest123` | 検索結果ページに表示 |
| HTMLタグの解釈 | `<b>XSSTEST</b>` | 文字が太字で表示 |
| JavaScriptの実行 | `<script>alert(1)</script>` | ダイアログ表示、Solved |

以上により、検索機能でReflected XSSが成立することを確認した。

## 5. 技術的な原因と考察

### 5.1 Reflected XSSとは

Reflected XSSは、HTTPリクエストに含まれる入力値が適切に処理されず、そのリクエストへのHTTPレスポンスに埋め込まれることで発生するXSSである。

入力値が実行可能なHTMLやJavaScriptとしてブラウザに解釈されると、攻撃コードが実行される可能性がある。

### 5.2 今回のHTMLコンテキスト

今回の検索キーワードは、次のようにHTML要素のテキスト部分へ反映されていた。

```html
<h1>0 search results for 'xsstest123'</h1>
```

この箇所に、HTMLタグを含む検索文字列が適切にエスケープされずに挿入されると、ブラウザはそのタグをHTML構造として解釈する。

例えば、次のようなHTMLがサーバーから返された場合を考える。

```html
<h1>0 search results for '<script>alert(1)</script>'</h1>
```

この場合、ブラウザは `<script>` 要素を解釈し、JavaScriptを実行する可能性がある。

※ 上記は仕組みを説明するための例であり、実際の攻撃時のHTMLレスポンス全文を取得したものではない。

### 5.3 根本原因

ユーザー入力をHTMLの文脈に適した方法でエスケープせず、レスポンスへ埋め込んでいることが根本原因と考えられる。

今回のラボでは、検索文字列がHTMLタグとして解釈され、さらにJavaScriptが実行されることを確認した。

ただし、サーバー側のソースコードは確認していないため、内部の実装方法については断定しない。

### 5.4 SQLインジェクションとの違い

SQLインジェクションは、入力値がSQL命令として解釈されることで成立する。

一方、今回のXSSは、入力値がHTMLやJavaScriptとしてブラウザに解釈されたことで成立した。

どちらも、信頼できない入力が本来のデータとして扱われず、別の意味を持つコードとして処理される点が共通している。

## 6. 想定される被害とリスク

### 今回実証した影響

- 検索キーワードのHTTPレスポンスへの反映
- HTMLタグの解釈
- JavaScriptの実行

### 実際のシステムで想定される影響

Reflected XSSが悪用された場合、攻撃者が用意したURLを利用者に開かせることで、利用者のブラウザ上で悪意ある処理を実行できる可能性がある。

例えば、ページに偽の入力フォームを表示したり、利用者が操作できるサイト内機能を不正に利用したりすることが考えられる。

ただし、実際の影響はブラウザのセキュリティ制限、認証方式、Cookie属性、サイト側の実装などによって異なる。

### 検証範囲と限界

今回の検証では、JavaScriptが実行できることまで確認した。

一方、認証情報の窃取、Cookieの取得、利用者になりすました操作などは実施していない。

そのため、これらの被害が実際に成立することまでは証明していない。

## 7. 推奨対策

### 7.1 HTML出力時の適切なエスケープ

ユーザー入力をHTMLへ表示する場合、出力先の文脈に応じてエスケープ処理を行う。

HTMLのテキスト部分では、少なくとも `<` や `&` などの特殊文字を安全に扱い、入力したHTMLタグが要素として解釈されないようにする。

例えば、次の入力値がある場合、

```html
<script>alert(1)</script>
```

HTMLテキストとして出力する際には、概念的に次のように変換する。

```html
&lt;script&gt;alert(1)&lt;/script&gt;
```

これにより、ブラウザはスクリプトではなく文字列として表示する。

ただし、HTML属性、URL、JavaScriptなど、出力先の文脈によって必要な処理は異なる。

### 7.2 安全なDOM操作の使用

JavaScriptで検索文字列を画面へ表示する場合、単なるテキストとして表示する用途には `textContent` を使用する。

**不適切な例**

```javascript
const keyword = new URLSearchParams(location.search).get('search');

document.getElementById('result').innerHTML = keyword;
```

**改善例**

```javascript
const keyword = new URLSearchParams(location.search).get('search');

document.getElementById('result').textContent = keyword ?? '';
```

`innerHTML` は入力文字列をHTMLとして解釈するため、不信頼データを直接代入すると危険な場合がある。

一方、`textContent` は文字列として挿入するため、この用途ではより安全に扱える。

ただし、今回のラボはサーバーから返されるHTMLに起因するReflected XSSであり、`textContent` はDOM操作を伴う別の実装例に対する対策である。

### 7.3 Content Security Policy（CSP）の導入

適切なCSPを設定することで、実行可能なスクリプトの読み込み元や実行方法を制限できる。

ただし、CSPだけでXSSを根本的に防げるわけではなく、出力時のエスケープなどと併用する必要がある。

### 7.4 Cookie属性の適切な設定

セッションCookieに `HttpOnly` などの適切な属性を設定することで、JavaScriptからのCookie参照を制限できる。

ただし、Cookieが読み取れない場合でも、XSSによるサイト内操作などの危険が残る可能性がある。

## 8. 学習内容と振り返り

### 今回学んだこと

- Reflected XSSはリクエストの入力がレスポンスに反映されることで発生し得る
- 文字列の反映だけではXSSの成立を証明できない
- HTMLタグが解釈されることと、JavaScriptが実行されることは別である
- JavaScriptは利用者のブラウザ上で実行される
- `innerHTML` と `textContent` では入力文字列の扱いが異なる
- XSS対策では出力先の文脈に応じた安全な処理が重要になる
- 実証した事実と、想定される被害を分けて報告する必要がある

### 理解度チェック

全5問を実施し、最終的に5問正解した。

うち4問は初見で自信を持って正解した。

`innerHTML` と `textContent` の違いについては、当初理解が不十分だったが、解説と具体例を通じて理解した。

### 今後の課題

- JavaScriptによる安全なDOM操作を実践する
- HTMLの出力コンテキストと適切なエスケープを理解する
- Stored XSSやDOM-based XSSとの違いを実技で確認する
- XSSの影響範囲を根拠に基づいて評価できるようになる

## 9. 参考資料

- [PortSwigger Web Security Academy：Reflected XSS into HTML context with nothing encoded](https://portswigger.net/web-security/cross-site-scripting/reflected/lab-html-context-nothing-encoded)
- [PortSwigger Web Security Academy：Cross-site scripting](https://portswigger.net/web-security/cross-site-scripting)
- [OWASP Cross Site Scripting Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html)

---

※ 本検証はPortSwiggerが提供する学習用ラボで実施した。第三者のシステムに対する攻撃や機密情報の取得は実施していない。
