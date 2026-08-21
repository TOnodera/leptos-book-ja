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

// 反復処理は、ほとんどのアプリケーションでよくある作業である
// データのリストをDOMへレンダリングする2つの方法を示す
// 1）ほぼ静的なリストではRustのイテレーターを使う
// 2）項目が増減・移動するリストでは<For/>を使う

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

/// 追加も削除もできないカウンターのリスト。
#[component]
fn StaticList(
    /// リストに含めるカウンターの数。
    length: usize,
) -> impl IntoView {
    // 連続する数値から始まるカウンターシグナルを作成する
    let counters = (1..=length).map(|idx| RwSignal::new(idx));

    // 変化しないリストは通常のRustイテレーターで操作し、
    // Vec<_>へ収集してDOMへ挿入できる
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

    // `counter_buttons`がリアクティブなリストで値が変化する場合、
    // リストが変わるたびに全行を再レンダリングするため非常に非効率になる
    view! {
        <ul>{counter_buttons}</ul>
    }
}

/// カウンターを追加・削除できるリスト。
#[component]
fn DynamicList(
    /// 最初に用意するカウンターの数。
    initial_length: usize,
) -> impl IntoView {
    // この動的リストでは<For/>コンポーネントを使う
    // <For/>はキー付きリストであり、各行にキーが定義される
    // キーが変化しなければ行は再レンダリングされない
    // リストが変化したとき、DOMには最小限の変更だけが加えられる

    // `next_counter_id`を使って一意のIDを生成する
    // カウンターを作成するたびにIDを1増やす
    let mut next_counter_id = initial_length;

    // <StaticList/>と同様に初期リストを生成するが、
    // 今回はシグナルとともにIDも含める
    // ArcRwSignalについては下のadd_counter内の注意を参照
    let initial_counters = (0..initial_length)
        .map(|id| (id, ArcRwSignal::new(id + 1)))
        .collect::<Vec<_>>();

    // 初期リストをシグナルへ保存する
    // これにより、カウンターを追加・削除してリストを
    // 時間とともに変更でき、リアクティブに反映される
    let (counters, set_counters) = signal(initial_counters);

    let add_counter = move |_| {
        // 新しいカウンターのシグナルを作成する
        // ここではRwSignalではなくArcRwSignalを使う
        // ArcRwSignalは、これまで使ってきたアリーナ割り当てのシグナル型ではなく、
        // 参照カウント方式の型である
        // このようなシグナルのコレクションでは、ArcRwSignalを使うことで
        // 行の削除時に各シグナルも解放できる
        let sig = ArcRwSignal::new(next_counter_id + 1);
        // カウンターをリストへ追加する
        set_counters.update(move |counters| {
            // `.update()`は`&mut T`を返すので、
            // `push`など通常のVecメソッドを使える
            counters.push((next_counter_id, sig))
        });
        // 常に一意になるようIDを増やす
        next_counter_id += 1;
    };

    view! {
        <div>
            <button on:click=add_counter>
                "Add Counter"
            </button>
            <ul>
                // ここでは<For/>コンポーネントが中心になる
                // キー付きリストを効率的にレンダリングできる
                <For
                    // `each`はイテレーターを返す任意の関数を受け取る
                    // 通常はシグナルまたは派生シグナルにする
                    // リアクティブでなければ<For/>ではなくVec<_>をレンダリングする
                    each=move || counters.get()
                    // キーは各行に対して一意かつ安定している必要がある
                    // リストが増えるだけの場合を除き、インデックスの使用は通常適切でない
                    // リスト内で項目を移動するとインデックスが変化し、すべて再レンダリングされるためである
                    key=|counter| counter.0
                    // `children`は`each`イテレーターの各項目を受け取り、ビューを返す
                    children=move |(id, count)| {
                        // ArcRwSignalをCopy可能なRwSignalへ変換し、
                        // ビューへmoveするときの開発体験を改善する
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
    // インデックスをシグナルとして、子をTとして渡す
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
