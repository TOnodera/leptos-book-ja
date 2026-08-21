# Resourceでデータを読み込む

Resourceは非同期タスクのリアクティブなラッパーであり、非同期の `Future` を同期的な
リアクティブシステムへ統合できます。

非同期データを読み込み、その後に同期または非同期のどちらでもリアクティブにアクセスできます。
通常の `Future` と同様にResourceを `.await` でき、その場合も追跡されます。また、解決済みなら
`Some(T)`、保留中なら `None` を返すシグナルであるかのように、`.get()` などのシグナル
アクセスメソッドを使うこともできます。

Resourceには主に `Resource` と `LocalResource` の2種類があります。本書で後ほど説明する
サーバーサイドレンダリングを使う場合は、基本的に `Resource` を使います。多くのブラウザ
APIのような `!Send` APIをクライアントサイドレンダリングで使う場合や、SSRを使いながら
ブラウザ上でしか実行できない非同期タスクがある場合は、`LocalResource` を使います。

## LocalResource

`LocalResource::new()` は、`Future` を返す「fetcher」関数をひとつだけ引数に取ります。

`Future` は `async` ブロック、`async fn` 呼び出しの結果、またはほかの任意のRustの
`Future` にできます。この関数は、これまで見てきた派生シグナルやリアクティブなクロージャと
同様に動作します。内部でシグナルを読み取ることができ、シグナルが変化するたびに関数が
再実行され、実行する新しい `Future` が作成されます。

```rust
// このcountは同期的なローカル状態
let (count, set_count) = signal(0);

// `count`を追跡し、変化するたびに`load_data`を呼び出して再読み込みする
let async_data = LocalResource::new(move || load_data(count.get()));
```

Resourceを作成すると、すぐにfetcherを呼び出して `Future` のポーリングを開始します。
非同期タスクが完了するまでResourceからの読み取りは `None` を返します。完了すると購読者へ
通知され、値は `Some(value)` になります。

Resourceを `.await` することもできます。`Future` のラッパーを作成してから `.await` するのは
無意味に思えるかもしれません。その理由は次の章で説明します。

## Resource

SSRを使う場合、ほとんどのケースで `LocalResource` ではなく `Resource` を使うべきです。

このAPIは少し異なり、`Resource::new()` は2つの関数を引数に取ります。

1. 「入力」を保持するsource関数。この入力はメモ化され、値が変化するたびにfetcherが呼び出されます。
2. source関数からデータを受け取り、`Future` を返すfetcher関数。

`LocalResource` とは異なり、`Resource` は値をサーバーからクライアントへシリアライズします。
クライアントでページを初めて読み込む際、非同期タスクを再実行する代わりに初期値が
デシリアライズされます。これは非常に重要で便利です。クライアントのWASMバンドルが
読み込まれてアプリケーションの実行を開始するのを待たず、サーバー上でデータ読み込みを
開始できるためです（詳しくは後の章で説明します）。

APIが2つの部分へ分かれているのもこのためです。*source* 関数内のシグナルは追跡されますが、
*fetcher* 内のシグナルは追跡されません。これにより、クライアントでの初回
ハイドレーション中にfetcherを再実行せず、Resourceのリアクティビティを維持できます。

同じ例で、`LocalResource` の代わりに `Resource` を使ってみましょう。

```rust
// このcountは同期的なローカル状態
let (count, set_count) = signal(0);

// Resourceを作成する
let async_data = Resource::new(
    move || count.get(),
    // `count`が変化するたびに実行される
    |count| load_data(count) 
);
```

Resourceは、ボタンクリックへの応答などでデータを手動再読み込みできる `refetch()` メソッドも
提供します。

一度だけ実行するResourceを作成するには `OnceResource` を使えます。これは `Future` をひとつ
受け取り、一度しか読み込まないことを利用した最適化を追加します。

```rust
let once = OnceResource::new(load_data(42));
```

## Resourceへアクセスする

`LocalResource` と `Resource` はどちらも `.read()`、`.with()`、`.get()` などのシグナル
アクセスメソッドを実装しますが、`T` ではなく `Option<T>` を返します。非同期データの
読み込みが完了するまでは `None` です。

```admonish sandbox title="実際に動く例" collapsible=true

[クリックしてCodeSandboxを開きます。](https://codesandbox.io/p/devbox/10-resource-0-7-q5xr9m?file=%2Fsrc%2Fmain.rs%3A7%2C30)

<noscript>
  例を表示するにはJavaScriptを有効にしてください。
</noscript>

<template>
  <iframe src="https://codesandbox.io/p/devbox/10-resource-0-7-q5xr9m?file=%2Fsrc%2Fmain.rs%3A7%2C30" width="100%" height="1000px" style="max-height: 100vh"></iframe>
</template>

```

<details>
<summary>CodeSandboxのソースコード</summary>

```rust
use gloo_timers::future::TimeoutFuture;
use leptos::prelude::*;

// 非同期関数を定義する
// ネットワークリクエストやデータベース読み取りなど、任意の処理を行えるが、
// ここでは数値を10倍するだけ
async fn load_data(value: i32) -> i32 {
    // 1秒の遅延を模擬する
    TimeoutFuture::new(1_000).await;
    value * 10
}

#[component]
pub fn App() -> impl IntoView {
    // このcountは同期的なローカル状態
    let (count, set_count) = signal(0);

    // `count`を追跡し、変化するたびに`load_data`を呼び出して再読み込みする
    let async_data = LocalResource::new(move || load_data(count.get()));

    // リアクティブデータを読み取らないResourceは一度だけ読み込まれる
    let stable = LocalResource::new(|| load_data(1));

    // .get()でResourceの値へアクセスできる
    // Futureが解決するまではリアクティブにNoneを返し、解決するとSome(T)へ更新される
    let async_result = move || {
        async_data
            .get()
            .map(|value| format!("Server returned {value:?}"))
            // この読み込み状態は初回読み込み前にだけ表示される
            .unwrap_or_else(|| "Loading...".into())
    };

    view! {
        <button
            on:click=move |_| *set_count.write() += 1
        >
            "Click me"
        </button>
        <p>
            <code>"stable"</code>": " {move || stable.get()}
        </p>
        <p>
            <code>"count"</code>": " {count}
        </p>
        <p>
            <code>"async_value"</code>": "
            {async_result}
            <br/>
        </p>
    }
}

fn main() {
    leptos::mount::mount_to_body(App)
}
```

</details>
</preview>
