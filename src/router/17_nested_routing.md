# ネストしたルーティング

先ほど、次のRoute群を定義しました。

```rust
<Routes fallback=|| "Not found.">
  <Route path=path!("/") view=Home/>
  <Route path=path!("/users") view=Users/>
  <Route path=path!("/users/:id") view=UserProfile/>
  <Route path=path!("/*any") view=|| view! { <h1>"Not Found"</h1> }/>
</Routes>
```

`/users` と `/users/:id` にはある程度の重複があります。小さなアプリケーションなら問題
ありませんが、うまくスケールしないことは想像できるでしょう。これらのRouteをネストできたら
便利ではないでしょうか。

実は……できます！

```rust
<Routes fallback=|| "Not found.">
  <Route path=path!("/") view=Home/>
  <ParentRoute path=path!("/users") view=Users>
    <Route path=path!(":id") view=UserProfile/>
  </ParentRoute>
  <Route path=path!("/*any") view=|| view! { <h1>"Not Found"</h1> }/>
</Routes>
```

`<ParentRoute/>` の内側へ `<Route/>` をネストできます。単純に見えます。

しかし待ってください。アプリケーションの動作がわずかに変化しています。

次の節は、このガイドのルーティングに関する部分全体でも特に重要です。注意深く読み、
わからないことがあれば遠慮なく質問してください。

# レイアウトとしてのネストしたRoute

ネストしたRouteはレイアウトの一形態であり、Routeを定義するためだけの方法ではありません。

別の言い方をしましょう。ネストしたRouteを定義する主な目的は、Route定義でパスを繰り返し
記述するのを避けることではありません。複数の `<Route/>` を同じページへ同時に並べて
表示するよう、Routerへ伝えることです。

実際の例をもう一度見てみましょう。

```rust
<Routes fallback=|| "Not found.">
  <Route path=path!("/users") view=Users/>
  <Route path=path!("/users/:id") view=UserProfile/>
</Routes>
```

これは次のことを意味します。

- `/users` へ移動すると `<Users/>` コンポーネントが表示されます。
- `/users/3` へ移動すると `<UserProfile/>` コンポーネントが表示されます（パラメーター `id` は `3`。詳しくは後述します）。

代わりにネストしたRouteを使うとします。

```rust
<Routes fallback=|| "Not found.">
  <ParentRoute path=path!("/users") view=Users>
    <Route path=path!(":id") view=UserProfile/>
  </ParentRoute>
</Routes>
```

これは次のことを意味します。

- `/users/3` へ移動すると、パスは `<Users/>` と `<UserProfile/>` の2つに一致します。
- `/users` へ移動すると、パスはどのRouteにも一致しません。

実際にはfallbackのRouteを追加する必要があります。

```rust
<Routes>
  <ParentRoute path=path!("/users") view=Users>
    <Route path=path!(":id") view=UserProfile/>
    <Route path=path!("") view=NoUser/>
  </ParentRoute>
</Routes>
```

これで次のようになります。

- `/users/3` へ移動すると、パスは `<Users/>` と `<UserProfile/>` に一致します。
- `/users` へ移動すると、パスは `<Users/>` と `<NoUser/>` に一致します。

つまりネストしたRouteを使うと、各**パス**が複数の**Route**に一致できます。各URLは、複数の
`<Route/>` コンポーネントが提供するビューを同じページへ同時にレンダリングできます。

直感に反するかもしれませんが、この後でわかるように非常に強力です。

## なぜルーティングをネストするのか

なぜこのようなことをするのでしょうか。

ほとんどのウェブアプリケーションには、レイアウトの異なる部分に対応する複数階層の
ナビゲーションがあります。たとえばメールアプリケーションの `/contacts/greg` というURLで、
画面左側に連絡先一覧、右側にGregの連絡先詳細を表示するとします。一覧と詳細は常に同時に
表示するべきです。連絡先が選択されていなければ、簡単な案内文を表示することもできます。

これはネストしたRouteで簡単に定義できます。

```rust
<Routes fallback=|| "Not found.">
  <ParentRoute path=path!("/contacts") view=ContactList>
    <Route path=path!(":id") view=ContactInfo/>
    <Route path=path!("") view=|| view! {
      <p>"Select a contact to view more info."</p>
    }/>
  </ParentRoute>
</Routes>
```

さらに深くできます。各連絡先の住所、メールアドレスと電話番号、会話履歴をタブで表示したいと
します。`:id` の内側へ_もうひとつ_ネストしたRoute群を追加できます。

```rust
<Routes fallback=|| "Not found.">
  <ParentRoute path=path!("/contacts") view=ContactList>
    <ParentRoute path=path!(":id") view=ContactInfo>
      <Route path=path!("") view=EmailAndPhone/>
      <Route path=path!("address") view=Address/>
      <Route path=path!("messages") view=Messages/>
    </ParentRoute>
    <Route path=path!("") view=|| view! {
      <p>"Select a contact to view more info."</p>
    }/>
  </ParentRoute>
</Routes>
```

> React Routerの作者が作ったReactフレームワークである[Remixのウェブサイト](https://remix.run/)の
> メインページを下へスクロールすると、「Sales > Invoices > 個別のinvoice」という3階層の
> ネストしたルーティングを視覚化した優れた例があります。

## `<Outlet/>`

親RouteはネストしたRouteを自動的にはレンダリングしません。結局は単なるコンポーネントであり、
子要素を正確にどこへレンダリングすべきか知らないからです。「親コンポーネントの末尾へ置く」
というだけでは適切な答えになりません。

代わりに `<Outlet/>` コンポーネントを使い、ネストしたコンポーネントをレンダリングする場所を
親コンポーネントへ伝えます。`<Outlet/>` は次のどちらかをレンダリングします。

- 一致したネストRouteがなければ、何も表示しません。
- 一致したネストRouteがあれば、その `view` を表示します。

それだけです。しかし「なぜ動かないのか」という混乱のよくある原因なので、覚えておくことが
重要です。`<Outlet/>` を用意しなければ、ネストしたRouteは表示されません。

```rust
#[component]
pub fn ContactList() -> impl IntoView {
  let contacts = todo!();

  view! {
    <div style="display: flex">
      // 連絡先一覧
      <For each=contacts
        key=|contact| contact.id
        children=|contact| todo!()
      />
      // ネストした子があれば表示する。忘れないこと！
      <Outlet/>
    </div>
  }
}
```

## Route定義をリファクタリングする

すべてのRouteをひとつの場所へ定義する必要はありません。任意の `<Route/>` とその子要素を
別のコンポーネントへ切り出せます。

たとえば上の例は、2つのコンポーネントを使うようにリファクタリングできます。

```rust
#[component]
pub fn App() -> impl IntoView {
    view! {
      <Router>
        <Routes fallback=|| "Not found.">
          <ParentRoute path=path!("/contacts") view=ContactList>
            <ContactInfoRoutes/>
            <Route path=path!("") view=|| view! {
              <p>"Select a contact to view more info."</p>
            }/>
          </ParentRoute>
        </Routes>
      </Router>
    }
}

#[component(transparent)]
fn ContactInfoRoutes() -> impl MatchNestedRoutes + Clone {
    view! {
      <ParentRoute path=path!(":id") view=ContactInfo>
        <Route path=path!("") view=EmailAndPhone/>
        <Route path=path!("address") view=Address/>
        <Route path=path!("messages") view=Messages/>
      </ParentRoute>
    }
    .into_inner()
    .into_any_nested_route()
}
```

2つ目は `#[component(transparent)]` であり、ビューではなくデータをそのまま返す
コンポーネントです。同様に `.into_inner()` で `view` マクロが追加したデバッグ情報を除き、
`<ParentRoute/>` が作成したRoute定義だけを返します。

## ネストしたルーティングと性能

概念としてはよさそうですが、何がそれほど重要なのでしょうか。

性能です。

Leptosのようなきめ細かなリアクティブライブラリでは、レンダリング処理を可能な限り減らすことが
常に重要です。仮想DOMの差分ではなく実際のDOMノードを扱うため、コンポーネントの
「再レンダリング」は最小限にしたいものです。ネストしたルーティングなら簡単に実現できます。

連絡先一覧の例を考えてください。GregからAlice、Bob、そしてGregへ戻ると、移動のたびに
連絡先情報は変化します。しかし `<ContactList/>` は一度も再レンダリングするべきでは
ありません。レンダリングコストを削減するだけでなく、UIの状態も維持できます。たとえば
`<ContactList/>` の上部に検索バーがあっても、連絡先間を移動して検索内容が消えることは
ありません。

実際、この場合は連絡先間を移動するときに `<Contact/>` コンポーネントさえ再レンダリングする
必要がありません。Routerが移動に応じて `:id` パラメーターをリアクティブに更新するため、
きめ細かな更新が可能です。追加の再レンダリングを_一切_行わず、個々のテキストノードだけを
更新して名前や住所などを変更します。

> このsandboxには、この節と前節で説明したネストしたルーティングなどの機能に加え、
> この章の残りで扱う機能も含まれます。Routerは統合されたシステムなので、ひとつの例として
> 示すのが適切です。まだ理解できない部分があっても驚かないでください。

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

            // <Outlet/>はネストした子Routeを表示する
            // レイアウト内の任意の場所へ配置できる
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
