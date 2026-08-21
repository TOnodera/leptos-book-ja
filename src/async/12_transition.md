# `<Transition/>`

`<Suspense/>` の例では、データを再読み込みし続けると `"Loading..."` へ戻って点滅し続ける
ことに気づくでしょう。これで問題ない場合もありますが、そうでない場合には
[`<Transition/>`](https://docs.rs/leptos/latest/leptos/suspense/fn.Transition.html) があります。

`<Transition/>` は `<Suspense/>` とまったく同じように動作しますが、毎回fallbackへ戻るのでは
なく、初回だけfallbackを表示します。それ以降の読み込みでは、新しいデータの準備ができるまで
古いデータを表示し続けます。点滅を防ぎ、利用者がアプリケーションを操作し続けられるため、
非常に便利です。

次の例では、`<Transition/>` を使って単純なタブ付き連絡先リストを作成します。新しいタブを
選択すると、新しいデータが読み込まれるまで現在の連絡先を表示し続けます。読み込みメッセージへ
毎回戻るより、はるかによいユーザー体験になる場合があります。

```admonish sandbox title="実際に動く例" collapsible=true

[クリックしてCodeSandboxを開きます。](https://codesandbox.io/p/devbox/12-transition-0-7-ln2hgd?file=%2Fsrc%2Fmain.rs%3A1%2C1-69%2C1&workspaceId=478437f3-1f86-4b1e-b665-5c27a31451fb)

<noscript>
  例を表示するにはJavaScriptを有効にしてください。
</noscript>

<template>
  <iframe src="https://codesandbox.io/p/devbox/12-transition-0-7-ln2hgd?file=%2Fsrc%2Fmain.rs%3A1%2C1-69%2C1&workspaceId=478437f3-1f86-4b1e-b665-5c27a31451fb" width="100%" height="1000px" style="max-height: 100vh"></iframe>
</template>

```

<details>
<summary>CodeSandboxのソースコード</summary>

```rust
use gloo_timers::future::TimeoutFuture;
use leptos::prelude::*;

async fn important_api_call(id: usize) -> String {
    TimeoutFuture::new(1_000).await;
    match id {
        0 => "Alice",
        1 => "Bob",
        2 => "Carol",
        _ => "User not found",
    }
    .to_string()
}

#[component]
fn App() -> impl IntoView {
    let (tab, set_tab) = signal(0);
    let (pending, set_pending) = signal(false);

    // `tab`が変化するたびに再読み込みされる
    let user_data = LocalResource::new(move || important_api_call(tab.get()));

    view! {
        <div class="buttons">
            <button
                on:click=move |_| set_tab.set(0)
                class:selected=move || tab.get() == 0
            >
                "Tab A"
            </button>
            <button
                on:click=move |_| set_tab.set(1)
                class:selected=move || tab.get() == 1
            >
                "Tab B"
            </button>
            <button
                on:click=move |_| set_tab.set(2)
                class:selected=move || tab.get() == 2
            >
                "Tab C"
            </button>
        </div>
        <p>
            {move || if pending.get() {
                "Hang on..."
            } else {
                "Ready."
            }}
        </p>
        <Transition
            // 最初はfallbackを表示し、それ以降の再読み込みでは現在の子要素を表示し続ける
            fallback=move || view! { <p>"Loading initial data..."</p> }
            // Transitionの実行中は`true`に設定される
            set_pending
        >
            <p>
                {move || user_data.read().as_deref().map(ToString::to_string)}
            </p>
        </Transition>
    }
}

fn main() {
    leptos::mount::mount_to_body(App)
}
```

</details>
</preview>
