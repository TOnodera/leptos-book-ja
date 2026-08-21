# パラメーターとクエリ

静的パスは異なるページを区別するのに便利ですが、ほぼすべてのアプリケーションで、いずれは
URLを通じてデータを渡す必要があります。

これには2つの方法があります。

1. `/users/:id` の `id` のような名前付きRoute **パラメーター**
2. `/search?q=Foo` の `q` のような名前付きRoute **クエリ**

URLの構造上、クエリには_任意の_ `<Route/>` のビューからアクセスできます。Route
パラメーターには、それを定義した `<Route/>` またはネストした任意の子からアクセスできます。

パラメーターとクエリには、いくつかのhookで簡単にアクセスできます。

- [`use_query`](https://docs.rs/leptos_router/latest/leptos_router/hooks/fn.use_query.html) or [`use_query_map`](https://docs.rs/leptos_router/latest/leptos_router/hooks/fn.use_query_map.html)
- [`use_params`](https://docs.rs/leptos_router/latest/leptos_router/hooks/fn.use_params.html) or [`use_params_map`](https://docs.rs/leptos_router/latest/leptos_router/hooks/fn.use_params_map.html)

それぞれに型付きの方法（`use_query`、`use_params`）と、型なしの方法（`use_query_map`、
`use_params_map`）があります。

型なし版は単純なキーバリューマップを保持します。型付き版を使うには、構造体で
[`Params`](https://docs.rs/leptos_router/latest/leptos_router/params/trait.Params.html)
トレイトをderiveします。

> `Params` は、文字列の平坦なキーバリューマップへ各フィールドの `FromStr` を適用し、
> 構造体へ変換する非常に軽量なトレイトです。RouteパラメーターとURLクエリが平坦な構造の
> ため、`serde` などより柔軟性は大幅に低い一方、バイナリサイズへの影響も小さくなります。

```rust
use leptos::Params;
use leptos_router::params::Params;

#[derive(Params, PartialEq)]
struct ContactParams {
    id: Option<usize>,
}

#[derive(Params, PartialEq)]
struct ContactSearch {
    q: Option<String>,
}
```

> 注意：`Params` deriveマクロは `leptos_router::params::Params` にあります。
>
> stable版ではパラメーターに `Option<T>` だけを使えます。`nightly` featureを使う場合は、
> `T` と `Option<T>` のどちらも使用できます。

これでコンポーネント内から使えます。`/contacts/:id?q=Search` のように、パラメーターと
クエリの両方を持つURLを考えましょう。

型付き版は `Memo<Result<T, _>>` を返します。`Memo` なのでURLの変化に反応します。
パラメーターやクエリをURLからパースする必要があり、不正な可能性もあるため `Result` です。

```rust
use leptos_router::hooks::{use_params, use_query};

let params = use_params::<ContactParams>();
let query = use_query::<ContactSearch>();

// id: || -> usize
let id = move || {
    params
        .read()
        .as_ref()
        .ok()
        .and_then(|params| params.id)
        .unwrap_or_default()
};
```

型なし版は `Memo<ParamsMap>` を返します。これもURLの変化に反応するため `Memo` です。
[`ParamsMap`](https://docs.rs/leptos_router/latest/leptos_router/params/struct.ParamsMap.html) は
ほかのマップ型とよく似ており、`Option<String>` を返す `.get()` メソッドを持ちます。

```rust
use leptos_router::hooks::{use_params_map, use_query_map};

let params = use_params_map();
let query = use_query_map();

// id: || -> Option<String>
let id = move || params.read().get("id");
```

`Option<_>` や `Result<_>` を包むシグナルの派生には複数の手順が必要で、少し煩雑になる場合が
あります。それでも行う価値がある理由は2つあります。

1. 正確になります。「利用者がこのクエリフィールドの値を渡さなかったら？ 不正な値を渡したら？」というケースを必ず考慮することになるためです。
2. 高性能です。パラメーターまたはクエリだけが変化し、同じ `<Route/>` に一致する異なるパス間を移動するとき、再レンダリングせずにアプリケーションの各部分をきめ細かく更新できます。連絡先一覧の例では、別の連絡先へ移動すると、包んでいる `<Contact/>` を置き換えたり再レンダリングしたりせず、名前フィールドと最終的には連絡先情報だけを更新します。これこそが、きめ細かなリアクティビティの目的です。

> これは前節と同じ例です。Routerは統合されたシステムなので、まだすべてを説明していなくても、
> 複数の機能を示すひとつの例を提供するのが適切です。

```admonish sandbox title="実際に動く例" collapsible=true

[クリックしてCodeSandboxを開きます。](https://codesandbox.io/p/devbox/16-router-0-7-csm8t5?file=%2Fsrc%2Fmain.rs)

<noscript>
  例を表示するにはJavaScriptを有効にしてください。
</noscript>

<template>
  <iframe src="https://codesandbox.io/p/devbox/16-router-0-7-csm8t5?file=%2Fsrc%2Fmain.rs" width="100%" height="1000px" style="max-height: 100vh"></iframe>
</template>

```

<details>
<summary>CodeSandboxのソースコード</summary>

```rust
use leptos::prelude::*;
use leptos_router::components::{Outlet, ParentRoute, Route, Router, Routes, A};
use leptos_router::hooks::use_params_map;
use leptos_router::path;

#[component]
pub fn App() -> impl IntoView {
    view! {
        <Router>
            <h1>"Contact App"</h1>
            // この<nav>は<Routes/>の外側にあるため、すべてのRouteで表示される
            // 通常の<a>タグを使うだけで、Routerがクライアント側ナビゲーションを行う
            <nav>
                <a href="/">"Home"</a>
                <a href="/contacts">"Contacts"</a>
            </nav>
            <main>
                <Routes fallback=|| "Not found.">
                    // /にはネストされていない「Home」だけがある
                    <Route path=path!("/") view=|| view! {
                        <h3>"Home"</h3>
                    }/>
                    // /contactsにはネストしたRouteがある
                    <ParentRoute
                        path=path!("/contacts")
                        view=ContactList
                      >
                        // idが指定されていなければfallbackを表示する
                        <ParentRoute path=path!(":id") view=ContactInfo>
                            <Route path=path!("") view=|| view! {
                                <div class="tab">
                                    "(Contact Info)"
                                </div>
                            }/>
                            <Route path=path!("conversations") view=|| view! {
                                <div class="tab">
                                    "(Conversations)"
                                </div>
                            }/>
                        </ParentRoute>
                        // idが指定されていなければfallbackを表示する
                        <Route path=path!("") view=|| view! {
                            <div class="select-user">
                                "Select a user to view contact info."
                            </div>
                        }/>
                    </ParentRoute>
                </Routes>
            </main>
        </Router>
    }
}

#[component]
fn ContactList() -> impl IntoView {
    view! {
        <div class="contact-list">
            // 連絡先一覧コンポーネント本体
            <h3>"Contacts"</h3>
            <div class="contact-list-contacts">
                <A href="alice">"Alice"</A>
                <A href="bob">"Bob"</A>
                <A href="steve">"Steve"</A>
            </div>

            // <Outlet/>はネストした子Routeを表示し、レイアウト内の任意の場所へ配置できる
            <Outlet/>
        </div>
    }
}

#[component]
fn ContactInfo() -> impl IntoView {
    // `use_params_map`で:idパラメーターへリアクティブにアクセスできる
    let params = use_params_map();
    let id = move || params.read().get("id").unwrap_or_default();

    // ここでAPIからデータを読み込むものとする
    let name = move || match id().as_str() {
        "alice" => "Alice",
        "bob" => "Bob",
        "steve" => "Steve",
        _ => "User not found.",
    };

    view! {
        <h4>{name}</h4>
        <div class="contact-info">
            <div class="tabs">
                <A href="" exact=true>"Contact Info"</A>
                <A href="conversations">"Conversations"</A>
            </div>

            // この<Outlet/>は/contacts/:id Routeの下へネストされたタブ
            <Outlet/>
        </div>
    }
}

fn main() {
    leptos::mount::mount_to_body(App)
}
```

</details>
</preview>
