# 反復処理

Todoの一覧、表、商品画像など、項目のリストを反復処理することはWebアプリケーションでよくある作業です。変化する項目の集合どうしの差分を調整することは、フレームワークが適切に処理すべき作業のなかでも特に難しいもののひとつです。

Leptosは、項目を反復処理するための2つのパターンをサポートしています。

1. 静的なビュー：`Vec<_>`
2. 動的なリスト：`<For/>`

## `Vec<_>`を使った静的なビュー

項目を繰り返し表示する必要はあっても、元になるリスト自体はあまり変化しないことがあります。この場合、`Vec<IV> where IV: IntoView`を満たす任意の値をビューへ挿入できることを覚えておくと重要です。言い換えると、`T`をレンダリングできるなら`Vec<T>`もレンダリングできます。

```rust
let values = vec![0, 1, 2];
view! {
    // 単に"012"とレンダリングされる
    <p>{values.clone()}</p>
    // または<li>で包むこともできる
    <ul>
        {values.into_iter()
            .map(|n| view! { <li>{n}</li>})
            .collect::<Vec<_>>()}
    </ul>
}
```

Leptosは`.collect_view()`ヘルパー関数も提供しています。これを使うと、`T: IntoView`を満たす任意のイテレーターを`Vec<View>`へ収集できます。

```rust
let values = vec![0, 1, 2];
view! {
    // 単に"012"とレンダリングされる
    <p>{values.clone()}</p>
    // または<li>で包むこともできる
    <ul>
        {values.into_iter()
            .map(|n| view! { <li>{n}</li>})
            .collect_view()}
    </ul>
}
```

_リスト_が静的だからといって、インターフェースまで静的である必要はありません。静的なリストの一部として、動的な項目をレンダリングできます。

```rust
// 5個のシグナルからなるリストを作成する
let length = 5;
let counters = (1..=length).map(|idx| RwSignal::new(idx));
```

ここでは`signal()`を呼び出してreaderとwriterのタプルを取得する代わりに、`RwSignal::new()`を使って読み書き可能なひとつのシグナルを取得しています。タプルをあちこちへ渡すことになる状況では、この方が便利です。

```rust
// 各項目はリアクティブなビューを管理するが、
// リスト自体は変化しない
let counter_buttons = counters
    .map(|count| {
        view! {
            <li>
                <button
                    on:click=move |_| *count.write() += 1
                >
                    {count}
                </button>
            </li>
        }
    })
    .collect_view();

view! {
    <ul>{counter_buttons}</ul>
}
```

`Fn() -> Vec<_>`もリアクティブにレンダリング_できます_。ただし、これはキーなしリストの更新です。既存のDOM要素を再利用し、新しい`Vec<_>`内の順序に従って新しい値で更新します。リスト末尾で項目を追加・削除するだけなら適切に動作しますが、項目を移動したりリストの途中へ挿入したりすると、ブラウザに不要な処理をさせることになります。また、入力状態やCSSアニメーションなどへ予想外の影響を与える可能性もあります。「キーあり」と「キーなし」の違いや実践的な例については、[こちらの記事](https://www.stefankrause.net/wp/?p=342)を参照してください。

幸い、キーありリストを効率的に反復処理する方法もあります。

## `<For/>`コンポーネントによる動的レンダリング

[`<For/>`](https://docs.rs/leptos/latest/leptos/control_flow/fn.For.html)コンポーネントは、キー付きの動的リストです。3つのpropsを受け取ります。

- `each`：反復処理する項目`T`を返すリアクティブ関数
- `key`：`&T`を受け取り、安定した一意のキーまたはIDを返すキー関数
- `children`：各`T`をビューへレンダリングする関数

その名のとおり、`key`が鍵になります。リスト内の項目は追加、削除、移動できます。各項目のキーが時間の経過に対して安定していれば、新しく追加された項目を除いてフレームワークが再レンダリングする必要はありません。変化に応じて項目を非常に効率よく追加、削除、移動できます。そのため、最小限の追加処理でリストの変化をきわめて効率的に反映できます。

適切な`key`を作るのは少し難しい場合があります。インデックスは安定しておらず、項目を削除または移動すると変化するため、通常はこの目的に使う_べきではありません_。

一方、各行の生成時に一意のIDを生成し、それをキー関数のIDとして使う方法は適切です。

例として、以下の`<DynamicList/>`コンポーネントを確認してください。

```admonish sandbox title="Live example" collapsible=true

[クリックしてCodeSandboxを開きます。](https://codesandbox.io/p/devbox/4-iteration-0-7-dw4dfl?file=%2Fsrc%2Fmain.rs%3A1%2C1-159%2C1&workspaceId=478437f3-1f86-4b1e-b665-5c27a31451fb)

<noscript>
  例を表示するにはJavaScriptを有効にしてください。
</noscript>

<template>
  <iframe src="https://codesandbox.io/p/devbox/4-iteration-0-7-dw4dfl?file=%2Fsrc%2Fmain.rs%3A1%2C1-159%2C1&workspaceId=478437f3-1f86-4b1e-b665-5c27a31451fb" width="100%" height="1000px" style="max-height: 100vh"></iframe>
</template>

```

<details>
<summary>CodeSandboxのソースコード</summary>

```rust
use leptos::prelude::*;

// Iteration is a very common task in most applications.
// So how do you take a list of data and render it in the DOM?
// This example will show you the two ways:
// 1) for mostly-static lists, using Rust iterators
// 2) for lists that grow, shrink, or move items, using <For/>

#[component]
fn App() -> impl IntoView {
    view! {
        <h1>"Iteration"</h1>
        <h2>"Static List"</h2>
        <p>"Use this pattern if the list itself is static."</p>
        <StaticList length=5/>
        <h2>"Dynamic List"</h2>
        <p>"Use this pattern if the rows in your list will change."</p>
        <DynamicList initial_length=5/>
    }
}

/// A list of counters, without the ability
/// to add or remove any.
#[component]
fn StaticList(
    /// How many counters to include in this list.
    length: usize,
) -> impl IntoView {
    // create counter signals that start at incrementing numbers
    let counters = (1..=length).map(|idx| RwSignal::new(idx));

    // when you have a list that doesn't change, you can
    // manipulate it using ordinary Rust iterators
    // and collect it into a Vec<_> to insert it into the DOM
    let counter_buttons = counters
        .map(|count| {
            view! {
                <li>
                    <button
                        on:click=move |_| *count.write() += 1
                    >
                        {count}
                    </button>
                </li>
            }
        })
        .collect::<Vec<_>>();

    // Note that if `counter_buttons` were a reactive list
    // and its value changed, this would be very inefficient:
    // it would rerender every row every time the list changed.
    view! {
        <ul>{counter_buttons}</ul>
    }
}

/// A list of counters that allows you to add or
/// remove counters.
#[component]
fn DynamicList(
    /// The number of counters to begin with.
    initial_length: usize,
) -> impl IntoView {
    // This dynamic list will use the <For/> component.
    // <For/> is a keyed list. This means that each row
    // has a defined key. If the key does not change, the row
    // will not be re-rendered. When the list changes, only
    // the minimum number of changes will be made to the DOM.

    // `next_counter_id` will let us generate unique IDs
    // we do this by simply incrementing the ID by one
    // each time we create a counter
    let mut next_counter_id = initial_length;

    // we generate an initial list as in <StaticList/>
    // but this time we include the ID along with the signal
    // see NOTE in add_counter below re: ArcRwSignal
    let initial_counters = (0..initial_length)
        .map(|id| (id, ArcRwSignal::new(id + 1)))
        .collect::<Vec<_>>();

    // now we store that initial list in a signal
    // this way, we'll be able to modify the list over time,
    // adding and removing counters, and it will change reactively
    let (counters, set_counters) = signal(initial_counters);

    let add_counter = move |_| {
        // create a signal for the new counter
        // we use ArcRwSignal here, instead of RwSignal
        // ArcRwSignal is a reference-counted type, rather than the arena-allocated
        // signal types we've been using so far.
        // When we're creating a collection of signals like this, using ArcRwSignal
        // allows each signal to be deallocated when its row is removed.
        let sig = ArcRwSignal::new(next_counter_id + 1);
        // add this counter to the list of counters
        set_counters.update(move |counters| {
            // since `.update()` gives us `&mut T`
            // we can just use normal Vec methods like `push`
            counters.push((next_counter_id, sig))
        });
        // increment the ID so it's always unique
        next_counter_id += 1;
    };

    view! {
        <div>
            <button on:click=add_counter>
                "Add Counter"
            </button>
            <ul>
                // The <For/> component is central here
                // This allows for efficient, key list rendering
                <For
                    // `each` takes any function that returns an iterator
                    // this should usually be a signal or derived signal
                    // if it's not reactive, just render a Vec<_> instead of <For/>
                    each=move || counters.get()
                    // the key should be unique and stable for each row
                    // using an index is usually a bad idea, unless your list
                    // can only grow, because moving items around inside the list
                    // means their indices will change and they will all rerender
                    key=|counter| counter.0
                    // `children` receives each item from your `each` iterator
                    // and returns a view
                    children=move |(id, count)| {
                        // we can convert our ArcRwSignal to a Copy-able RwSignal
                        // for nicer DX when moving it into the view
                        let count = RwSignal::from(count);
                        view! {
                            <li>
                                <button
                                    on:click=move |_| *count.write() += 1
                                >
                                    {count}
                                </button>
                                <button
                                    on:click=move |_| {
                                        set_counters
                                            .write()
                                            .retain(|(counter_id, _)| {
                                                counter_id != &id
                                            });
                                    }
                                >
                                    "Remove"
                                </button>
                            </li>
                        }
                    }
                />
            </ul>
        </div>
    }
}

fn main() {
    leptos::mount::mount_to_body(App)
}
```

</details>
</preview>

### `<ForEnumerate/>`で反復処理中のインデックスへアクセスする

反復処理中にリアルタイムのインデックスへアクセスする必要がある場合のために、Leptosは[`<ForEnumerate/>`](https://docs.rs/leptos/latest/leptos/control_flow/fn.ForEnumerate.html)コンポーネントを提供しています。

propsは[`<For/>`](https://docs.rs/leptos/latest/leptos/control_flow/fn.For.html)コンポーネントと同じですが、`children`のレンダリング時にインデックスとして`ReadSignal<usize>`引数も提供されます。

```rust
#[derive(Copy, Clone, Debug, PartialEq, Eq)]
struct Counter {
  id: usize,
  count: RwSignal<i32>
}

<ForEnumerate
    each=move || counters.get() // Same as <For/>
    key=|counter| counter.id    // Same as <For/>
    // Provides the index as a signal and the child T
    children={move |index: ReadSignal<usize>, counter: Counter| {
        view! {
            <button>{move || index.get()} ". Value: " {move || counter.count.get()}</button>
        }
    }}
/>
```

より便利な`let`構文を使うこともできます。
```rust
<ForEnumerate
    each=move || counters.get() // Same as <For/>
    key=|counter| counter.id    // Same as <For/>
    let(idx, counter)           // let syntax
>
    <button>{move || idx.get()} ". Value: " {move || counter.count.get()}</button>
</ ForEnumerate>
```
