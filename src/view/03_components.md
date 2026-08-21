# コンポーネントとprops

ここまでは、アプリケーション全体をひとつのコンポーネントとして構築してきました。ごく小さな例であれば問題ありませんが、実際のアプリケーションではユーザーインターフェースを複数のコンポーネントへ分割する必要があります。そうすることで、インターフェースを小さく、再利用可能で、組み合わせ可能な単位へ分解できます。

先ほどのプログレスバーを例に考えてみましょう。プログレスバーをひとつではなく2つ表示し、一方はクリックするたびに1目盛り、もう一方は2目盛り進めたいとします。

単純に`<progress>`要素を2つ作れば、これを実現_することはできます_。

```rust
let (count, set_count) = signal(0);
let double_count = move || count.get() * 2;

view! {
    <progress
        max="50"
        value=count
    />
    <progress
        max="50"
        value=double_count
    />
}
```

しかし当然ながら、この方法はうまくスケールしません。3つ目のプログレスバーを追加したければ、同じコードをもう一度追加する必要があります。何か変更したい場合には、3か所すべてを編集しなければなりません。

代わりに、`<ProgressBar/>`コンポーネントを作成しましょう。

```rust
#[component]
fn ProgressBar() -> impl IntoView {
    view! {
        <progress
            max="50"
            // さて……この値はどこから受け取ればよいだろう？
            value=progress
        />
    }
}
```

ひとつだけ問題があります。`progress`が定義されていません。この値はどこから受け取ればよいのでしょうか。すべてを手動で定義していたときは、単にローカル変数名を使っていました。今度は、コンポーネントへ引数を渡す方法が必要です。

## コンポーネントのprops

これには、コンポーネントのプロパティ、つまり「props」を使います。ほかのフロントエンドフレームワークを使ったことがあれば、おそらく馴染みのある考え方でしょう。基本的には、HTML要素に対する属性と同じ役割をコンポーネントに対して果たすものがpropsです。propsを使うと、コンポーネントへ追加情報を渡せます。

Leptosでは、コンポーネント関数へ引数を追加することでpropsを定義します。

```rust
#[component]
fn ProgressBar(
    progress: ReadSignal<i32>
) -> impl IntoView {
    view! {
        <progress
            max="50"
            // 今度は正しく動作する
            value=progress
        />
    }
}
```

これで、メインの`<App/>`コンポーネントのビュー内から作成したコンポーネントを使えます。

```rust
#[component]
fn App() -> impl IntoView {
    let (count, set_count) = signal(0);
    view! {
        <button on:click=move |_| *set_count.write() += 1>
            "Click me"
        </button>
        // 作成したコンポーネントを使う
        <ProgressBar progress=count/>
    }
}
```

ビュー内でコンポーネントを使う方法は、HTML要素を使う方法とよく似ています。コンポーネント名は必ず`PascalCase`なので、要素とコンポーネントは簡単に見分けられます。`progress`というpropsを、HTML要素の属性と同じように渡します。単純ですね。

### リアクティブなpropsと静的なprops

この例を通して、`progress`が通常の`i32`ではなく、リアクティブな`ReadSignal<i32>`を受け取っていることに気づいたでしょう。これは**非常に重要**です。

コンポーネントのprops自体に、特別な意味が付与されているわけではありません。コンポーネントは、ユーザーインターフェースを設定するために一度だけ実行される単なる関数です。変化へ反応するようインターフェースに伝える唯一の方法は、シグナル型を渡すことです。そのため、今回の`progress`のように時間とともに変化するコンポーネントのプロパティは、シグナルにする必要があります。

### `optional`なprops

現在、`max`の設定はハードコードされています。これもpropsとして受け取るようにしましょう。ただし、このpropsは省略可能にします。`#[prop(optional)]`という注釈を付けることで実現できます。

```rust
#[component]
fn ProgressBar(
    // このpropsを省略可能にする
    // <ProgressBar/>を使うときに指定してもしなくてもよい
    #[prop(optional)]
    max: u16,
    progress: ReadSignal<i32>
) -> impl IntoView {
    view! {
        <progress
            max=max
            value=progress
        />
    }
}
```

これで`<ProgressBar max=50 progress=count/>`と記述できるほか、`max`を省略してデフォルト値を使うこともできます（つまり`<ProgressBar progress=count/>`です）。`optional`なpropsのデフォルト値は、その型の`Default::default()`の値です。`u16`の場合は`0`になります。プログレスバーの最大値が`0`では、あまり役に立ちません。

そこで、代わりに具体的なデフォルト値を指定しましょう。

### `default` props

`Default::default()`以外のデフォルト値も、
`#[prop(default = ...)]`で簡単に指定できます。

```rust
#[component]
fn ProgressBar(
    #[prop(default = 100)]
    max: u16,
    progress: ReadSignal<i32>
) -> impl IntoView {
    view! {
        <progress
            max=max
            value=progress
        />
    }
}
```

### ジェネリックなprops

ここまでは順調です。しかし、最初は2つのカウンターがありました。一方は`count`、もう一方は派生シグナル`double_count`によって駆動されていました。別の`<ProgressBar/>`の`progress` propsとして`double_count`を渡し、同じ状態を再現してみましょう。

```rust,compile_fail
#[component]
fn App() -> impl IntoView {
    let (count, set_count) = signal(0);
    let double_count = move || count.get() * 2;

    view! {
        <button on:click=move |_| { set_count.update(|n| *n += 1); }>
            "Click me"
        </button>
        <ProgressBar progress=count/>
        // 2つ目のプログレスバーを追加する
        <ProgressBar progress=double_count/>
    }
}
```

しかし、これはコンパイルできません。理由は簡単です。`progress` propsは`ReadSignal<i32>`を受け取ると宣言しましたが、`double_count`は`ReadSignal<i32>`ではありません。rust-analyzerが示すとおり、その型は`|| -> i32`、つまり`i32`を返すクロージャです。

これに対処する方法はいくつかあります。ひとつは次のように考えることです。「ビューをリアクティブにするには、関数かシグナルを受け取る必要がある。シグナルをクロージャで包めば、いつでも関数に変換できる。それなら、どのような関数でも受け取れるようにすればよいのでは？」

`nightly` featureを有効にしたnightly版Rustを使用している場合、シグナルは関数です。そのため、ジェネリックなコンポーネントを使い、任意の`Fn() -> i32`を受け取れます。

```rust
#[component]
fn ProgressBar(
    #[prop(default = 100)]
    max: u16,
    progress: impl Fn() -> i32 + Send + Sync + 'static
) -> impl IntoView {
    view! {
        <progress
            max=max
            value=progress
        />
        // 重なりを避けるため改行を追加する
        <br/>
    }
}
```

> ジェネリックなpropsは、`where`句、または`ProgressBar<F: Fn() -> i32 + 'static>`のようなインラインジェネリクスでも指定できます。

ビュー内では、型を表す構文`<Component<T>/>`を使ってジェネリックを指定できます（ターボフィッシュ形式の`<Component::<T>/>`ではありません）。

```rust
#[component]
fn SizeOf<T: Sized>(#[prop(marker)] _ty: PhantomData<T>) -> impl IntoView {
    std::mem::size_of::<T>()
}

#[component]
pub fn App() -> impl IntoView {
    view! {
        <SizeOf<usize>/>
        <SizeOf<String>/>
    }
}
```

> いくつか制約があることに注意してください。たとえば、viewマクロのパーサーは`<SizeOf<Vec<T>>/>`のような入れ子のジェネリクスを処理できません。

### `marker` props

ジェネリクスは、コンポーネントのprops内のどこかで使う必要があります。propsは構造体として構築されるため、すべてのジェネリック型をその構造体内のどこかで使用しなければならないからです。省略可能な`PhantomData` props、つまり`#[prop(optional)] _ty: PhantomData<T>`を使えば簡単に実現できます。ただし、これによって`_ty`フィールドのドキュメントとsetterが生成されます。常にデフォルト値を使う型には`#[prop(marker)]`を使用できます。これにより、propsがドキュメントとbuilderから除外され、islandsではフィールドに`#[serde(skip)]`が適用されます。

### `into` props

stable版Rustでは、シグナルは`Fn()`を直接実装していません。シグナルをクロージャ（`move || progress.get()`）で包むこともできますが、少し煩雑です。

別の実装方法として、`#[prop(into)]`を使用できます。この属性はpropsとして渡した値に対して自動的に`.into()`を呼び出すため、異なる型の値をpropsへ簡単に渡せます。

この場合は、[`Signal`](https://docs.rs/leptos/latest/leptos/reactive/wrappers/read/struct.Signal.html)型について知っておくと役立ちます。`Signal`は、読み取り可能なあらゆる種類のリアクティブシグナル、または通常の値を表す列挙型です。さまざまな種類のシグナルを渡して再利用したいコンポーネントのAPIを定義するときに便利です。

```rust
#[component]
fn ProgressBar(
    #[prop(default = 100)]
    max: u16,
    #[prop(into)]
    progress: Signal<i32>
) -> impl IntoView
{
    view! {
        <progress
            max=max
            value=progress
        />
        <br/>
    }
}

#[component]
fn App() -> impl IntoView {
    let (count, set_count) = signal(0);
    let double_count = move || count.get() * 2;

    view! {
        <button on:click=move |_| *set_count.write() += 1>
            "Click me"
        </button>
        // .into()は`ReadSignal`を`Signal`へ変換する
        <ProgressBar progress=count/>
        // `Signal::derive()`を使い、派生シグナルを`Signal`型で包む
        <ProgressBar progress=Signal::derive(double_count)/>
    }
}
```

### 省略可能なジェネリックprops

コンポーネントには、省略可能なジェネリックpropsを指定できないことに注意してください。指定しようとすると何が起きるか見てみましょう。

```rust,compile_fail
#[component]
fn ProgressBar<F: Fn() -> i32 + Send + Sync + 'static>(
    #[prop(optional)] progress: Option<F>,
) -> impl IntoView {
    progress.map(|progress| {
        view! {
            <progress
                max=100
                value=progress
            />
            <br/>
        }
    })
}

#[component]
pub fn App() -> impl IntoView {
    view! {
        <ProgressBar/>
    }
}
```

Rustは親切にも、次のエラーを表示します。

```
xx |         <ProgressBar/>
   |          ^^^^^^^^^^^ cannot infer type of the type parameter `F` declared on the function `ProgressBar`
   |
help: consider specifying the generic argument
   |
xx |         <ProgressBar::<F>/>
   |                     +++++
```

コンポーネントのジェネリクスは`<ProgressBar<F>/>`という構文で指定できます（`view`マクロ内ではターボフィッシュを使いません）。しかし、ここで正しい型を指定することはできません。一般にクロージャや関数は名前を付けられない型だからです。コンパイラーは短縮表記で表示できますが、自分でその型を指定することはできません。

ただし、`Box<dyn _>`や`&dyn _`を使って具体的な型を与えれば、この問題を回避できます。

```rust
#[component]
fn ProgressBar(
    #[prop(optional)] progress: Option<Box<dyn Fn() -> i32 + Send + Sync>>,
) -> impl IntoView {
    progress.map(|progress| {
        view! {
            <progress
                max=100
                value=progress
            />
            <br/>
        }
    })
}

#[component]
pub fn App() -> impl IntoView {
    view! {
        <ProgressBar/>
    }
}
```

これでRustコンパイラーはpropsの具体的な型を把握し、`None`の場合でもメモリ上のサイズがわかるため、問題なくコンパイルできます。

> この例で`&dyn Fn() -> i32`を使うとライフタイムの問題が発生しますが、別の状況では選択肢になることがあります。

## コンポーネントをドキュメント化する

これは、本書のなかで必須度は低いものの、特に重要な節のひとつです。コンポーネントとpropsをドキュメント化することは、厳密には必須ではありません。しかし、チームやアプリの規模によっては非常に重要になります。しかも簡単に行え、すぐに効果が得られます。

コンポーネントとpropsをドキュメント化するには、コンポーネント関数と個々のpropsへドキュメントコメントを追加するだけです。

```rust
/// 目標に対する進捗を表示する。
#[component]
fn ProgressBar(
    /// プログレスバーの最大値。
    #[prop(default = 100)]
    max: u16,
    /// 表示する進捗量。
    #[prop(into)]
    progress: Signal<i32>,
) -> impl IntoView {
    /* ... */
}
```

必要なのはこれだけです。通常のRustドキュメントコメントと同じように動作しますが、Rustの関数引数ではできない、個々のコンポーネントpropsのドキュメント化も可能です。

これにより、コンポーネント、その`Props`型、およびpropsの追加に使われる各フィールドのドキュメントが自動生成されます。コンポーネント名やpropsにカーソルを合わせ、`#[component]`マクロとrust-analyzerの組み合わせによる効果を実際に見るまでは、その強力さを実感しにくいかもしれません。

## コンポーネントへ属性を展開する

コンポーネントへ追加の属性を設定できるようにしたいことがあります。たとえば、スタイル設定などの目的で、利用者が独自の`class`属性や`id`属性を追加できるようにしたい場合です。

`class`や`id`のpropsを作り、それを適切な要素へ適用することでも実現_できます_。しかしLeptosは、追加属性をコンポーネントへ「展開（spread）」する機能もサポートしています。コンポーネントへ追加した属性は、そのビューから返されるすべてのトップレベルHTML要素へ適用されます。

```rust
// viewマクロでタグ名にspreadの{..}を使うと、属性リストを作成できる
let spread_onto_component = view! {
    <{..} aria-label="a component with attribute spreading"/>
};


view! {
    // コンポーネントへ展開された属性は、コンポーネントのビューが返す*すべて*の要素へ適用される
    // 一部だけへ適用する場合は、コンポーネントのprops経由で渡す
    <ComponentThatTakesSpread
        // 通常の識別子はpropsを表す
        some_prop="foo"
        another_prop=42

        // class:、style:、prop:、on:構文は要素の場合と同じように動作する
        class:foo=true
        style:font-weight="bold"
        prop:cool=42
        on:click=move |_| alert("clicked ComponentThatTakesSpread")

        // 通常のHTML属性を渡すにはattr:を先頭に付ける
        attr:id="foo"

        // 複数の属性を含める場合は、それぞれにattr:を付ける代わりに、
        // spreadの{..}を使ってコンポーネントのpropsと分離できる
        {..} // これ以降はすべてHTML属性として扱われる
        title="ooh, a title!"

        // 上で定義した属性リスト全体を追加できる
        {..spread_onto_component}
    />
}
```

``````admonish note
複数のコンポーネントで使えるよう属性を関数へ切り出したい場合は、`impl Attribute`を返す関数を実装します。

先ほどの例は次のようになります。

```rust
fn spread_onto_component() -> impl Attribute {
    view!{
        <{..} aria-label="a component with attribute spreading"/>
    }
}

view!{
    <SomeComponent {..spread_onto_component()} />
}
```
``````

属性をコンポーネントへ展開しつつ、すべてのトップレベル要素以外へ適用したい場合は、[`AttributeInterceptor`](https://docs.rs/leptos/latest/leptos/attribute_interceptor/fn.AttributeInterceptor.html)を使います。

その他の例については、[`spread`の例](https://github.com/leptos-rs/leptos/blob/main/examples/spread/src/lib.rs)を参照してください。

```admonish sandbox title="Live example" collapsible=true

[クリックしてCodeSandboxを開きます。](https://codesandbox.io/p/devbox/3-components-0-7-rkjn3j?file=%2Fsrc%2Fmain.rs%3A39%2C10)

<noscript>
  例を表示するにはJavaScriptを有効にしてください。
</noscript>

<template>
  <iframe src="https://codesandbox.io/p/devbox/3-components-0-7-rkjn3j?file=%2Fsrc%2Fmain.rs%3A39%2C10" width="100%" height="1000px" style="max-height: 100vh"></iframe>
</template>

```

<details>
<summary>CodeSandboxのソースコード</summary>

```rust
use leptos::prelude::*;

// 複数のコンポーネントを組み合わせてユーザーインターフェースを構築する
// ここでは再利用可能な<ProgressBar/>を定義する
// ドキュメントコメントでコンポーネントとpropsを
// ドキュメント化する方法も示す

/// 目標に対する進捗を表示する。
#[component]
fn ProgressBar(
    // 省略可能なpropsとして指定する
    // デフォルトは型のデフォルト値、つまり0になる
    #[prop(default = 100)]
    /// プログレスバーの最大値。
    max: u16,
    // propsへ渡された値に対して`.into()`を実行する
    #[prop(into)]
    // `Signal<T>` is a wrapper for several reactive types.
    // あらゆる種類のリアクティブな値を受け取りたい
    // このようなコンポーネントAPIで役立つ
    /// 表示する進捗量。
    progress: Signal<i32>,
) -> impl IntoView {
    view! {
        <progress
            max={max}
            value=progress
        />
        <br/>
    }
}

#[component]
fn App() -> impl IntoView {
    let (count, set_count) = signal(0);

    let double_count = move || count.get() * 2;

    view! {
        <button
            on:click=move |_| {
                *set_count.write() += 1;
            }
        >
            "Click me"
        </button>
        <br/>
        // CodeSandboxまたはrust-analyzer対応エディターで開いている場合は、
        // `ProgressBar`、`max`、`progress`へカーソルを合わせ、
        // 上で定義したドキュメントを確認する
        <ProgressBar max=50 progress=count/>
        // こちらではデフォルトの最大値を使う
        // デフォルトは100なので、半分の速さで進む
        <ProgressBar progress=count/>
        // Signal::deriveは派生シグナルからSignalラッパーを作成する
        // double_countを使うため2倍の速さで進む
        <ProgressBar max=50 progress=Signal::derive(double_count)/>
    }
}

fn main() {
    leptos::mount::mount_to_body(App)
}
```

</details>
</preview>
