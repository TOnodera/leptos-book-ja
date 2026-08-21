# 制御フロー

ほとんどのアプリケーションでは、ときどき判断を下す必要があります。ビューのこの部分を
レンダリングするべきか、しないべきか。`<ButtonA/>` と `<WidgetB/>` のどちらを
レンダリングするべきか。これが**制御フロー**です。

## いくつかのヒント

Leptosで制御フローを扱う方法を考える際には、いくつか覚えておくべきことがあります。

1. Rustは式指向の言語です。`if x() { y } else { z }` や `match x() { ... }` といった
   制御フロー式は値を返します。この性質は、宣言的なユーザーインターフェイスで非常に
   役立ちます。
2. `IntoView` を実装する任意の `T`、言い換えればLeptosがレンダリング方法を知っている
   任意の型について、`Option<T>` と `Result<T, impl Error>` も `IntoView` を実装します。
   また、`Fn() -> T` がリアクティブな `T` をレンダリングするのと同様に、
   `Fn() -> Option<T>` と `Fn() -> Result<T, impl Error>` もリアクティブです。
3. Rustには [Option::map](https://doc.rust-lang.org/std/option/enum.Option.html#method.map)、
   [Option::and_then](https://doc.rust-lang.org/std/option/enum.Option.html#method.and_then),
   [Option::ok_or](https://doc.rust-lang.org/std/option/enum.Option.html#method.ok_or),
   [Result::map](https://doc.rust-lang.org/std/result/enum.Result.html#method.map),
   [Result::ok](https://doc.rust-lang.org/std/result/enum.Result.html#method.ok), and
   [bool::then](https://doc.rust-lang.org/std/primitive.bool.html#method.then) that
   など、いくつかの標準型を宣言的に相互変換できる便利なヘルパーが多数あります。これらの
   型はすべてレンダリングできます。特に `Option` と `Result` のドキュメントをじっくり
   読むことは、Rustの腕を磨く最良の方法のひとつです。
4. そして常に覚えておいてください。リアクティブにするには、値が関数でなければなりません。
   以下では、頻繁に値を `move ||` クロージャで包みます。依存するシグナルが変化したときに
   確実に再実行され、UIのリアクティビティが保たれるようにするためです。

## それが何を意味するのか

話をつなげると、これは制御フロー専用のコンポーネントや特別な知識がなくても、通常の
Rustコードだけでほとんどの制御フローを実装できるということです。

たとえば、単純なシグナルと派生シグナルから始めましょう。

```rust
let (value, set_value) = signal(0);
let is_odd = move || value.get() % 2 != 0;
```

これらのシグナルと通常のRustを使って、ほとんどの制御フローを構築できます。

### `if`文

数値が奇数ならあるテキストを、偶数なら別のテキストをレンダリングしたいとします。
次のコードはどうでしょうか。

```rust
view! {
    <p>
        {move || if is_odd() {
            "Odd"
        } else {
            "Even"
        }}
    </p>
}
```

`if` 式は値を返し、`&str` は `IntoView` を実装しています。そのため `Fn() -> &str` も
`IntoView` を実装し、このコードは……そのまま動作します！

### `Option<T>`

奇数ならテキストをレンダリングし、偶数なら何も表示したくないとします。

```rust
let message = move || {
    if is_odd() {
        Some("Ding ding ding!")
    } else {
        None
    }
};

view! {
    <p>{message}</p>
}
```

これは正しく動作します。`bool::then()` を使えば、もう少し短くできます。

```rust
let message = move || is_odd().then(|| "Ding ding ding!");
view! {
    <p>{message}</p>
}
```

必要ならインラインにすることもできます。ただ個人的には、`view` の外へ取り出した方が
`cargo fmt` と `rust-analyzer` のサポートが充実するので、そちらを好むこともあります。

### `match`文

ここでも通常のRustコードを書いているだけです。そのため、Rustの強力なパターンマッチングを
存分に利用できます。

```rust
let message = move || {
    match value.get() {
        0 => "Zero",
        1 => "One",
        n if is_odd() => "Odd",
        _ => "Even"
    }
};
view! {
    <p>{message}</p>
}
```

使わない手はありません。人生は一度きりですからね。

## 過剰なレンダリングを防ぐ

とはいえ、何でもありというわけではありません。

ここまで行ってきたことは基本的に問題ありません。ただし、覚えて注意しておくべき点が
ひとつあります。ここまで作成した制御フロー関数は、どれも本質的には派生シグナルです。
値が変化するたびに再実行されます。上の例では、値が変化するたびに偶数と奇数が切り替わる
ため、これで問題ありません。

しかし、次の例を考えてみてください。

```rust
let (value, set_value) = signal(0);

let message = move || if value.get() > 5 {
    "Big"
} else {
    "Small"
};

view! {
    <p>{message}</p>
}
```

これは確かに_動作します_。しかしログを追加すると、驚くかもしれません。

```rust
let message = move || if value.get() > 5 {
    logging::log!("{}: rendering Big", value.get());
    "Big"
} else {
    logging::log!("{}: rendering Small", value.get());
    "Small"
};
```

利用者が `value` を増加させるボタンを繰り返しクリックすると、次のようなログが表示されます。

```
1: rendering Small
2: rendering Small
3: rendering Small
4: rendering Small
5: rendering Small
6: rendering Big
7: rendering Big
8: rendering Big
... ad infinitum
```

`value` が変化するたびに `if` 文が再実行されます。リアクティビティの仕組みを考えれば
自然な動作ですが、欠点もあります。単純なテキストノードなら、`if` 文を再実行して
再レンダリングしても大きな問題にはなりません。しかし、次のような場合を想像してください。

```rust
let message = move || if value.get() > 5 {
    <Big/>
} else {
    <Small/>
};
```

このコードは `<Small/>` を5回再レンダリングし、その後は `<Big/>` を際限なく
再レンダリングします。リソースの読み込みやシグナルの作成、あるいは単にDOMノードの作成を
行うコンポーネントなら、これは不要な処理です。

### `<Show/>`

これを解決するのが
[`<Show/>`](https://docs.rs/leptos/latest/leptos/control_flow/fn.Show.html) コンポーネントです。
条件を表す `when` 関数、`when` 関数が `false` を返したときに表示する `fallback`、そして
`when` が `true` のときにレンダリングする子要素を渡します。

```rust
let (value, set_value) = signal(0);

view! {
  <Show
    when=move || { value.get() > 5 }
    fallback=|| view! { <Small/> }
  >
    <Big/>
  </Show>
}
```

`<Show/>` は `when` 条件をメモ化するため、`<Small/>` を一度だけレンダリングし、`value` が
5より大きくなるまで同じコンポーネントを表示し続けます。その後 `<Big/>` を一度だけ
レンダリングし、以後ずっと表示するか、`value` が5以下になった時点で `<Small/>` を
もう一度レンダリングします。

これは、動的な `if` 式を使うときの再レンダリングを避ける便利な道具です。例によって多少の
オーバーヘッドはあります。ひとつのテキストノード、クラス、属性の更新など非常に単純な
ノードでは、`move || if ...` の方が効率的です。しかし、どちらかの分岐のレンダリングに
少しでもコストがかかるなら、`<Show/>` を使いましょう。

## 注意：型変換

この節の最後に、もうひとつ重要なことがあります。

Leptosは静的に型付けされたビューツリーを使います。`view` マクロはビューの種類に応じて
異なる型を返します。

次のコードはコンパイルできません。異なるHTML要素は異なる型だからです。

```rust,compile_error
view! {
    <main>
        {move || match is_odd() {
            true if value.get() == 1 => {
                view! { <pre>"One"</pre> }
            },
            false if value.get() == 2 => {
                view! { <p>"Two"</p> }
            }
            // HtmlElement<Textarea>を返す
            _ => view! { <textarea>{value.get()}</textarea> }
        }}
    </main>
}
```

この強い型付けは、さまざまなコンパイル時最適化を可能にするため非常に強力です。しかし、
Rustでは条件の各分岐から異なる型を返せないため、このような条件ロジックでは少し不便な
場合があります。この状況を解決する方法は2つあります。

1. 列挙型 `Either`（および `EitherOf3`、`EitherOf4` など）を使って、異なる型を同じ型へ変換する。
2. `.into_any()` を使って、複数の型を型消去されたひとつの `AnyView` へ変換する。

先ほどと同じ例へ変換を追加すると、次のようになります。

```rust,compile_error
view! {
    <main>
        {move || match is_odd() {
            true if value.get() == 1 => {
                // HtmlElement<Pre>を返す
                view! { <pre>"One"</pre> }.into_any()
            },
            false if value.get() == 2 => {
                // HtmlElement<P>を返す
                view! { <p>"Two"</p> }.into_any()
            }
            // HtmlElement<Textarea>を返す
            _ => view! { <textarea>{value.get()}</textarea> }.into_any()
        }}
    </main>
}
```

```admonish sandbox title="実際に動く例" collapsible=true

[クリックしてCodeSandboxを開きます。](https://codesandbox.io/p/devbox/6-control-flow-0-7-3m4c9j?file=%2Fsrc%2Fmain.rs%3A1%2C1-91%2C2&workspaceId=478437f3-1f86-4b1e-b665-5c27a31451fb)

<noscript>
  例を表示するにはJavaScriptを有効にしてください。
</noscript>

<template>
  <iframe src="https://codesandbox.io/p/devbox/6-control-flow-0-7-3m4c9j?file=%2Fsrc%2Fmain.rs%3A1%2C1-91%2C2&workspaceId=478437f3-1f86-4b1e-b665-5c27a31451fb" width="100%" height="1000px" style="max-height: 100vh"></iframe>
</template>

```

<details>
<summary>CodeSandboxのソースコード</summary>

```rust
use leptos::prelude::*;

#[component]
fn App() -> impl IntoView {
    let (value, set_value) = signal(0);
    let is_odd = move || value.get() & 1 == 1;
    let odd_text = move || if is_odd() {
        Some("How odd!")
    } else {
        None
    };

    view! {
        <h1>"Control Flow"</h1>

        // 値を更新して表示する単純なUI
        <button on:click=move |_| *set_value.write() += 1>
            "+1"
        </button>
        <p>"Value is: " {value}</p>

        <hr/>

        <h2><code>"Option<T>"</code></h2>
        // `IntoView`を実装する任意の`T`について、
        // `Option<T>`も`IntoView`を実装する

        <p>{odd_text}</p>
        // そのため`Option`のメソッドを使用できる
        <p>{move || odd_text().map(|text| text.len())}</p>

        <h2>"Conditional Logic"</h2>
        // 動的なif-then-elseの条件ロジックは
        // いくつかの方法で記述できる
        //
        // a. 関数内の`if`式
        //    値が変化するたびに単純に再レンダリングされるため、
        //    軽量なUIに適している
        <p>
            {move || if is_odd() {
                "Odd"
            } else {
                "Even"
            }}
        </p>

        // b. クラスを切り替える
        //    状態を切り替える間も要素を破棄しないため、
        //    頻繁に切り替わる要素に適している
        //    （`hidden`クラスは`index.html`にある）
        <p class:hidden=is_odd>"Appears if even."</p>

        // c. <Show/>コンポーネント
        //    fallbackと子要素を遅延して一度だけレンダリングし、
        //    必要に応じて切り替える。多くの場合、
        //    {move || if ...}ブロックより効率的である
        <Show when=is_odd
            fallback=|| view! { <p>"Even steven"</p> }
        >
            <p>"Oddment"</p>
        </Show>

        // d. `bool::then()`は`bool`を`Option`へ変換するため、
        //    表示と非表示の切り替えに使用できる
        {move || is_odd().then(|| view! { <p>"Oddity!"</p> })}

        <h2>"Converting between Types"</h2>
        // e. 注意：ifの分岐が異なる型を返す場合は、
        //    `.into_any()`または`Either`列挙型
        //    （`Either`、`EitherOf3`、`EitherOf4`など）で変換できる
        {move || match is_odd() {
            true if value.get() == 1 => {
                // <pre>はHtmlElement<Pre>を返す
                view! { <pre>"One"</pre> }.into_any()
            },
            false if value.get() == 2 => {
                // <p>はHtmlElement<P>を返すため、
                // より汎用的な型へ変換する
                view! { <p>"Two"</p> }.into_any()
            }
            _ => view! { <textarea>{value.get()}</textarea> }.into_any()
        }}
    }
}

fn main() {
    leptos::mount::mount_to_body(App)
}
```

</details>
</preview>
