# `<Suspense/>`

前章では、Resourceの読み込み中にfallbackを表示する単純な読み込み画面の作成方法を示しました。

```rust
let (count, set_count) = signal(0);
let once = Resource::new(move || count.get(), |count| async move { load_a(count).await });

view! {
    <h1>"My Data"</h1>
    {move || match once.get() {
        None => view! { <p>"Loading..."</p> }.into_any(),
        Some(data) => view! { <ShowData data/> }.into_any()
    }}
}
```

では、2つのResourceがあり、両方を待ちたい場合はどうでしょうか。

```rust
let (count, set_count) = signal(0);
let (count2, set_count2) = signal(0);
let a = Resource::new(move || count.get(), |count| async move { load_a(count).await });
let b = Resource::new(move || count2.get(), |count| async move { load_b(count).await });

view! {
    <h1>"My Data"</h1>
    {move || match (a.get(), b.get()) {
        (Some(a), Some(b)) => view! {
            <ShowA a/>
            <ShowA b/>
        }.into_any(),
        _ => view! { <p>"Loading..."</p> }.into_any()
    }}
}
```

それほど_悪くはありません_が、少し面倒です。制御の流れを反転できたらどうでしょうか。

[`<Suspense/>`](https://docs.rs/leptos/latest/leptos/suspense/fn.Suspense.html) コンポーネントを
使うと、まさにそれを実現できます。`fallback` propsと子要素を渡します。通常、子要素の
ひとつ以上がResourceを読み取ります。`<Suspense/>` の「下」、つまり子要素内でResourceを
読み取ると、そのResourceが `<Suspense/>` へ登録されます。Resourceの読み込みを待っている
間は `fallback` を表示し、すべて読み込まれると子要素を表示します。

```rust
let (count, set_count) = signal(0);
let (count2, set_count2) = signal(0);
let a = Resource::new(count, |count| async move { load_a(count).await });
let b = Resource::new(count2, |count| async move { load_b(count).await });

view! {
    <h1>"My Data"</h1>
    <Suspense
        fallback=move || view! { <p>"Loading..."</p> }
    >
        <h2>"My Data"</h2>
        <h3>"A"</h3>
        {move || {
            a.get()
                .map(|a| view! { <ShowA a/> })
        }}
        <h3>"B"</h3>
        {move || {
            b.get()
                .map(|b| view! { <ShowB b/> })
        }}
    </Suspense>
}
```

いずれかのResourceが再読み込みされるたびに、`"Loading..."` のfallbackが再び表示されます。

このように制御の流れを反転すると、自分でパターンマッチを処理する必要がなくなり、個々の
Resourceを簡単に追加・削除できます。また、サーバーサイドレンダリング時の大幅な性能向上も
可能になります。これについては後の章で説明します。

`<Suspense/>` を使うとResourceを直接 `.await` する便利な方法も利用でき、上のコードから
ネストをひとつ減らせます。`Suspend` 型を使うと、ビュー内で利用できるレンダリング可能な
`Future` を作成できます。

```rust
view! {
    <h1>"My Data"</h1>
    <Suspense
        fallback=move || view! { <p>"Loading..."</p> }
    >
        <h2>"My Data"</h2>
        {move || Suspend::new(async move {
            let a = a.await;
            let b = b.await;
            view! {
                <h3>"A"</h3>
                <ShowA a/>
                <h3>"B"</h3>
                <ShowB b/>
            }
        })}
    </Suspense>
}
```

`Suspend` を使うと各Resourceのnullチェックを避けられ、コードの複雑さをさらに減らせます。

## `<Await/>`

単に `Future` の解決を待ってからレンダリングしたい場合、`<Await/>` コンポーネントを使うと
定型コードを減らせます。`<Await/>` は本質的に `OnceResource` と、fallbackを持たない
`<Suspense/>` を組み合わせたものです。

つまり、次のように動作します。

1. `Future` を一度だけポーリングし、リアクティブな変化には応答しません。
2. `Future` が解決するまで何もレンダリングしません。
3. `Future` の解決後、データを指定した変数名へ束縛し、その変数をスコープに含めて子要素をレンダリングします。

```rust
async fn fetch_monkeys(monkey: i32) -> i32 {
    // これは非同期でなくてもよかったかもしれない
    monkey * 2
}
view! {
    <Await
        // `future`へ解決対象の`Future`を指定する
        future=fetch_monkeys(3)
        // データは指定した変数名へ束縛される
        let:data
    >
        // データを参照として受け取り、ここでビューに使用できる
        <p>{*data} " little monkeys, jumping on the bed."</p>
    </Await>
}
```

```admonish sandbox title="実際に動く例" collapsible=true

[クリックしてCodeSandboxを開きます。](https://codesandbox.io/p/devbox/11-suspense-0-7-sr2srk?file=%2Fsrc%2Fmain.rs%3A1%2C1-55%2C1)

<noscript>
  例を表示するにはJavaScriptを有効にしてください。
</noscript>

<template>
  <iframe src="https://codesandbox.io/p/devbox/11-suspense-0-7-sr2srk?file=%2Fsrc%2Fmain.rs%3A1%2C1-55%2C1" width="100%" height="1000px" style="max-height: 100vh"></iframe>
</template>

```

<details>
<summary>CodeSandboxのソースコード</summary>

```rust
use gloo_timers::future::TimeoutFuture;
use leptos::prelude::*;

async fn important_api_call(name: String) -> String {
    TimeoutFuture::new(1_000).await;
    name.to_ascii_uppercase()
}

#[component]
pub fn App() -> impl IntoView {
    let (name, set_name) = signal("Bill".to_string());

    // `name`が変化するたびに再読み込みされる
    let async_data = LocalResource::new(move || important_api_call(name.get()));

    view! {
        <input
            on:change:target=move |ev| {
                set_name.set(ev.target().value());
            }
            prop:value=name
        />
        <p><code>"name:"</code> {name}</p>
        <Suspense
            // Suspenseの「下」で読み取るResourceの読み込み中はfallbackを表示する
            fallback=move || view! { <p>"Loading..."</p> }
        >
            // Suspendを使うとビュー内でasyncブロックを使用できる
            <p>
                "Your shouting name is "
                {move || Suspend::new(async move {
                    async_data.await
                })}
            </p>
        </Suspense>
        <Suspense
            // Suspenseの「下」で読み取るResourceの読み込み中はfallbackを表示する
            fallback=move || view! { <p>"Loading..."</p> }
        >
            // 子要素は最初に一度、その後はいずれかのResourceが解決するたびにレンダリングされる
            <p>
                "Which should be the same as... "
                {move || async_data.get().as_deref().map(ToString::to_string)}
            </p>
        </Suspense>
    }
}

fn main() {
    leptos::mount::mount_to_body(App)
}
```

</details>
</preview>
