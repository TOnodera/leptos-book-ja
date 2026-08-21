# エラー処理

[前章](./06_control_flow.md)では、`Option<T>`をレンダリングできることを説明しました。`None`の場合は何もレンダリングせず、`Some(T)`の場合は`T`をレンダリングします（`T`が`IntoView`を実装している場合）。`Result<T, E>`でも、よく似たことができます。`Err(_)`の場合は何もレンダリングせず、`Ok(T)`の場合は`T`をレンダリングします。

数値入力を受け取る単純なコンポーネントから始めましょう。

```rust
#[component]
fn NumericInput() -> impl IntoView {
    let (value, set_value) = signal(Ok(0));

    view! {
        <label>
            "Type an integer (or not!)"
            <input type="number" on:input:target=move |ev| {
              // 入力が変化したら、入力値を数値として解析する
              set_value.set(ev.target().value().parse::<i32>())
            }/>
            <p>
                "You entered "
                <strong>{value}</strong>
            </p>
        </label>
    }
}
```

入力を変更するたびに、`on_input`はその値を32ビット整数（`i32`）として解析し、`Result<i32, _>`である`value`シグナルへ保存しようとします。数値`42`を入力すると、UIには次のように表示されます。

```
You entered 42
```

しかし、文字列`foo`を入力すると、次のように表示されます。

```
You entered
```

これはあまり望ましい動作ではありません。`.unwrap_or_default()`などを使わずに済んではいますが、エラーを捕捉して何らかの処理を行えれば、もっと便利です。

それを実現するのが[`<ErrorBoundary/>`](https://docs.rs/leptos/latest/leptos/error/fn.ErrorBoundary.html)コンポーネントです。

```admonish note
`<input type="number">`では`foo`のような文字列や数値以外のものを入力できない、と指摘されることがよくあります。一部のブラウザではそのとおりですが、すべてではありません。また、通常の数値入力欄には、浮動小数点数、32ビットを超える数値、文字`e`など、`i32`ではないさまざまな値を入力できます。これらの条件の一部を守るようブラウザへ指示することはできますが、ブラウザごとに動作は異なります。自分で解析することが重要です。
```

## `<ErrorBoundary/>`

`<ErrorBoundary/>`は、前章で見た`<Show/>`コンポーネントに少し似ています。すべてが正常、つまりすべて`Ok(_)`であれば子をレンダリングします。しかし、その子のなかで`Err(_)`がレンダリングされると、`<ErrorBoundary/>`の`fallback`が呼び出されます。

この例へ`<ErrorBoundary/>`を追加しましょう。

```rust
#[component]
fn NumericInput() -> impl IntoView {
        let (value, set_value) = signal(Ok(0));

    view! {
        <h1>"Error Handling"</h1>
        <label>
            "Type a number (or something that's not a number!)"
            <input type="number" on:input:target=move |ev| {
                // 入力が変化したら、入力値を数値として解析する
                set_value.set(ev.target().value().parse::<i32>())
            }/>
            // <ErrorBoundary/>内で`Err(_)`がレンダリングされるとfallbackが表示され、
            // それ以外の場合は<ErrorBoundary/>の子が表示される
            <ErrorBoundary
                // fallbackは現在のエラーを格納するシグナルを受け取る
                fallback=|errors| view! {
                    <div class="error">
                        <p>"Not a number! Errors: "</p>
                        // 必要であれば、エラー一覧を文字列としてレンダリングできる
                        <ul>
                            {move || errors.get()
                                .into_iter()
                                .map(|(_, e)| view! { <li>{e.to_string()}</li>})
                                .collect::<Vec<_>>()
                            }
                        </ul>
                    </div>
                }
            >
                <p>
                    "You entered "
                    // `value`は`Result<i32, _>`なので、`Ok`なら`i32`をレンダリングし、
                    // `Err`なら何もレンダリングせずエラー境界を起動する
                    // シグナルなので、`value`が変化すると動的に更新される
                    <strong>{value}</strong>
                </p>
            </ErrorBoundary>
        </label>
    }
}
```

これで`42`を入力すると`value`は`Ok(42)`になり、次のように表示されます。

```
You entered 42
```

`foo`を入力すると、valueは`Err(_)`になり、`fallback`がレンダリングされます。ここではエラー一覧を`String`としてレンダリングするため、次のような内容が表示されます。

```
Not a number! Errors:
- cannot parse integer from empty string
```

エラーを修正するとエラーメッセージが消え、`<ErrorBoundary/>`で包んだコンテンツが再び表示されます。

```admonish sandbox title="実際に動く例" collapsible=true

[クリックしてCodeSandboxを開きます。](https://codesandbox.io/p/devbox/7-errors-0-7-qqywqz?file=%2Fsrc%2Fmain.rs%3A5%2C1-46%2C6&workspaceId=478437f3-1f86-4b1e-b665-5c27a31451fb)

<noscript>
  例を表示するにはJavaScriptを有効にしてください。
</noscript>

<template>
  <iframe src="https://codesandbox.io/p/devbox/7-errors-0-7-qqywqz?file=%2Fsrc%2Fmain.rs%3A5%2C1-46%2C6&workspaceId=478437f3-1f86-4b1e-b665-5c27a31451fb" width="100%" height="1000px" style="max-height: 100vh"></iframe>
</template>
```

<details>
<summary>CodeSandboxのソースコード</summary>

```rust
use leptos::prelude::*;

#[component]
fn App() -> impl IntoView {
    let (value, set_value) = signal(Ok(0));

    view! {
        <h1>"Error Handling"</h1>
        <label>
            "Type a number (or something that's not a number!)"
            <input type="number" on:input:target=move |ev| {
                // 入力が変化したら、その値を数値としてパースする
                set_value.set(ev.target().value().parse::<i32>())
            }/>
            // <ErrorBoundary/>内で`Err(_)`がレンダリングされた場合は
            // fallbackが表示される。それ以外の場合は
            // <ErrorBoundary/>の子要素が表示される
            <ErrorBoundary
                // fallbackは現在のエラーを保持するシグナルを受け取る
                fallback=|errors| view! {
                    <div class="error">
                        <p>"Not a number! Errors: "</p>
                        // 必要ならエラーの一覧を文字列としてレンダリングできる
                        <ul>
                            {move || errors.get()
                                .into_iter()
                                .map(|(_, e)| view! { <li>{e.to_string()}</li>})
                                .collect::<Vec<_>>()
                            }
                        </ul>
                    </div>
                }
            >
                <p>
                    "You entered "
                    // `value`は`Result<i32, _>`なので、`Ok`なら`i32`を
                    // レンダリングし、`Err`なら何もレンダリングせず
                    // エラーバウンダリを発動する。シグナルなので、
                    // `value`の変化に応じて動的に更新される
                    <strong>{value}</strong>
                </p>
            </ErrorBoundary>
        </label>
    }
}

fn main() {
    leptos::mount::mount_to_body(App)
}
```

</details>
</preview>
