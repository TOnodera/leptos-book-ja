# 親子コンポーネント間の通信

アプリケーションは、コンポーネントが入れ子になったツリーとして考えられます。各コンポーネントは
自身のローカル状態を処理し、ユーザーインターフェイスの一部分を管理するため、比較的
自己完結する傾向があります。

しかし、親コンポーネントと子コンポーネントの間で通信したい場合もあります。たとえば、
`<button/>` へスタイルやログ出力などを追加する `<FancyButton/>` コンポーネントを
定義したとします。この `<FancyButton/>` を `<App/>` コンポーネントで使いたいのですが、
両者の間ではどうやって通信すればよいのでしょうか。

親コンポーネントから子コンポーネントへ状態を伝えるのは簡単です。その一部は
[コンポーネントとprops](./03_components.md)で説明しました。基本的には、親から子へ
情報を伝えたい場合、propsとして
[`ReadSignal`](https://docs.rs/leptos/latest/leptos/reactive/signal/struct.ReadSignal.html) または
[`Signal`](https://docs.rs/leptos/latest/leptos/reactive/wrappers/read/struct.Signal.html) の
いずれかを渡せます。

では、反対方向はどうでしょうか。子から親へイベントや状態変化の通知を送るには、どうすれば
よいのでしょうか。

Leptosにおける親子間の通信には、4つの基本パターンがあります。

## 1. [`WriteSignal`](https://docs.rs/leptos/latest/leptos/reactive/signal/struct.WriteSignal.html)を渡す

ひとつの方法は、親から子へ `WriteSignal` を渡し、子の中で更新することです。これにより、
子から親の状態を操作できます。

```rust
#[component]
pub fn App() -> impl IntoView {
    let (toggled, set_toggled) = signal(false);
    view! {
        <p>"Toggled? " {toggled}</p>
        <ButtonA setter=set_toggled/>
    }
}

#[component]
pub fn ButtonA(setter: WriteSignal<bool>) -> impl IntoView {
    view! {
        <button
            on:click=move |_| setter.update(|value| *value = !*value)
        >
            "Toggle"
        </button>
    }
}
```

このパターンは単純ですが、注意が必要です。`WriteSignal` をあちこちへ渡すと、コードの
挙動を把握しにくくなる可能性があります。この例では、`<App/>` を読めば `toggled` を
変更する権限を渡していることは明らかですが、それがいつ、どのように変化するかはまったく
明らかではありません。このような小さな局所的な例なら理解は簡単です。しかし、コード全体で
このように `WriteSignal` を渡しているなら、スパゲッティコードを書きやすくしすぎて
いないか、よく検討するべきです。

## 2. コールバックを使う

別の方法は、`on_click` などのコールバックを子へ渡すことです。

```rust
#[component]
pub fn App() -> impl IntoView {
    let (toggled, set_toggled) = signal(false);
    view! {
        <p>"Toggled? " {toggled}</p>
        <ButtonB on_click=move |_| set_toggled.update(|value| *value = !*value)/>
    }
}

#[component]
pub fn ButtonB(on_click: impl FnMut(MouseEvent) + 'static) -> impl IntoView {
    view! {
        <button on:click=on_click>
            "Toggle"
        </button>
    }
}
```

`<ButtonA/>` は `WriteSignal` を受け取り、変更方法を自身で決めていたのに対し、
`<ButtonB/>` は単にイベントを発生させるだけです。実際の変更は `<App/>` 側で行われます。
これには、ローカル状態をローカルに保ち、変更処理がスパゲッティ状になるのを防げる利点が
あります。一方、シグナルを変更するロジックは `<ButtonB/>` ではなく `<App/>` に置く
必要があります。これは現実に存在するトレードオフであり、単純な正解・不正解の問題では
ありません。

## 3. イベントリスナーを使う

実は、選択肢2は少し違う方法でも記述できます。コールバックがネイティブDOMイベントへ
直接対応するなら、`<App/>` の `view` マクロ内でコンポーネントを使う場所へ、`on:`
リスナーを直接追加できます。

```rust
#[component]
pub fn App() -> impl IntoView {
    let (toggled, set_toggled) = signal(false);
    view! {
        <p>"Toggled? " {toggled}</p>
        // on_clickではなくon:clickであることに注目
        // HTML要素のイベントリスナーと同じ構文である
        <ButtonC on:click=move |_| set_toggled.update(|value| *value = !*value)/>
    }
}

#[component]
pub fn ButtonC() -> impl IntoView {
    view! {
        <button>"Toggle"</button>
    }
}
```

これにより、`<ButtonB/>` よりも `<ButtonC/>` のコードを大幅に減らしながら、正しく
型付けされたイベントをリスナーへ渡せます。この仕組みでは、`<ButtonC/>` が返す各要素へ
`on:` イベントリスナーを追加します。この場合は、ひとつの `<button>` だけです。

もちろん、この方法が使えるのは、コンポーネント内でレンダリングする要素へ直接渡す、実際の
DOMイベントだけです。要素へ直接対応しない複雑なロジック（たとえば `<ValidatedForm/>` を
作成して `on_valid_form_submit` コールバックが必要な場合）には、選択肢2を使ってください。

## 4. コンテキストを提供する

これは実際には選択肢1の変形です。深くネストしたコンポーネントツリーがあるとします。

```rust
#[component]
pub fn App() -> impl IntoView {
    let (toggled, set_toggled) = signal(false);
    view! {
        <p>"Toggled? " {toggled}</p>
        <Layout/>
    }
}

#[component]
pub fn Layout() -> impl IntoView {
    view! {
        <header>
            <h1>"My Page"</h1>
        </header>
        <main>
            <Content/>
        </main>
    }
}

#[component]
pub fn Content() -> impl IntoView {
    view! {
        <div class="content">
            <ButtonD/>
        </div>
    }
}

#[component]
pub fn ButtonD() -> impl IntoView {
    todo!()
}

```

ここでは `<ButtonD/>` が `<App/>` の直接の子ではなくなったため、そのpropsへ単純に
`WriteSignal` を渡すことはできません。「prop drilling」と呼ばれることがある方法、つまり
両者の間にある各階層へpropsを追加することはできます。

```rust
#[component]
pub fn App() -> impl IntoView {
    let (toggled, set_toggled) = signal(false);
    view! {
        <p>"Toggled? " {toggled}</p>
        <Layout set_toggled/>
    }
}

#[component]
pub fn Layout(set_toggled: WriteSignal<bool>) -> impl IntoView {
    view! {
        <header>
            <h1>"My Page"</h1>
        </header>
        <main>
            <Content set_toggled/>
        </main>
    }
}

#[component]
pub fn Content(set_toggled: WriteSignal<bool>) -> impl IntoView {
    view! {
        <div class="content">
            <ButtonD set_toggled/>
        </div>
    }
}

#[component]
pub fn ButtonD(set_toggled: WriteSignal<bool>) -> impl IntoView {
    todo!()
}
```

これは煩雑です。`<Layout/>` と `<Content/>` は `set_toggled` を必要としておらず、単に
`<ButtonD/>` へ受け渡しているだけです。それでもpropsを3回宣言しなければなりません。
面倒なだけでなく、保守も困難です。「半分切り替えた」選択肢を追加し、`set_toggled` の型を
列挙型へ変更する必要が生じた場合を想像してください。3か所も変更しなければなりません！

途中の階層を飛び越える方法はないのでしょうか。

あります！

### 4.1 コンテキストAPI

[`provide_context`](https://docs.rs/leptos/latest/leptos/context/fn.provide_context.html) と
[`use_context`](https://docs.rs/leptos/latest/leptos/context/fn.use_context.html) を使うと、
階層を飛び越えてデータを提供できます。コンテキストは提供するデータの型（この例では
`WriteSignal<bool>`）によって識別され、UIツリーの形に沿った上から下へのツリー内に
存在します。この例ではコンテキストを使い、不要なprop drillingを省けます。

```rust
#[component]
pub fn App() -> impl IntoView {
    let (toggled, set_toggled) = signal(false);

    // `set_toggled`をこのコンポーネントのすべての子と共有する
    provide_context(set_toggled);

    view! {
        <p>"Toggled? " {toggled}</p>
        <Layout/>
    }
}

// <Layout/>と<Content/>は省略
// このバージョンでは、それぞれから`set_toggled`引数を削除する

#[component]
pub fn ButtonD() -> impl IntoView {
    // use_contextはコンテキストツリーを上へたどり、
    // `WriteSignal<bool>`を探す
    // ここでは提供済みだとわかっているため.expect()を使う
    let setter = use_context::<WriteSignal<bool>>().expect("to have found the setter provided");

    view! {
        <button
            on:click=move |_| setter.update(|value| *value = !*value)
        >
            "Toggle"
        </button>
    }
}

```

これにも `<ButtonA/>` と同じ注意点があります。`WriteSignal` を渡すとコード内の任意の
場所から状態を変更できるため、慎重に行うべきです。しかし注意して使えば、Leptosにおける
グローバル状態管理の最も効果的な手法のひとつになります。必要となる最上位の階層で状態を
提供し、その下にある必要な場所で使うだけです。

この方法に性能上の欠点はありません。きめ細かなリアクティブシグナルを渡しているため、
更新時に途中のコンポーネント（`<Layout/>` と `<Content/>`）では_何も起こりません_。
`<ButtonD/>` と `<App/>` の間で直接通信しています。実際、これこそがきめ細かな
リアクティビティの力ですが、`<ButtonD/>` 内のボタンクリックと `<App/>` 内のひとつの
テキストノードが直接通信しています。コンポーネント自体がまったく存在しないかのようです。
そして実のところ……実行時には存在しません。末端までシグナルとエフェクトだけです。

この方法には重要なトレードオフがあります。`provide_context` と `use_context` の間では、
型安全性が失われます。子コンポーネントが正しいコンテキストを受け取るかどうかは実行時に
検査されます（`use_context.expect(...)` を参照）。先ほどまでの方法とは異なり、
リファクタリング時にコンパイラの支援は受けられません。

```admonish sandbox title="実際に動く例" collapsible=true

[クリックしてCodeSandboxを開きます。](https://codesandbox.io/p/devbox/8-parent-child-0-7-cgcgk9?file=%2Fsrc%2Fmain.rs%3A1%2C1-116%2C2&workspaceId=478437f3-1f86-4b1e-b665-5c27a31451fb)

<noscript>
  例を表示するにはJavaScriptを有効にしてください。
</noscript>

<template>
  <iframe src="https://codesandbox.io/p/devbox/8-parent-child-0-7-cgcgk9?file=%2Fsrc%2Fmain.rs%3A1%2C1-116%2C2&workspaceId=478437f3-1f86-4b1e-b665-5c27a31451fb" width="100%" height="1000px" style="max-height: 100vh"></iframe>
</template>

```

<details>
<summary>CodeSandboxのソースコード</summary>

```rust
use leptos::{ev::MouseEvent, prelude::*};

// 子コンポーネントが親と通信する4つの方法を示す
// 1) <ButtonA/>：子コンポーネントのpropsとしてWriteSignalを渡し、
//    子が書き込んで親が読み取る
// 2) <ButtonB/>：子コンポーネントのpropsとしてクロージャを渡し、子が呼び出す
// 3) <ButtonC/>：コンポーネントへ`on:`イベントリスナーを追加する
// 4) <ButtonD/>：prop drillingの代わりに、コンポーネントで使うコンテキストを提供する

#[derive(Copy, Clone)]
struct SmallcapsContext(WriteSignal<bool>);

#[component]
pub fn App() -> impl IntoView {
    // <p>の4つのクラスを切り替えるシグナル
    let (red, set_red) = signal(false);
    let (right, set_right) = signal(false);
    let (italics, set_italics) = signal(false);
    let (smallcaps, set_smallcaps) = signal(false);

    // ここでnewtypeパターンは「必須」ではないが、よい慣習である
    // 将来追加されるかもしれない別の`WriteSignal<bool>`コンテキストとの混同を防ぎ、
    // ButtonDから参照しやすくなる
    provide_context(SmallcapsContext(set_smallcaps));

    view! {
        <main>
            <p
                // class:属性はF: Fn() => boolを受け取り、これらのシグナルはすべてFn()を実装する
                class:red=red
                class:right=right
                class:italics=italics
                class:smallcaps=smallcaps
            >
                "Lorem ipsum sit dolor amet."
            </p>

            // ボタンA：シグナルのsetterを渡す
            <ButtonA setter=set_red/>

            // ボタンB：クロージャを渡す
            <ButtonB on_click=move |_| set_right.update(|value| *value = !*value)/>

            // ボタンC：通常のイベントリスナーを使う
            // このようにコンポーネントへイベントリスナーを設定すると、
            // コンポーネントが返す最上位の各要素へ適用される
            <ButtonC on:click=move |_| set_italics.update(|value| *value = !*value)/>

            // ボタンDはpropsではなくコンテキストからsetterを取得する
            <ButtonD/>
        </main>
    }
}

/// ボタンAはシグナルのsetterを受け取り、自身でシグナルを更新する
#[component]
pub fn ButtonA(
    /// ボタンのクリック時に切り替えるシグナル。
    setter: WriteSignal<bool>,
) -> impl IntoView {
    view! {
        <button
            on:click=move |_| setter.update(|value| *value = !*value)
        >
            "Toggle Red"
        </button>
    }
}

/// ボタンBはクロージャを受け取る
#[component]
pub fn ButtonB(
    /// ボタンのクリック時に呼び出されるコールバック。
    on_click: impl FnMut(MouseEvent) + 'static,
) -> impl IntoView
{
    view! {
        <button
            on:click=on_click
        >
            "Toggle Right"
        </button>
    }
}

/// ボタンCはダミーである。ボタンをレンダリングするが、クリックは処理しない。
/// 代わりに親コンポーネントがイベントリスナーを追加する。
#[component]
pub fn ButtonC() -> impl IntoView {
    view! {
        <button>
            "Toggle Italics"
        </button>
    }
}

/// ボタンDはボタンAとよく似ているが、setterをpropsとして渡す代わりに
/// コンテキストから取得する
#[component]
pub fn ButtonD() -> impl IntoView {
    let setter = use_context::<SmallcapsContext>().unwrap().0;

    view! {
        <button
            on:click=move |_| setter.update(|value| *value = !*value)
        >
            "Toggle Small Caps"
        </button>
    }
}

fn main() {
    leptos::mount::mount_to_body(App)
}
```

</details>
</preview>
