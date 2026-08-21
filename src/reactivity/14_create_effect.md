# エフェクトで変化に応答する

ここまで、リアクティブシステムの半分を占めるエフェクトについて触れずに進んできました。

リアクティビティは2つの要素で動作します。個々のリアクティブな値（「シグナル」）を更新すると、
それに依存するコード（「エフェクト」）へ再実行が必要だと通知されます。この2つは相互に
依存しています。エフェクトがなければ、シグナルはリアクティブシステム内で変化しても、
外部世界とやり取りする形で観測されることはありません。シグナルがなければ、購読できる
観測可能な値がないため、エフェクトは一度実行されるだけです。エフェクトは文字通り
リアクティブシステムの「副作用」であり、リアクティブシステムを外側の非リアクティブな
世界と同期するために存在します。

レンダラーはエフェクトを使い、シグナルの変化に応じてDOMの一部を更新します。独自の
エフェクトを作成し、別の方法でリアクティブシステムと外部世界を同期することもできます。

[`Effect::new`](https://docs.rs/leptos/latest/leptos/reactive/effect/struct.Effect.html) は関数を
引数に取り、リアクティブシステムの次の「tick」で実行します（たとえばコンポーネント内で
使うと、そのコンポーネントがレンダリングされた_直後_に実行されます）。関数内で
リアクティブシグナルへアクセスすると、エフェクトがそのシグナルに依存することが登録されます。
依存するシグナルのいずれかが変化するたびに、エフェクトが再実行されます。

```rust
let (a, set_a) = signal(0);
let (b, set_b) = signal(0);

Effect::new(move |_| {
  // すぐに「Value: 0」と出力し、`a`を購読する
  logging::log!("Value: {}", a.get());
});
```

エフェクト関数は、前回の実行時に返した値を保持する引数とともに呼び出されます。初回実行時は
`None` です。

デフォルトでは、エフェクトは**サーバー上で実行されません**。そのため、エフェクト関数内で
ブラウザ固有のAPIを問題なく呼び出せます。サーバー上でも実行する必要がある場合は、
[`Effect::new_isomorphic`](https://docs.rs/leptos/latest/leptos/reactive/effect/struct.Effect.html#method.new_isomorphic) を使います。

## 自動追跡と動的な依存関係

Reactのようなフレームワークに馴染みがあれば、重要な違いに気づくでしょう。Reactなどでは
通常、エフェクトを再実行するタイミングを決める変数を明示した「依存配列」を渡す必要があります。

Leptosは同期リアクティブプログラミングの流れをくむため、この明示的な依存リストは不要です。
代わりに、エフェクト内でアクセスされたシグナルに応じて依存関係を自動的に追跡します。

これには2つの効果があります（洒落ではありません）。依存関係は次の性質を持ちます。

1. **自動的**：依存リストを保守したり、何を含めるべきか悩んだりする必要はありません。フレームワークがエフェクトを再実行させる可能性のあるシグナルを追跡し、処理します。
2. **動的**：エフェクトが実行されるたびに依存リストが消去され、更新されます。たとえば条件分岐を含む場合、現在の分岐で使われたシグナルだけが追跡されます。そのため、エフェクトの再実行回数は必要最小限になります。

> 魔法のように聞こえ、自動的な依存関係追跡の仕組みを詳しく知りたい場合は、[こちらの動画](https://www.youtube.com/watch?v=GWB3vTWeLd4)をご覧ください（音量が小さくて申し訳ありません！）。

## ほぼゼロコストの抽象化としてのエフェクト

厳密な技術的意味では、追加のメモリを使用し実行時にも存在するため「ゼロコスト抽象化」では
ありません。しかし、エフェクト内で行う高コストなAPI呼び出しなどの処理から一段高い視点で
見れば、エフェクトはゼロコストの抽象化です。記述された内容に基づき、必要最小限の回数だけ
再実行されます。

チャットソフトウェアを作成しており、利用者が氏名全体または名だけを表示でき、名前が変化する
たびにサーバーへ通知したいとします。

```rust
let (first, set_first) = signal(String::new());
let (last, set_last) = signal(String::new());
let (use_last, set_use_last) = signal(true);

// 元になるシグナルのいずれかが変化するたびに、
// 名前をログへ追加する
Effect::new(move |_| {
    logging::log!(
        "{}", if use_last.get() {
            format!("{} {}", first.get(), last.get())
        } else {
            first.get()
        },
    )
});
```

`use_last` が `true` なら、`first`、`last`、`use_last` のいずれかが変化するたびに
エフェクトが再実行されます。しかし `use_last` を `false` に切り替えると、`last` の変化が
氏名全体を変えることはありません。実際、`use_last` が再び切り替わるまで `last` は依存
リストから削除されます。`use_last` が `false` の間に `last` を何度も変更しても、APIへ
不要なリクエストを繰り返し送らずに済みます。

## エフェクトを作るべきか、作らざるべきか

エフェクトは、異なるリアクティブ値どうしを同期するためではなく、リアクティブシステムと
外側の非リアクティブな世界を同期するためのものです。つまり、あるシグナルから値を読み取り、
別のシグナルへ設定するためにエフェクトを使う方法は、常に最適ではありません。

ほかのシグナルの値に依存するシグナルを定義する必要があるなら、派生シグナルまたは
[`Memo`](https://docs.rs/leptos/latest/leptos/reactive/computed/struct.Memo.html) を使ってください。
エフェクト内でシグナルへ書き込んでも致命的な問題ではなく、コンピューターが火を噴くことも
ありません。しかし、データフローが明確になるだけでなく性能も向上するため、派生シグナルや
メモの方が常に優れています。

```rust
let (a, set_a) = signal(0);

// ⚠️ あまりよくない
let (b, set_b) = signal(0);
Effect::new(move |_| {
    set_b.set(a.get() * 2);
});

// ✅ すばらしい！
let b = move || a.get() * 2;
```

ウェブAPI、コンソール、ファイルシステム、DOMなど、リアクティブ値を外側の非リアクティブな
世界と同期する必要があるなら、エフェクト内でシグナルへ書き込むのは適切な方法です。ただし
多くの場合、実際にはエフェクトではなくイベントリスナーなどの中でシグナルへ書き込んでいる
ことに気づくでしょう。その場合は、[`leptos-use`](https://leptos-use.rs/) が目的に合う
リアクティブなラッパープリミティブをすでに提供していないか確認してください。

> `create_effect` を使うべき場合と使うべきでない場合について詳しく知りたいなら、[こちらの動画](https://www.youtube.com/watch?v=aQOFJQ2JkvQ)をご覧ください。

## エフェクトとレンダリング

ここまでエフェクトへ触れずに進められたのは、LeptosのDOMレンダラーへ組み込まれているためです。
シグナルを作成して `view` マクロへ渡すと、シグナルが変化するたびに対応するDOMノードが
更新されることはすでに確認しました。

```rust
let (count, set_count) = signal(0);

view! {
    <p>{count}</p>
}
```

これは、フレームワークが本質的に更新処理を包むエフェクトを作成することで動作します。
Leptosがこのビューを次のようなコードへ変換すると考えられます。

```rust
let (count, set_count) = signal(0);

// DOM要素を作成する
let document = leptos::document();
let p = document.create_element("p").unwrap();

// テキストをリアクティブに更新するエフェクトを作成する
Effect::new(move |prev_value| {
    // まずシグナルの値へアクセスし、文字列へ変換する
    let text = count.get().to_string();

    // 前回の値と異なる場合はノードを更新する
    if prev_value != Some(text) {
        p.set_text_content(&text);
    }

    // 次回の更新時にメモ化できるよう、この値を返す
    text
});
```

`count` が更新されるたびに、このエフェクトが再実行されます。これによってDOMを
リアクティブかつきめ細かく更新できます。

## `Effect::watch()`による明示的な追跡

Leptosは `Effect::new()` に加え、追跡する値を明示的に渡すことで、追跡処理と変化への応答を
分離できる [`Effect::watch()`](https://docs.rs/leptos/latest/leptos/reactive/effect/struct.Effect.html#method.watch) 関数を提供しています。

`watch` は3つの引数を取ります。`dependency_fn` 引数はリアクティブに追跡されますが、
`handler` と `immediate` は追跡されません。`dependency_fn` が変化するたびに `handler` が
実行されます。`immediate` がfalseの場合、`handler` は `dependency_fn` 内でアクセスした
シグナルの最初の変化が検出された後にだけ実行されます。`watch` は `Effect` を返し、
`.stop()` を呼び出すと依存関係の追跡を停止できます。

```rust
let (num, set_num) = signal(0);

let effect = Effect::watch(
    move || num.get(),
    move |num, prev_num, _| {
        leptos::logging::log!("Number: {}; Prev: {:?}", num, prev_num);
    },
    false,
);

set_num.set(1); // > "Number: 1; Prev: Some(0)"

effect.stop(); // 監視を停止する

set_num.set(2); // （何も起きない）
```

```admonish sandbox title="実際に動く例" collapsible=true

[クリックしてCodeSandboxを開きます。](https://codesandbox.io/p/devbox/14-effect-0-7-fxpy2d?file=%2Fsrc%2Fmain.rs%3A21%2C28&workspaceId=478437f3-1f86-4b1e-b665-5c27a31451fb)

<noscript>
  例を表示するにはJavaScriptを有効にしてください。
</noscript>

<template>
  <iframe src="https://codesandbox.io/p/devbox/14-effect-0-7-fxpy2d?file=%2Fsrc%2Fmain.rs%3A21%2C28&workspaceId=478437f3-1f86-4b1e-b665-5c27a31451fb" width="100%" height="1000px" style="max-height: 100vh"></iframe>
</template>

```

<details>
<summary>CodeSandboxのソースコード</summary>

```rust
use leptos::html::Input;
use leptos::prelude::*;

#[derive(Copy, Clone)]
struct LogContext(RwSignal<Vec<String>>);

#[component]
fn App() -> impl IntoView {
    // ここでは見える形のログを作るだけなので、無視してよい……
    let log = RwSignal::<Vec<String>>::new(vec![]);
    let logged = move || log.get().join("\n");

    // ここでnewtypeパターンは「必須」ではないが、よい慣習である
    // 将来追加されるかもしれない別の`RwSignal<Vec<String>>`コンテキストとの混同を防ぎ、
    // 参照しやすくなる
    provide_context(LogContext(log));

    view! {
        <CreateAnEffect/>
        <pre>{logged}</pre>
    }
}

#[component]
fn CreateAnEffect() -> impl IntoView {
    let (first, set_first) = signal(String::new());
    let (last, set_last) = signal(String::new());
    let (use_last, set_use_last) = signal(true);

    // 元になるシグナルのいずれかが変化するたびに、名前をログへ追加する
    Effect::new(move |_| {
        log(if use_last.get() {
            let first = first.read();
            let last = last.read();
            format!("{first} {last}")
        } else {
            first.get()
        })
    });

    view! {
        <h1>
            <code>"create_effect"</code>
            " Version"
        </h1>
        <form>
            <label>
                "First Name"
                <input
                    type="text"
                    name="first"
                    prop:value=first
                    on:change:target=move |ev| set_first.set(ev.target().value())
                />
            </label>
            <label>
                "Last Name"
                <input
                    type="text"
                    name="last"
                    prop:value=last
                    on:change:target=move |ev| set_last.set(ev.target().value())
                />
            </label>
            <label>
                "Show Last Name"
                <input
                    type="checkbox"
                    name="use_last"
                    prop:checked=use_last
                    on:change:target=move |ev| set_use_last.set(ev.target().checked())
                />
            </label>
        </form>
    }
}

#[component]
fn ManualVersion() -> impl IntoView {
    let first = NodeRef::<Input>::new();
    let last = NodeRef::<Input>::new();
    let use_last = NodeRef::<Input>::new();

    let mut prev_name = String::new();
    let on_change = move |_| {
        log("      listener");
        let first = first.get().unwrap();
        let last = last.get().unwrap();
        let use_last = use_last.get().unwrap();
        let this_one = if use_last.checked() {
            format!("{} {}", first.value(), last.value())
        } else {
            first.value()
        };

        if this_one != prev_name {
            log(&this_one);
            prev_name = this_one;
        }
    };

    view! {
        <h1>"Manual Version"</h1>
        <form on:change=on_change>
            <label>"First Name" <input type="text" name="first" node_ref=first/></label>
            <label>"Last Name" <input type="text" name="last" node_ref=last/></label>
            <label>
                "Show Last Name" <input type="checkbox" name="use_last" checked node_ref=use_last/>
            </label>
        </form>
    }
}

fn log(msg: impl std::fmt::Display) {
    let log = use_context::<LogContext>().unwrap().0;
    log.update(|log| log.push(msg.to_string()));
}

fn main() {
    leptos::mount::mount_to_body(App)
}
```

</details>
</preview>
