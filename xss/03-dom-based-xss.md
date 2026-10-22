# DOM-based XSS：innerHTMLを使用した検索機能の検証

## 1. 検証概要

| 項目 | 内容 |
|---|---|
| 学習プラットフォーム | PortSwigger Web Security Academy |
| ラボ名 | DOM XSS in innerHTML sink using source location.search |
| 難易度 | Apprentice |
| 検証対象 | ブログサイトの検索機能 |
| 脆弱性の種類 | DOM-based XSS |
| 使用ツール | Burp Suite Community Edition / ブラウザ開発者ツール |
| 検証環境 | Parrot Security（VirtualBox） |
| 検証結果 | Solved |

### 検証目的

URLの検索パラメータに由来する入力値が、ブラウザのJavaScriptによってDOMへ挿入される過程を調査する。

特に、`innerHTML` を使用した処理において、入力したHTMLタグが解釈され、JavaScriptの実行につながるかを検証する。

## 2. 攻撃シナリオと背景

### 2.1 想定するシステム

記事の検索機能を備えたブログサイトを想定する。

検索キーワードはURLのクエリパラメータに含まれ、ブラウザ側のJavaScriptによって検索結果ページに表示される。

### 2.2 攻撃者の目的

攻撃者は、検索機能に存在するDOM-based XSSを悪用し、細工したURLを開いた利用者のブラウザで不正なJavaScriptを実行させることを狙う。

実際のサイトでは、偽の入力フォームの表示、ページ内容の改ざん、利用者の権限で実行可能な操作の悪用などにつながる可能性がある。

### 2.3 想定する攻撃の流れ

**第1段階：入力値の反映箇所を調査**

検索欄に通常の文字列を入力し、画面にどのように表示されるか確認する。

**第2段階：HTMLとして解釈されるか検証**

HTMLタグを検索キーワードとして入力し、ブラウザがそのタグをHTML要素として解釈するか確認する。

**第3段階：JavaScriptの実行可否を検証**

イベント属性を含むHTML要素を入力し、JavaScriptが実行されるか確認する。

**第4段階：第三者への攻撃（未実施）**

実際の攻撃では、悪意ある検索パラメータを含むURLを第三者に開かせ、第三者のブラウザでJavaScriptを実行させる可能性がある。

今回のラボでは第三者へのURL送信や情報窃取は実施していない。

## 3. 技術的な背景

### 3.1 DOM-based XSSとは

DOM-based XSSは、ブラウザ側のJavaScriptが信頼できないデータを不安全に処理し、DOMを変更することで発生するXSSである。

今回のラボでは、URLに由来する検索キーワードが `innerHTML` に渡される処理が問題となる。

### 3.2 SourceとSink

DOM-based XSSでは、入力値がどこから取得され、どこで処理されるかを把握することが重要である。

- **Source（入力元）**：外部データの取得元。今回の場合は `location.search`
- **Sink（処理先）**：データが渡され、処理される場所。今回の場合は `innerHTML`

概念を説明するため、以下のコードを考える。

```javascript
const keyword = new URLSearchParams(location.search).get('search');

document.getElementById('searchMessage').innerHTML = keyword;
```

※ このコードは概念説明用であり、ラボの実際のJavaScriptを転記したものではない。

データは次のように流れる。

```text
URLの検索パラメータ
    ↓
location.search
    ↓
URLSearchParamsで値を取得
    ↓
keyword変数
    ↓
innerHTML
    ↓
HTMLとして解釈される
```

このとき、`innerHTML` に不信頼な文字列を直接渡すと、HTML要素やイベント属性が解釈され、XSSにつながる可能性がある。

## 4. 検証手順と結果

### 4.1 通常文字列の反映確認

検索欄に次の文字列を入力した。

```text
DOMTEST123
```

**結果：検索結果ページに `DOMTEST123` が表示された。**

ブラウザの開発者ツールでElementsタブを確認したところ、次のDOM構造を確認した。

```html
<h1>
    <span>0 search results for '</span>
    <span id="searchMessage">DOMTEST123</span>
    <span>'</span>
</h1>
```

この結果から、入力した検索キーワードが `searchMessage` 要素の内容に反映されていることを確認した。

ただし、この段階ではXSSが成立するとは判断できない。

### 4.2 HTMLタグの解釈確認

次に、HTMLタグを含む検索キーワードを入力した。

**入力値**

```html
<b>DOMTEST456</b>
```

**結果：`DOMTEST456` が太字として表示された。**

検索キーワードに含まれる `<b>` タグが、文字列としてではなくHTML要素として解釈されたことを確認した。

この結果から、入力値によってDOMの構造を変更できる状態であると考えられる。

ただし、HTMLタグの解釈だけではJavaScriptの実行可否までは分からない。

### 4.3 JavaScript実行の検証

続いて、以下の文字列を検索欄に入力した。

```html
<img src=x onerror=alert(1)>
```

**結果**

- ブラウザに「1」のダイアログが表示された
- PortSwiggerのラボがSolvedになった

この結果から、入力した文字列を通じてブラウザ上でJavaScriptを実行できることを確認した。

### 4.4 検証結果一覧

| 検証内容 | 入力値 | 結果 |
|---|---|---|
| 通常文字列の反映 | `DOMTEST123` | 画面に表示 |
| HTMLタグの解釈 | `<b>DOMTEST456</b>` | 太字表示 |
| JavaScriptの実行 | `<img src=x onerror=alert(1)>` | ダイアログ表示・Solved |

## 5. 技術的な原因と考察

### 5.1 innerHTMLによるHTMLの解釈

`innerHTML` は、渡された文字列をHTMLとして解析し、対象要素の内容を書き換えるプロパティである。

例えば、次のコードでは、

```javascript
result.innerHTML = '<b>Hello</b>';
```

`<b>` タグがHTML要素として生成され、`Hello` が太字になる。

一方、不信頼なデータにHTMLタグやイベント属性が含まれる場合、意図しないHTML要素や動作が生成される可能性がある。

### 5.2 onerrorによるJavaScript実行

今回使用したペイロードは以下である。

```html
<img src=x onerror=alert(1)>
```

それぞれの役割は次のとおり。

| 要素 | 役割 |
|---|---|
| `img` | 画像要素を生成する |
| `src=x` | 画像の読み込みを試みる |
| `onerror` | 読み込み失敗時のイベント処理 |
| `alert(1)` | JavaScriptの実行確認 |

`src=x` の画像読み込みに失敗すると、`error` イベントが発生する。

その結果、`onerror` に設定された `alert(1)` が実行される。

### 5.3 scriptタグとの違い

`innerHTML` によって後から挿入された `<script>` 要素は、通常そのままでは実行されない。

一方、イベント属性を含むHTML要素が作られると、そのイベントをきっかけにJavaScriptが実行される場合がある。

今回のラボでは、画像の読み込みエラーによってJavaScriptを実行できた。

### 5.4 根本原因

URLから取得した不信頼な入力値が、HTMLとして解釈される処理に適切な対策なしで渡されていたことが根本原因である。

ただし、実際のラボ内部のJavaScriptコードをすべて確認したわけではないため、実装の詳細は断定しない。

## 6. 想定される被害とリスク

### 6.1 今回実証した影響

- 検索キーワードのDOMへの反映
- HTMLタグの解釈
- イベント属性を利用したJavaScriptの実行

### 6.2 想定される被害

実際のWebサイトでDOM-based XSSが悪用された場合、以下の影響が考えられる。

- 偽ログインフォームなどによるフィッシング
- ページ内容の改ざん
- 利用者の権限で実行可能なサイト内操作の悪用
- JavaScriptからアクセス可能な情報の不正取得
- 利用者や企業の信用低下

実際の影響範囲は、ブラウザの制限、アプリケーションの機能、認証方式、Cookie属性などによって異なる。

### 6.3 検証範囲と限界

今回確認したのは、ブラウザ上でJavaScriptを実行できることまでである。

認証情報の取得、Cookieの窃取、第三者に対する攻撃などは実施していない。

そのため、これらの被害が実際に成立するとまでは証明していない。

## 7. 推奨対策

### 7.1 安全なDOM操作を使用する

検索キーワードを単なるテキストとして表示する場合、`innerHTML` ではなく `textContent` を使用する。

**不適切な例**

```javascript
const keyword = new URLSearchParams(location.search).get('search');

document.getElementById('searchMessage').innerHTML = keyword;
```

**改善例**

```javascript
const keyword = new URLSearchParams(location.search).get('search');

document.getElementById('searchMessage').textContent = keyword ?? '';
```

`textContent` は入力値をHTMLとして解析せず、文字列として表示するため、今回のようなHTML解釈によるXSSの防止に適している。

### 7.2 HTMLを許可する場合のサニタイズ

機能上HTMLを表示する必要がある場合は、信頼できるHTMLサニタイズライブラリを使用し、危険な要素や属性を除去する。

単純な文字列置換だけに依存しないことが重要である。

### 7.3 Content Security Policy（CSP）

適切なCSPを導入して、実行できるスクリプトを制限する。

ただし、CSPは補助的な対策であり、安全なDOM操作の代替にはならない。

### 7.4 不信頼な入力の追跡

URLパラメータなどから取得した値が、どのような処理を経てDOMへ反映されるかを確認する。

入力元と処理先の関係を把握し、危険なDOM操作につながっていないか検証する。

## 8. 学習内容と振り返り

### 8.1 今回学んだこと

- DOM-based XSSはブラウザ側のJavaScriptによるDOM操作が原因となる
- URLの検索パラメータが入力元になる場合がある
- `innerHTML` に文字列を渡すとHTMLとして解釈される
- `<script>` タグ以外にも、イベント属性を利用してJavaScriptが実行される場合がある
- `textContent` はテキスト表示のための安全な選択肢となる
- HTMLの解釈とJavaScriptの実行は分けて確認する必要がある
- 検証で実証した事実と想定される被害を区別する必要がある

### 8.2 理解度チェック

全5問中4問正解。

`onerror` の実行条件、安全なDOM操作、データの流れ、実証した影響については正しく回答できた。

一方、SourceとSinkを特定する問題では誤答があり、復習問題では正解したものの、概念に対する理解にはまだ不安が残っている。

### 8.3 今後の課題

- JavaScriptのDOM操作に関する基礎知識を強化する
- Source（入力元）とSink（処理先）の違いを理解する
- 変数を介したデータの流れを追跡できるようになる
- 別のDOM-based XSSラボで、異なる入力元と処理先を確認する
- ブラウザ開発者ツールによるJavaScriptの調査に慣れる

## 9. 参考資料

- [PortSwigger Web Security Academy：DOM XSS in innerHTML sink using source location.search](https://portswigger.net/web-security/cross-site-scripting/dom-based/lab-innerhtml-sink)
- [PortSwigger Web Security Academy：Cross-site scripting](https://portswigger.net/web-security/cross-site-scripting)
- [OWASP Cross Site Scripting Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html)

---

※ 本検証はPortSwiggerが提供する学習用ラボ環境で実施した。第三者のシステムに対する攻撃や機密情報の取得は実施していない。
