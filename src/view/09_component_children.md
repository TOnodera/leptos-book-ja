# コンポーネントの子要素

HTML要素へ子要素を渡すのと同じように、コンポーネントへ子要素を渡したいことはよくあります。
たとえば、HTMLの `<form>` を拡張する `<FancyForm/>` コンポーネントがあるとします。
そこへすべての入力を渡す方法が必要です。

```rust
view! {
    <FancyForm>
        <fieldset>
            <label>
                "Some Input"
                <input type="text" name="something"/>
            </label>
        </fieldset>
        <button>"Submit"</button>
    </FancyForm>
}
```

Leptosではどうすればよいのでしょうか。コンポーネントを別のコンポーネントへ渡す方法は、
基本的に2つあります。

1. **render props**：ビューを返す関数となるプロパティ
2. **`children`** props：コンポーネントの子として渡したものをすべて含む、特別な
   コンポーネントプロパティ

実は、[`<Show/>`](/view/06_control_flow.html#show) コンポーネントですでに両方を
使っています。

```rust
view! {
  <Show
    // `when`は通常のprops
    when=move || value.get() > 5
    // `fallback`は「render props」：ビューを返す関数
    fallback=|| view! { <Small/> }
  >
    // `<Big/>`（およびここにあるほかのすべて）は
    // `children` propsへ渡される
    <Big/>
  </Show>
}
```

子要素とrender propsを受け取るコンポーネントを定義しましょう。

```rust
/// マークアップ内に`render_prop`と子要素を表示する。
#[component]
pub fn TakesChildren<F, IV>(
    /// View（IV型）へ変換できるものを返す関数（F型）を受け取る
    render_prop: F,
    /// `children`はいくつかの型のいずれかを受け取れる。
    /// どの型も何らかのビュー型を返す関数である
    children: Children,
) -> impl IntoView
where
    F: Fn() -> IV,
    IV: IntoView,
{
    view! {
        <h1><code>"<TakesChildren/>"</code></h1>
        <h2>"Render Prop"</h2>
        {render_prop()}
        <hr/>
        <h2>"Children"</h2>
        {children()}
    }
}
```

`render_prop` と `children` はどちらも関数なので、呼び出して適切なビューを生成できます。
特に `Children` は `Box<dyn FnOnce() -> AnyView>` のエイリアスです（代わりに `Children`
という名前が付いていてよかったと思いませんか？）。ここで返される `AnyView` は、不透明で
型消去されたビューです。その中身を調べることはできません。子要素の型にはほかにもさまざまな
ものがあります。たとえば `ChildrenFragment` は、子要素を反復処理できるコレクションである
`Fragment` を返します。

> `children` を複数回呼び出すために `Fn` や `FnMut` が必要な場合は、`ChildrenFn` と
> `ChildrenMut` のエイリアスも用意されています。

このコンポーネントは次のように使えます。

```rust
view! {
    <TakesChildren render_prop=|| view! { <p>"Hi, there!"</p> }>
        // これらが`children`へ渡される
        "Some text"
        <span>"A span"</span>
    </TakesChildren>
}
```

## 型付きの子要素：スロット

ここまでは `children` propsをひとつだけ持つコンポーネントについて説明してきました。
しかし、型の異なる複数の子要素を持つコンポーネントが役立つ場合もあります。たとえば次の
ようなものです。
```rust
view! {
    <If condition=a_is_true>
        <Then>"Show content when a is true"</Then>
        <ElseIf condition=b_is_true>"b is true"</ElseIf>
        <ElseIf condition=c_is_true>"c is true"</ElseIf>
        <Else>"None of the above are true"</Else>
    </If>
}
```
`If` コンポーネントは常に `Then` の子要素を必要とし、任意で複数の `ElseIf` 子要素と、
ひとつの `Else` 子要素を受け取ります。これを扱うため、Leptosは
[スロット](https://docs.rs/leptos/latest/leptos/attr.slot.html)を提供しています。

`#[slot]` マクロは通常のRust構造体へ注釈を付け、コンポーネントのスロットにします。
```rust
// `#[slot]`で注釈を付けた、子要素を受け取る単純な構造体
#[slot]
struct Then {
    children: ChildrenFn,
}
```

このスロットは、コンポーネント内でpropsとして使用できます。
```rust
#[component]
fn If(
    condition: Signal<bool>,
    // コンポーネントのスロット。<Then slot>構文で渡す
    then_slot: Then,
) -> impl IntoView {
    move || {
        if condition.get() {
            (then_slot.children)().into_any()
        } else {
            ().into_any()
        }
    }
}
```

これで `If` コンポーネントは `Then` 型の子要素を必要とします。使用するスロットには
`slot:<prop_name>` で注釈を付ける必要があります。
```rust
view! {
    <If condition=a_is_true>
        // `If`コンポーネントは`then_slot`として常に`Then`の子要素を必要とする
        <Then slot:then_slot>"Show content when a is true"</Then>
    </If>
}
```

> 名前を付けずに `slot` を指定すると、構造体名をスネークケースにしたものが選択される
> スロットのデフォルトになります。そのため、この場合の `<Then slot>` は
> `<Then slot:then>` と同じです。

完全な例は[スロットの例](https://github.com/leptos-rs/leptos/tree/main/examples/slots)を
ご覧ください。

### スロットのイベントハンドラー

次のように、イベントハンドラーをスロットへ直接指定することはできません。
```rust
<ComponentWithSlot>
    // ⚠️ スロットへイベントハンドラー`on:click`を直接指定することはできない
    <SlotWithChildren slot:slot on:click=move |_| {}> 
        <h1>"Hello, World!"</h1>
    </SlotWithChildren>
</ComponentWithSlot>
```

代わりに、スロットの内容を通常の要素で包み、その要素へイベントハンドラーを追加します。
```rust
<ComponentWithSlot>
    <SlotWithChildren slot:slot>
        // ✅ イベントハンドラーはスロットへ直接定義されていない
        <div on:click=move |_| {}>
            <h1>"Hello, World!"</h1>
        </div>
    </SlotWithChildren>
</ComponentWithSlot>
```

## 子要素を操作する

[`Fragment`](https://docs.rs/leptos/latest/leptos/tachys/view/fragment/struct.Fragment.html) 型は、
基本的に `Vec<AnyView>` を包むためのものです。ビュー内のどこへでも挿入できます。

また、内部のビューへ直接アクセスして操作することもできます。たとえば、次のコンポーネントは
子要素を受け取り、順序なしリストへ変換します。

```rust
/// 各子要素を`<li>`で包み、`<ul>`へ埋め込む。
#[component]
pub fn WrapsChildren(children: ChildrenFragment) -> impl IntoView {
    // children()は`Fragment`を返し、その`nodes`フィールドには
    // Vec<View>が格納されている
    // つまり、子要素を反復処理して新しいものを作成できる！
    let children = children()
        .nodes
        .into_iter()
        .map(|child| view! { <li>{child}</li> })
        .collect::<Vec<_>>();

    view! {
        <h1><code>"<WrapsChildren/>"</code></h1>
        // 包んだ子要素をULで包む
        <ul>{children}</ul>
    }
}
```

次のように呼び出すと、リストが作成されます。

```rust
view! {
    <WrapsChildren>
        "A"
        "B"
        "C"
    </WrapsChildren>
}
```

```admonish sandbox title="実際に動く例" collapsible=true

[クリックしてCodeSandboxを開きます。](https://codesandbox.io/p/devbox/9-component-children-0-7-736s9r?file=%2Fsrc%2Fmain.rs%3A1%2C1-90%2C2&workspaceId=478437f3-1f86-4b1e-b665-5c27a31451fb)

<noscript>
  例を表示するにはJavaScriptを有効にしてください。
</noscript>

<template>
  <iframe src="https://codesandbox.io/p/devbox/9-component-children-0-7-736s9r?file=%2Fsrc%2Fmain.rs%3A1%2C1-90%2C2&workspaceId=478437f3-1f86-4b1e-b665-5c27a31451fb" width="100%" height="1000px" style="max-height: 100vh"></iframe>
</template>

```

<details>
<summary>CodeSandboxのソースコード</summary>

```rust
use leptos::prelude::*;

// ある種の子ビューを別のコンポーネントへ渡したいことはよくある。
// これには2つの基本パターンがある。
// - 「render props」：ビューを作成する関数を受け取る
//   コンポーネントpropsを作成する
// - `children` props：プロパティとしてではなく、ビュー内でコンポーネントの
//   子として渡された内容を保持する特別なプロパティ

#[component]
pub fn App() -> impl IntoView {
    let (items, set_items) = signal(vec![0, 1, 2]);
    let render_prop = move || {
        let len = move || items.read().len();
        view! {
            <p>"Length: " {len}</p>
        }
    };

    view! {
        // このコンポーネントは2種類の子要素を別のマークアップへ埋め込み、表示するだけ
        <TakesChildren
            // コンポーネントのpropsでは、`render_prop=render_prop`を
            // `render_prop`と省略できる
            // （HTML要素の属性では使えない）
            render_prop
        >
            // HTML要素の子要素とまったく同じように見える
            <p>"Here's a child."</p>
            <p>"Here's another child."</p>
        </TakesChildren>
        <hr/>
        // このコンポーネントは実際に子要素を反復処理して包む
        <WrapsChildren>
            <p>"Here's a child."</p>
            <p>"Here's another child."</p>
        </WrapsChildren>
    }
}

/// マークアップ内に`render_prop`と子要素を表示する。
#[component]
pub fn TakesChildren<F, IV>(
    /// View（IV型）へ変換できるものを返す関数（F型）を受け取る
    render_prop: F,
    /// `children`は`Children`型を受け取る
    /// これは`Box<dyn FnOnce() -> Fragment>`のエイリアスである
    /// ……代わりに`Children`という名前が付いていてよかったと思わないだろうか？
    children: Children,
) -> impl IntoView
where
    F: Fn() -> IV,
    IV: IntoView,
{
    view! {
        <h1><code>"<TakesChildren/>"</code></h1>
        <h2>"Render Prop"</h2>
        {render_prop()}
        <hr/>
        <h2>"Children"</h2>
        {children()}
    }
}

/// 各子要素を`<li>`で包み、`<ul>`へ埋め込む。
#[component]
pub fn WrapsChildren(children: ChildrenFragment) -> impl IntoView {
    // children()は`Fragment`を返し、その`nodes`フィールドには
    // Vec<View>が格納されている
    // つまり、子要素を反復処理して新しいものを作成できる！
    let children = children()
        .nodes
        .into_iter()
        .map(|child| view! { <li>{child}</li> })
        .collect::<Vec<_>>();

    view! {
        <h1><code>"<WrapsChildren/>"</code></h1>
        // 包んだ子要素をULで包む
        <ul>{children}</ul>
    }
}

fn main() {
    leptos::mount::mount_to_body(App)
}
```

</details>
</preview>
