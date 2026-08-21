# 基本的なコンポーネント

先ほどの「Hello, world!」は_非常に_単純な例でした。もう少し一般的なアプリに近いものへ進みましょう。

まず`main`関数を編集し、アプリ全体を直接レンダリングする代わりに、`<App/>`コンポーネントだけをレンダリングするようにします。ほとんどのWebフレームワークでは、コンポーネントが構成と設計の基本単位であり、Leptosも例外ではありません。概念的にはHTML要素に似ており、独立して定義された振る舞いを備えたDOMの一部分を表します。HTML要素とは異なり、コンポーネント名には`PascalCase`を使います。そのため、ほとんどのLeptosアプリケーションは`<App/>`のようなコンポーネントから始まります。

```rust
use leptos::mount::mount_to_body;

fn main() {
    mount_to_body(App);
}
```

それでは、`App`コンポーネントそのものを定義しましょう。比較的単純なので、最初に全体を示してから、1行ずつ説明します。

```rust
use leptos::prelude::*;

#[component]
fn App() -> impl IntoView {
    let (count, set_count) = signal(0);

    view! {
        <button
            on:click=move |_| set_count.set(3)
        >
            "Click me: "
            {count}
        </button>
        <p>
            "Double count: "
            {move || count.get() * 2}
        </p>
    }
}
```

## Preludeをインポートする

```rust
use leptos::prelude::*;
```

Leptosは、よく使われるトレイトと関数をまとめたpreludeを提供しています。個別にインポートしたければ、もちろんそうして構いません。コンパイラーが必要なインポートを適切に提案してくれます。

## コンポーネントのシグネチャ

```rust
#[component]
```

すべてのコンポーネント定義と同様に、[`#[component]`](https://docs.rs/leptos/latest/leptos/attr.component.html)マクロから始めます。`#[component]`は、Leptosアプリケーション内でコンポーネントとして使えるよう関数に注釈を付けます。このマクロが備えるほかの機能については、数章後に説明します。

```rust
fn App() -> impl IntoView
```

すべてのコンポーネントは、次の特徴を持つ関数です。

1. 任意の型の引数を0個以上受け取ります。
2. `impl IntoView`を返します。これは、Leptosの`view`から返せるあらゆるものを含む不透明型です。

> コンポーネント関数の引数は、ひとつのprops構造体へまとめられます。この構造体は、必要に応じて`view`マクロによって構築されます。

## コンポーネント本体

コンポーネント関数の本体は、一度だけ実行されるセットアップ関数です。何度も再実行されるレンダー関数ではありません。通常は、いくつかのリアクティブ変数を作成し、それらの値の変化に応じて実行される副作用を定義し、ユーザーインターフェースを記述するために使います。

```rust
let (count, set_count) = signal(0);
```

[`signal`](https://docs.rs/leptos/latest/leptos/reactive/signal/fn.signal.html)
は、Leptosにおけるリアクティブな変化と状態管理の基本単位であるシグナルを作成します。戻り値は`(getter, setter)`のタプルです。現在の値へアクセスするには`count.get()`を使います（`nightly`版Rustでは、短縮記法の`count()`も使えます）。現在の値を設定するには`set_count.set(...)`を呼び出します（nightlyでは`set_count(...)`も使えます）。

> `.get()`は値をクローンし、`.set()`は値を上書きします。多くの場合、`.with()`や`.update()`を使う方が効率的です。この時点でそれぞれのトレードオフを詳しく知りたい場合は、[`ReadSignal`](https://docs.rs/leptos/latest/leptos/reactive/signal/struct.ReadSignal.html)と[`WriteSignal`](https://docs.rs/leptos/latest/leptos/reactive/signal/struct.WriteSignal.html)のドキュメントを参照してください。

## ビュー

Leptosでは、[`view`](https://docs.rs/leptos/latest/leptos/macro.view.html)マクロによるJSX風の形式を使ってユーザーインターフェースを定義します。

```rust
view! {
    <button
        // on:でイベントリスナーを定義する
        on:click=move |_| set_count.set(3)
    >
        // テキストノードは引用符で囲む
        "Click me: "

        // ブロックにはRustコードを記述できる
        // ここではシグナルの値をレンダリングする
        {count}
    </button>
    <p>
        "Double count: "
        {move || count.get() * 2}
    </p>
}
```

大部分は簡単に理解できるはずです。ほぼHTMLのように見えますが、`click`イベントリスナーを定義するための特別な`on:click`構文と、Rustの文字列のように見えるテキストノードがいくつか含まれています。`<p>`のような組み込み要素だけでなく、`<my-custom-element>`のようなカスタム要素やWeb Componentsを含め、すべてのHTML要素がサポートされています。

```admonish info
**引用符で囲まないテキスト**：`view`マクロは、HTMLやJSXで一般的な、引用符で囲まないテキストノードもある程度サポートしています（つまり、`<p>"Hello!"</p>`ではなく`<p>Hello!</p>`と記述できます）。Rustの手続きマクロの制約により、引用符で囲まないテキストを使うと、句読点の前後で空白の問題が起きることがあり、すべてのUnicode文字列を扱えるわけでもありません。好みに応じて引用符なしのテキストを使用できますが、問題が起きた場合は、テキストノードを通常のRust文字列として引用符で囲めば必ず解決できます。
```

さらに、中括弧で囲まれた値が2つあります。ひとつ目の`{count}`は簡単に理解できそうです（単なるシグナルの値です）。そしてもうひとつは……

```rust
{move || count.get() * 2}
```

これはいったい何なのでしょう。

初めてのLeptosアプリケーションで、それまでの人生で使った数より多くのクロージャを使った、と冗談を言う人がいます。たしかに、そう言いたくなるのもわかります。

ビューへ関数を渡すことは、「これは変化する可能性がある値だ」とフレームワークへ伝えることを意味します。

ボタンをクリックして`set_count`を呼び出すと、`count`シグナルが更新されます。値が`count`の値に依存する`move || count.get() * 2`クロージャが再実行され、フレームワークはその特定のテキストノードだけを狙って更新します。アプリケーション内のほかの部分には一切触れません。これによって、DOMをきわめて効率的に更新できます。

覚えておいてください。これは_非常に重要_です。ビュー内でリアクティブな値として扱われるのは、シグナルと関数だけです。

つまり、ビュー内の`{count}`と`{count.get()}`はまったく異なる動作をします。`{count}`はシグナルを渡し、`count`が変化するたびにビューを更新するようフレームワークへ伝えます。`{count.get()}`は`count`の値へ一度だけアクセスし、`i32`をビューへ渡します。そのため、リアクティブではなく、一度だけレンダリングされます。

同様に、`{move || count.get() * 2}`と`{count.get() * 2}`の動作も異なります。前者は関数なのでリアクティブにレンダリングされます。後者は値なので一度だけレンダリングされ、`count`が変化しても更新されません。

下のCodeSandboxで、その違いを確認できます。

最後にもうひとつ変更しましょう。クリックハンドラーで`set_count.set(3)`を実行しても、あまり役に立ちません。「この値を3に設定する」を「この値を1増やす」へ置き換えます。

```rust
move |_| {
    *set_count.write() += 1;
}
```

ここからわかるように、`set_count`は値を設定するだけですが、`set_count.write()`は可変参照を返し、その場で値を変更します。どちらを使っても、UIのリアクティブな更新が発生します。

> このチュートリアルでは、インタラクティブな例を示すためにCodeSandboxを使います。
> 変数にマウスカーソルを重ねると、Rust Analyzerによる詳細情報や、処理内容についてのドキュメントが表示されます。自由に例をforkして、実際に試してみてください。

```admonish sandbox title="Live example" collapsible=true

[クリックしてCodeSandboxを開きます。](https://codesandbox.io/p/devbox/1-basic-component-0-7-qvgdxs?file=%2Fsrc%2Fmain.rs%3A1%2C1-59%2C2&workspaceId=478437f3-1f86-4b1e-b665-5c27a31451fb)

<noscript>
  例を表示するにはJavaScriptを有効にしてください。
</noscript>

> サンドボックス内にブラウザを表示するには、`Add DevTools > Other Previews > 8080`をクリックする必要がある場合があります。

<template>
  <iframe src="https://codesandbox.io/p/devbox/1-basic-component-0-7-qvgdxs?file=%2Fsrc%2Fmain.rs%3A1%2C1-59%2C2&workspaceId=478437f3-1f86-4b1e-b665-5c27a31451fb" width="100%" height="1000px" style="max-height: 100vh"></iframe>
</template>

```

<details>
<summary>CodeSandboxのソースコード</summary>

```rust
use leptos::prelude::*;

// #[component]マクロは、関数を再利用可能なコンポーネントとして指定する
// コンポーネントはユーザーインターフェースの構成要素であり、
// 再利用可能な振る舞いの単位を定義する
#[component]
fn App() -> impl IntoView {
    // ここでリアクティブなシグナルを作成し、
    // (getter, setter)のペアを取得する
    // シグナルはフレームワークにおける変化の基本単位である
    // 詳細は後ほど説明する
    let (count, set_count) = signal(0);

    // `view`マクロを使ってユーザーインターフェースを定義する
    // 一部のRust値を受け取れるHTML風の形式を使用する
    view! {
        <button
            // on:clickは`click`イベントが発生するたびに実行される
            // イベントハンドラーはすべて`on:{eventname}`として定義する

            // シグナルはCopyかつ'staticなので、
            // `set_count`をクロージャへmoveできる

            on:click=move |_| *set_count.write() += 1
        >
            // RSX内のテキストノードは、通常のRust文字列と同様に
            // 引用符で囲む
            "Click me: "
            {count}
        </button>
        <p>
            <strong>"Reactive: "</strong>
            // Rust式を中括弧で囲むと、DOMへ値として挿入できる
            // 関数を渡した場合はリアクティブに更新される
            {move || count.get()}
        </p>
        <p>
            <strong>"Reactive shorthand: "</strong>
            // getterを包むだけの関数の短縮記法として、
            // ビュー内でシグナルを直接使用できる
            {count}
        </p>
        <p>
            <strong>"Not reactive: "</strong>
            // 注意：{count.get()}とだけ記述した場合は*リアクティブにならない*
            // countの値を一度取得するだけである
            {count.get()}
        </p>
    }
}

// この`main`関数がアプリのエントリーポイントになる
// コンポーネントを<body>へマウントするだけである
// `fn App`として定義したので、テンプレート内で
// <App/>として使用できる
fn main() {
    leptos::mount::mount_to_body(App)
}
```
</details>
