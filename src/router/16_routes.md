# Routeを定義する

## はじめに

Routerは簡単に使い始められます。

まず、依存関係へ `leptos_router` パッケージを追加したことを確認してください。`leptos` と
異なり、個別の `csr` と `hydrate` featureはありません。サーバー側でだけ使うことを想定した
`ssr` featureがあるため、サーバー向けビルドでは有効にしてください。

> Routerが `leptos` 本体とは別のパッケージであることは重要です。Routerのすべてを
> ユーザーランドのコードで定義できるということだからです。独自のRouterを作成することも、
> Routerを使わないことも完全に自由です！

そして、次のようにRouterから必要な型をインポートします。

```rust
use leptos_router::components::{Router, Route, Routes};
```

## `<Router/>`を提供する

ルーティングの動作は [`<Router/>`](https://docs.rs/leptos_router/latest/leptos_router/components/fn.Router.html)
コンポーネントが提供します。通常はアプリケーションのルート付近へ配置し、残りの部分を包みます。

> アプリケーション内で複数の `<Router/>` を使うべきではありません。Routerはグローバル
> 状態を駆動します。複数あった場合、URLの変化時にどれが処理を決定するのでしょうか。

Routerを使う単純な `<App/>` コンポーネントから始めましょう。

```rust
use leptos::prelude::*;
use leptos_router::components::Router;

#[component]
pub fn App() -> impl IntoView {
    view! {
      <Router>
        <nav>
          /* ... */
        </nav>
        <main>
          /* ... */
        </main>
      </Router>
    }
}

```

## `<Routes/>`を定義する

[`<Routes/>`](https://docs.rs/leptos_router/latest/leptos_router/components/fn.Routes.html)
コンポーネントでは、利用者がアプリケーション内で移動できるすべてのRouteを定義します。
各Routeは [`<Route/>`](https://docs.rs/leptos_router/latest/leptos_router/components/fn.Route.html)
コンポーネントで定義します。

`<Routes/>` コンポーネントは、Routeをレンダリングしたいアプリケーション内の場所へ配置します。
`<Routes/>` の外側にあるものはすべてのページに表示されるため、ナビゲーションバーやメニュー
などは外側へ置けます。

```rust
use leptos::prelude::*;
use leptos_router::components::*;

#[component]
pub fn App() -> impl IntoView {
    view! {
      <Router>
        <nav>
          /* ... */
        </nav>
        <main>
          // すべてのRouteが<main>内へ表示される
          <Routes fallback=|| "Not found.">
            /* ... */
          </Routes>
        </main>
      </Router>
    }
}
```

`<Routes/>` には、どのRouteにも一致しなかった場合の表示を定義する `fallback` 関数も
指定します。

個々のRouteは、`<Route/>` コンポーネントを `<Routes/>` の子要素として渡して定義します。
`<Route/>` は `path` と `view` を受け取ります。現在のlocationが `path` と一致すると、
`view` が作成されて表示されます。

`path` は `path` マクロを使うと簡単に定義でき、次のものを含められます。

- 静的パス（`/users`）
- コロンで始まる動的な名前付きパラメーター（`/:id`）
- アスタリスクで始まるワイルドカード（`/user/*any`）

`view` はビューを返す関数です。propsを持たない任意のコンポーネントや、ビューを返す
クロージャを指定できます。

```rust
<Routes fallback=|| "Not found.">
  <Route path=path!("/") view=Home/>
  <Route path=path!("/users") view=Users/>
  <Route path=path!("/users/:id") view=UserProfile/>
  <Route path=path!("/*any") view=|| view! { <h1>"Not Found"</h1> }/>
</Routes>
```

> `view` は `Fn() -> impl IntoView` を受け取ります。コンポーネントがpropsを持たなければ、
> `view` へ直接渡せます。この場合の `view=Home` は
> `view=|| view! { <Home/> }` の省略形です。

これで `/` または `/users` へ移動すると、ホームページまたは `<Users/>` が表示されます。
`/users/3` または `/blahblah` へ移動すると、ユーザープロフィールまたは404ページ
（`<NotFound/>`）が表示されます。移動のたびにRouterが一致する `<Route/>` を決定し、
`<Routes/>` コンポーネントを定義した場所へ表示する内容を選びます。

簡単ですね。
