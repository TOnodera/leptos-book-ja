# Actionでデータを変更する

Resourceで `async` データを読み込む方法を説明しました。Resourceはすぐにデータを読み込み、
`<Suspense/>` や `<Transition/>` と連携して、アプリケーションでデータが読み込み中かどうかを
表示します。しかし、任意の `async` 関数を呼び出し、その状態を追跡したいだけの場合は
どうすればよいでしょうか。

もちろん [`spawn_local`](https://docs.rs/leptos/latest/leptos/task/fn.spawn_local.html) を使えます。
`Future` をブラウザ、またはサーバー上のTokioなどのランタイムへ渡し、同期環境で `async`
タスクを起動できます。しかし、まだ保留中かどうかを知るにはどうすればよいでしょうか。
読み込み中かを示すシグナルと、結果を示す別のシグナルを設定することはできます……。

それでも正しく動作しますが、最後の `async` プリミティブである
[`Action`](https://docs.rs/leptos/latest/leptos/reactive/actions/struct.Action.html) も使えます。

ActionとResourceは似ていますが、根本的に異なるものを表します。`async` 関数を一度、または
別の値が変化したときに実行してデータを読み込むなら、Resourceが適しています。利用者の
ボタンクリックなどに応じて、ときどき `async` 関数を実行するなら、`Action` が適しています。

実行したい `async` 関数があるとします。

```rust
async fn add_todo_request(new_title: &str) -> Uuid {
    /* サーバー上で新しいTodoを追加する処理 */
}
```

`Action::new()` は、ひとつの引数への参照を受け取る `async` 関数を取ります。この引数は
Actionの「入力型」と考えられます。

> 入力は常にひとつの型です。複数の引数を渡したい場合は構造体またはタプルを使えます。
>
> ```rust
> // 引数がひとつなら、その型を使う
> let action1 = Action::new(|input: &String| {
>    let input = input.clone();
>    async move { todo!() }
> });
>
> // 引数がなければユニット型`()`を使う
> let action2 = Action::new(|input: &()| async { todo!() });
>
> // 引数が複数ならタプルを使う
> let action3 = Action::new(
>   |input: &(usize, String)| async { todo!() }
> );
> ```
>
> Action関数は参照を受け取りますが、`Future` には `'static` ライフタイムが必要なため、
> 通常は値をクローンして `Future` へ渡す必要があります。確かに不格好ですが、楽観的UIの
> ような強力な機能を可能にします。これについては後の章でもう少し説明します。

この場合、Actionを作成するために必要なのは次のコードだけです。

```rust
let add_todo_action = Action::new(|input: &String| {
    let input = input.to_owned();
    async move { add_todo_request(&input).await }
});
```

`add_todo_action` を直接呼び出す代わりに、次のように `.dispatch()` で呼び出します。

```rust
add_todo_action.dispatch("Some value".to_string());
```

イベントリスナー、タイムアウトなど、どこからでも呼び出せます。`.dispatch()` は `async`
関数ではないため、同期コンテキストから呼び出せます。

Actionは、呼び出した非同期Actionと同期的なリアクティブシステムを同期する複数のシグナルを
提供します。

```rust
let submitted = add_todo_action.input(); // RwSignal<Option<String>>
let pending = add_todo_action.pending(); // ReadSignal<bool>
let todo_id = add_todo_action.value(); // RwSignal<Option<Uuid>>
```

これにより、リクエストの現在状態の追跡、読み込みインジケーターの表示、送信が成功すると
仮定した「楽観的UI」の実装が簡単になります。

```rust
let input_ref = NodeRef::<Input>::new();

view! {
    <form
        on:submit=move |ev| {
            ev.prevent_default(); // ページを再読み込みしない……
            let input = input_ref.get().expect("input to exist");
            add_todo_action.dispatch(input.value());
        }
    >
        <label>
            "What do you need to do?"
            <input type="text"
                node_ref=input_ref
            />
        </label>
        <button type="submit">"Add Todo"</button>
    </form>
    // 読み込み状態を使う
    <p>{move || pending.get().then_some("Loading...")}</p>
}
```

ここまでの内容が少し複雑すぎる、あるいは制約が多すぎると感じるかもしれません。Resourceと
並ぶ最後のピースとして、ここでActionを紹介しました。実際のLeptosアプリケーションでは、
Actionをサーバー関数、[`ServerAction`](https://docs.rs/leptos/latest/leptos/server/struct.ServerAction.html)、
[`<ActionForm/>`](https://docs.rs/leptos/latest/leptos/form/fn.ActionForm.html) と組み合わせ、
非常に強力なプログレッシブエンハンスメント対応フォームを作成することが多いでしょう。
このプリミティブが役に立たないように見えても心配はいりません。後で理解できるかもしれません
（または今すぐ [`todo_app_sqlite`](https://github.com/leptos-rs/leptos/blob/main/examples/todo_app_sqlite/src/todo.rs) の例をご覧ください）。

```admonish sandbox title="実際に動く例" collapsible=true

[クリックしてCodeSandboxを開きます。](https://codesandbox.io/p/devbox/13-action-0-7-g73rl9?file=%2Fsrc%2Fmain.rs)

<noscript>
  例を表示するにはJavaScriptを有効にしてください。
</noscript>

<template>
  <iframe src="https://codesandbox.io/p/devbox/13-action-0-7-g73rl9?file=%2Fsrc%2Fmain.rs" width="100%" height="1000px" style="max-height: 100vh"></iframe>
</template>

```

<details>
<summary>CodeSandboxのソースコード</summary>

```rust
use gloo_timers::future::TimeoutFuture;
use leptos::{html::Input, prelude::*};
use uuid::Uuid;

// 非同期関数を定義する。ネットワークリクエストやデータベース読み取りなど、
// 任意の処理を行える。Resourceが読み込む非同期データであるのに対し、
// これは命令的に実行する非同期の変更処理と考えられる
async fn add_todo(text: &str) -> Uuid {
    _ = text;
    // 1秒の遅延を模擬する
    // SendWrapperによって!SendのブラウザAPIを使える。ここでは詳細を気にしなくてよい
    send_wrapper::SendWrapper::new(TimeoutFuture::new(1_000)).await;
    // 投稿IDなどを返すものとする
    Uuid::new_v4()
}

#[component]
pub fn App() -> impl IntoView {
    // Actionは引数をひとつ持つ非同期関数を受け取る
    // 引数には単純な型、構造体、または()を使える
    let add_todo = Action::new(|input: &String| {
        // 入力は参照だが、Futureが所有する必要がある
        // 'staticライフタイムを持たせるため、クローンしてFutureへmoveすることが重要
        let input = input.to_owned();
        async move { add_todo(&input).await }
    });

    // Actionは、その状態に関する情報を示す同期的でリアクティブな変数を複数提供する
    let submitted = add_todo.input();
    let pending = add_todo.pending();
    let todo_id = add_todo.value();

    let input_ref = NodeRef::<Input>::new();

    view! {
        <form
            on:submit=move |ev| {
                ev.prevent_default(); // ページを再読み込みしない……
                let input = input_ref.get().expect("input to exist");
                add_todo.dispatch(input.value());
            }
        >
            <label>
                "What do you need to do?"
                <input type="text"
                    node_ref=input_ref
                />
            </label>
            <button type="submit">"Add Todo"</button>
        </form>
        <p>{move || pending.get().then_some("Loading...")}</p>
        <p>
            "Submitted: "
            <code>{move || format!("{:#?}", submitted.get())}</code>
        </p>
        <p>
            "Pending: "
            <code>{move || format!("{:#?}", pending.get())}</code>
        </p>
        <p>
            "Todo ID: "
            <code>{move || format!("{:#?}", todo_id.get())}</code>
        </p>
    }
}

fn main() {
    leptos::mount::mount_to_body(App)
}
```

</details>
</preview>
