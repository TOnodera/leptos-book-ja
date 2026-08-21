# マクロを使わない：ビュービルダー構文

> ここまで説明してきた`view!`マクロの構文にまったく不満がなければ、この章は読み飛ばして構いません。この節で説明するビルダー構文はいつでも使えますが、必須ではありません。

さまざまな理由から、マクロを避けたい開発者は少なくありません。`rustfmt`のサポートが限定的であることを好ましく思わないかもしれません（とはいえ、優れたツールである[`leptosfmt`](https://github.com/bram209/leptosfmt)はぜひ確認してください）。マクロがコンパイル時間へ与える影響が気になる場合もあるでしょう。純粋なRust構文の見た目を好む人や、HTML風の構文とRustコードの間で頭を切り替えるのが難しい人もいます。あるいは、`view`マクロが提供する以上の柔軟性でHTML要素を作成・操作したいかもしれません。

いずれかに当てはまるなら、ビルダー構文が適しているかもしれません。

`view`マクロは、HTML風の構文を一連のRust関数とメソッド呼び出しへ展開します。`view`マクロを使いたくなければ、その展開後の構文を自分で直接使えます。実際、かなり使いやすい構文です。

まず、必要なら`#[component]`マクロさえ省略できます。コンポーネントはビューを作成するセットアップ関数にすぎないため、単純な関数として定義できます。

```rust
pub fn counter(initial_value: i32, step: u32) -> impl IntoView { }
```

要素は、HTML要素と同じ名前の関数を呼び出して作成します。

```rust
p()
```

カスタム要素やWeb Componentsは、その名前を指定して[`custom()`](https://docs.rs/leptos/latest/leptos/html/fn.custom.html)関数を呼び出すことで作成できます。
```rust
custom("my-custom-element")
```

要素へ子を追加するには、[`.child()`](https://docs.rs/leptos/latest/leptos/html/trait.ElementChild.html#tymethod.child)を使います。このメソッドは、ひとつの子、または[`IntoView`](https://docs.rs/leptos/latest/leptos/trait.IntoView.html)を実装する型のタプルや配列を受け取ります。

```rust
p().child((em().child("Big, "), strong().child("bold "), "text"))
```

属性は[`.attr()`](https://docs.rs/leptos/latest/leptos/attr/custom/trait.CustomAttribute.html#method.attr)で追加します。viewマクロで属性として渡せるものと同じ型、つまり[`Attribute`](https://docs.rs/leptos/latest/leptos/attr/trait.Attribute.html)を実装する型を受け取れます。

```rust
p().attr("id", "foo")
    .attr("data-count", move || count.get().to_string())
```

組み込みHTML属性の名前ごとに用意された属性メソッドを使って追加することもできます。

```rust
p().id("foo")
    .attr("data-count", move || count.get().to_string())
```

同様に、`class:`、`prop:`、`style:`構文は、それぞれ[`.class()`](https://docs.rs/leptos/latest/leptos/attr/global/trait.ClassAttribute.html#tymethod.class)、[`.prop()`](https://docs.rs/leptos/latest/leptos/attr/global/trait.PropAttribute.html#tymethod.prop)、[`.style()`](https://docs.rs/leptos/latest/leptos/attr/global/trait.StyleAttribute.html#tymethod.style)メソッドへ直接対応します。

イベントリスナーは[`.on()`](https://docs.rs/leptos/latest/leptos/attr/global/trait.OnAttribute.html#tymethod.on)で追加できます。[`leptos::ev`](https://docs.rs/leptos/latest/leptos/tachys/html/event/index.html)の型付きイベントを使うと、イベント名の入力ミスを防ぎ、コールバック関数で型を正しく推論できます。

```rust
button()
    .on(ev::click, move |_| set_count.set(0))
    .child("Clear")
```

この形式を好むなら、これらを組み合わせることで、機能の揃ったビューを非常にRustらしい構文で構築できます。

```rust
/// 単純なカウンタービュー。
// コンポーネントは実際には単なる関数呼び出しであり、
// 一度だけ実行されてDOMとリアクティブシステムを作成する
pub fn counter(initial_value: i32, step: i32) -> impl IntoView {
    let (count, set_count) = signal(initial_value);
    div().child((
        button()
            // leptos::evの型付きイベントには次の利点がある
            // 1）イベント名の入力ミスを防ぐ
            // 2）コールバックで正しく型推論できる
            .on(ev::click, move |_| set_count.set(0))
            .child("Clear"),
        button()
            .on(ev::click, move |_| *set_count.write() -= step)
            .child("-1"),
        span().child(("Value: ", move || count.get(), "!")),
        button()
            .on(ev::click, move |_| *set_count.write() += step)
            .child("+1"),
    ))
}
```

## ビルダー構文でコンポーネントを使う

ビルダー構文で独自のコンポーネントを作成するには、通常の関数を使うだけです（前述の例を参照してください）。組み込みの`For`や`Show`制御フローコンポーネントなど、ほかのコンポーネントを使う場合は、各コンポーネントがひとつのコンポーネントprops引数を受け取る関数であり、コンポーネントpropsには専用のビルダーがあることを利用できます。

コンポーネントpropsのビルダーを使う方法があります。
```rust
use leptos::html::p;

let (value, set_value) = signal(0);

Show(
    ShowProps::builder()
        .when(move || value.get() > 5)
        .fallback(|| p().child("I will appear if `value` is 5 or lower"))
        .children(ToChildren::to_children(|| {
            p().child("I will appear if `value` is above 5")
        }))
        .build(),
)
```
または、props構造体を直接構築できます。
```rust
use leptos::html::p;

let (value, set_value) = signal(0);

Show(ShowProps {
    when: move || value.get() > 5,
    fallback: (|| p().child("I will appear if `value` is 5 or lower")).into(),
    children: ToChildren::to_children(|| p().child("I will appear if `value` is above 5")),
})
```
コンポーネントビルダーを使うと、`#[prop(into)]`などの各種修飾子が正しく適用されます。構造体構文を使う場合は、自分で`.into()`を呼び出して手動で適用しています。

## マクロを展開する

ここでは、`view`マクロや`component`マクロ構文のすべての機能を詳しく説明したわけではありません。しかしRustには、どのようなマクロでも内部で何が起きているかを理解するためのツールがあります。具体的には、rust-analyzerの[「expand macro recursively」機能](https://rust-analyzer.github.io/book/features.html#expand-macro-recursively)を使うと任意のマクロを展開し、生成されるコードを表示できます。[`cargo-expand`](https://crates.io/crates/cargo-expand)は、プロジェクト内のすべてのマクロを通常のRustコードへ展開します。本書の残りの部分では引き続き`view`マクロ構文を使いますが、ビルダー構文へどう置き換えればよいかわからない場合は、これらのツールで生成コードを調べられます。
