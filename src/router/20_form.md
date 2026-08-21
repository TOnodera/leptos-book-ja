# `<Form/>`コンポーネント

リンクとフォームは無関係に見えることがありますが、実際には非常によく似た仕組みで動作します。

通常のHTMLで別のページへ移動する方法は3つあります。

1. 別のページへリンクする `<a>` 要素：`href` 属性のURLへHTTP `GET` メソッドで移動します。
2. `<form method="GET">`：入力のフォームデータをURLクエリ文字列へエンコードし、`action` 属性のURLへHTTP `GET` メソッドで移動します。
3. `<form method="POST">`：入力のフォームデータをリクエストボディへエンコードし、`action` 属性のURLへHTTP `POST` メソッドで移動します。

クライアント側Routerがあるため、サーバーと完全に往復してページを再読み込みせず、リンクを
クライアント側で移動できます。同じ方法でフォームもクライアント側から移動できます。

RouterはHTMLの `<form>` 要素と同様に動作しながら、ページ全体の再読み込みではなく
クライアント側ナビゲーションを使う [`<Form>`](https://docs.rs/leptos_router/latest/leptos_router/components/fn.Form.html)
を提供します。`GET` と `POST` の両方に対応します。`method="GET"` ではフォームデータを
エンコードしたURLへ移動し、`method="POST"` では `POST` リクエストを行ってサーバーの応答を処理します。

`<Form/>` は後の章で扱う `<ActionForm/>` や `<MultiActionForm/>` などの基盤です。
また、それ自体でも強力なパターンを実現します。

たとえば、利用者の入力に応じてページを再読み込みせず検索結果をリアルタイムで更新しながら、
検索内容をURLへ保存し、コピーしてほかの人と結果を共有できる検索欄を作るとします。

ここまで学んだパターンを使えば簡単に実装できます。

```rust
async fn fetch_results() {
    // 検索結果を取得する非同期関数
}

#[component]
pub fn FormExample() -> impl IntoView {
    // URLクエリ文字列へリアクティブにアクセスする
    let query = use_query_map();
    // 検索内容は?q=として保存される
    let search = move || query.read().get("q").unwrap_or_default();
    // 検索文字列によって駆動されるResource
    let search_results = Resource::new(search, |_| fetch_results());

    view! {
        <Form method="GET" action="">
            <input type="search" name="q" value=search/>
            <input type="submit"/>
        </Form>
        <Transition fallback=move || ()>
            /* 検索結果をレンダリングする */
            {todo!()}
        </Transition>
    }
}
```

`Submit` をクリックするたびに `<Form/>` は `?q={search}` へ「移動」します。クライアント側で
移動するため、ページのちらつきや再読み込みはありません。URLクエリ文字列が変化すると
`search` が更新されます。`search` は `search_results` Resourceのsourceシグナルなので、
Resourceが再読み込みされます。`<Transition/>` は新しいデータの読み込みが完了するまで現在の
検索結果を表示し続け、完了すると新しい結果へ切り替えます。

これは優れたパターンです。すべてのデータがURLからResourceを経てUIへ流れ、データフローが
非常に明確です。アプリケーションの現在状態はURLへ保存されるため、ページを更新したりリンクを
友人へ送ったりしても、期待どおりの内容が表示されます。サーバーレンダリングを導入すると、
非常に耐障害性が高いこともわかります。内部で `<form>` 要素とURLを使うため、クライアントで
WASMを読み込まなくても正しく動作します。

さらに一歩進めて、少し巧妙なこともできます。

```rust
view! {
	<Form method="GET" action="">
		<input type="search" name="q" value=search
			oninput="this.form.requestSubmit()"
		/>
	</Form>
}
```

このバージョンでは `Submit` ボタンを削除し、入力へ `oninput` 属性を追加しています。これは
`input` イベントを監視してRustコードを実行する `on:input` では_ありません_。コロンのない
`oninput` は通常のHTML属性なので、値はJavaScript文字列です。`this.form` で入力が属する
フォームを取得し、`requestSubmit()` で `<form>` の `submit` イベントを発生させます。
`Submit` ボタンをクリックした場合と同様に `<Form/>` が捕捉します。これでキー入力のたびに
フォームが「移動」し、URLと検索内容を利用者の入力へ完全に同期します。

```admonish sandbox title="実際に動く例" collapsible=true

[クリックしてCodeSandboxを開きます。](https://codesandbox.io/p/devbox/20-form-0-7-m73jsz)

<noscript>
  例を表示するにはJavaScriptを有効にしてください。
</noscript>

<template>
  <iframe src="https://codesandbox.io/p/devbox/20-form-0-7-m73jsz" width="100%" height="1000px" style="max-height: 100vh"></iframe>
</template>

```

<details>
<summary>CodeSandboxのソースコード</summary>

```rust
use leptos::prelude::*;
use leptos_router::components::{Form, Route, Router, Routes};
use leptos_router::hooks::use_query_map;
use leptos_router::path;

#[component]
pub fn App() -> impl IntoView {
    view! {
        <Router>
            <h1><code>"<Form/>"</code></h1>
            <main>
                <Routes fallback=|| "Not found.">
                    <Route path=path!("") view=FormExample/>
                </Routes>
            </main>
        </Router>
    }
}

#[component]
pub fn FormExample() -> impl IntoView {
    // URLクエリへリアクティブにアクセスする
    let query = use_query_map();
    let name = move || query.read().get("name").unwrap_or_default();
    let number = move || query.read().get("number").unwrap_or_default();
    let select = move || query.read().get("select").unwrap_or_default();

    view! {
        // URLクエリ文字列を読み取る
        <table>
            <tr>
                <td><code>"name"</code></td>
                <td>{name}</td>
            </tr>
            <tr>
                <td><code>"number"</code></td>
                <td>{number}</td>
            </tr>
            <tr>
                <td><code>"select"</code></td>
                <td>{select}</td>
            </tr>
        </table>
        // <Form/>は送信されるたびに移動する
        <h2>"Manual Submission"</h2>
        <Form method="GET" action="">
            // inputのnameがクエリ文字列のキーを決める
            <input type="text" name="name" value=name/>
            <input type="number" name="number" value=number/>
            <select name="select">
                // `selected`で最初に選択される項目を設定する
                <option selected=move || select() == "A">
                    "A"
                </option>
                <option selected=move || select() == "B">
                    "B"
                </option>
                <option selected=move || select() == "C">
                    "C"
                </option>
            </select>
            // 送信時には全体を再読み込みせず、クライアント側で移動する
            <input type="submit"/>
        </Form>
        // この<Form/>はJavaScriptを使い、入力のたびに送信する
        <h2>"Automatic Submission"</h2>
        <Form method="GET" action="">
            <input
                type="text"
                name="name"
                value=name
                // このoninput属性により、フィールドへの入力のたびにフォームを送信する
                oninput="this.form.requestSubmit()"
            />
            <input
                type="number"
                name="number"
                value=number
                oninput="this.form.requestSubmit()"
            />
            <select name="select"
                onchange="this.form.requestSubmit()"
            >
                <option selected=move || select() == "A">
                    "A"
                </option>
                <option selected=move || select() == "B">
                    "B"
                </option>
                <option selected=move || select() == "C">
                    "C"
                </option>
            </select>
            // 送信時には全体を再読み込みせず、クライアント側で移動する
            <input type="submit"/>
        </Form>
    }
}

fn main() {
    leptos::mount::mount_to_body(App)
}
```

</details>
</preview>
