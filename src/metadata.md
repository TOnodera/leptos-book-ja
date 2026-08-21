# メタデータ

ここまでレンダリングしたものは、すべてHTML文書の `<body>` 内にありました。ウェブページで
目に見えるものはすべて `<body>` 内にあるため、当然です。

しかし、UIで使うものと同じリアクティブプリミティブやコンポーネントパターンを使って、
文書の `<head>` 内を更新したい場面も数多くあります。

そこで [`leptos_meta`](https://docs.rs/leptos_meta/latest/leptos_meta/) パッケージを使います。

## メタデータコンポーネント

`leptos_meta` は、アプリケーション内の任意のコンポーネントから `<head>` へデータを
挿入できる特別なコンポーネントを提供します。

[`<Title/>`](https://docs.rs/leptos_meta/latest/leptos_meta/fn.Title.html) を使うと、任意の
コンポーネントから文書タイトルを設定できます。ほかのページが設定するタイトルへ同じ形式を
適用する `formatter` 関数も受け取ります。たとえば `<App/>` に
`<Title formatter=|text| format!("{text} — My Awesome Site")/>` を置き、各Routeへ
`<Title text="Page 1"/>` と `<Title text="Page 2"/>` を置くと、タイトルはそれぞれ
`Page 1 — My Awesome Site` と `Page 2 — My Awesome Site` になります。

[`<Link/>`](https://docs.rs/leptos_meta/latest/leptos_meta/fn.Link.html) は `<head>` へ `<link>` 要素を挿入します。

[`<Stylesheet/>`](https://docs.rs/leptos_meta/latest/leptos_meta/fn.Stylesheet.html) は、指定した `href` を持つ `<link rel="stylesheet">` を作成します。

[`<Style/>`](https://docs.rs/leptos_meta/latest/leptos_meta/fn.Style.html) は渡した子要素（通常は文字列）を持つ `<style>` を作成します。`<Style>{include_str!("my_route.css")}</Style>` のように、コンパイル時に別ファイルから独自CSSを読み込めます。

[`<Meta/>`](https://docs.rs/leptos_meta/latest/leptos_meta/fn.Meta.html) は、説明などのメタデータを持つ `<meta>` タグを設定します。

```admonish warning
注意：これらのコンポーネントは、アプリケーション本体にあるコンポーネント内で使ってください。
サーバーサイドレンダリング時などに `<head>` 内で使うべきではありません。`<head>` へ
`leptos_meta` コンポーネントを置く代わりに、対応するHTML要素をそのまま使ってください。
```

## `<Script/>`と`<script>`

`leptos_meta` は [`<Script/>`](https://docs.rs/leptos_meta/latest/leptos_meta/fn.Script.html)
コンポーネントも提供します。ここには注意が必要です。これまでのコンポーネントは `<head>`
専用の要素を `<head>` へ挿入しますが、`<script>` はbody内にも置けます。

大文字の `<Script/>` と小文字の `<script>` の使い分けは単純です。`<Script/>` は `<head>`
へレンダリングされ、`<script>` はほかの通常のHTML要素と同様に、UIの `<body>` 内で
記述した場所へレンダリングされます。JavaScriptの読み込みと実行のタイミングが異なるため、
目的に合う方を使ってください。

## `<Body/>`と`<Html/>`

セマンティックHTMLとスタイリングを簡単にする要素もあります。`<Body/>` と `<Html/>` を
使うと、ページの `<html>` と `<body>` タグへ任意の属性を追加できます。spread演算子
（`{..}`）の後へ通常のLeptos構文で任意の数の属性を記述すると、対応する要素へ直接追加されます。

```rust
<Html
    {..}
    lang="he"
    dir="rtl"
    data-theme="dark"
/>
```

## メタデータとサーバーレンダリング

これらはあらゆる場面で役立ちますが、検索エンジン最適化（SEO）では特に重要です。適切な
`<title>` や `<meta>` タグの設定は欠かせません。現代の検索エンジンクローラーは、空の
`index.html` として配信されJS/WASMだけでレンダリングされるクライアントサイド
アプリケーションにも対応します。しかし、アプリケーションが実際のHTMLへレンダリングされ、
`<head>` にメタデータを持つページの方が好まれます。

`leptos_meta` はまさにこのためのものです。サーバーレンダリング時には、アプリケーション内の
コンポーネントで宣言したすべての `<head>` 向けコンテンツを収集し、実際の `<head>` へ
挿入します。

少し先走りました。サーバーサイドレンダリングはまだ説明していません。次章ではJavaScript
ライブラリとの統合を説明し、その後クライアント側の話を締めくくってサーバーサイド
レンダリングへ進みます。
