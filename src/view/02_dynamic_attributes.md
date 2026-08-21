# `view`：動的なクラス、スタイル、属性

ここまでは、`view`マクロを使ってイベントリスナーを作成する方法と、シグナルなどの関数をビューへ渡して動的なテキストを作成する方法を見てきました。

もちろん、ユーザーインターフェース内で更新したいものはほかにもあります。この節では、クラス、スタイル、属性を動的に更新する方法を説明し、**派生シグナル（derived signal）**という概念を紹介します。

まずは、すでに見慣れているはずの単純なコンポーネントから始めます。ボタンをクリックするとカウンターが増加します。

```rust
#[component]
fn App() -> impl IntoView {
    let (count, set_count) = signal(0);

    view! {
        <button
            on:click=move |_| {
                *set_count.write() += 1;
            }
        >
            "Click me: "
            {count}
        </button>
    }
}
```

ここまでの内容は、すべて前章で説明しました。

## 動的なクラス

ここで、この要素に設定するCSSクラスの一覧を動的に更新したいとします。たとえば、カウントが奇数のときに`red`クラスを追加してみましょう。これは`class:`構文で実現できます。

```rust
class:red=move || count.get() % 2 == 1
```

`class:`属性は次のものを受け取ります。

1. コロンに続くクラス名（`red`）
2. `bool`値、または`bool`を返す関数

値が`true`ならクラスが追加され、`false`ならクラスが削除されます。また、値がシグナルへアクセスする関数であれば、シグナルが変化したときにクラスもリアクティブに更新されます。

これでボタンをクリックするたびに、数値が偶数と奇数の間で切り替わるのに合わせて、テキストの色も赤と黒の間で切り替わるはずです。

```rust
<button
    on:click=move |_| {
        *set_count.write() += 1;
    }
    // class:構文は、ひとつのクラスをリアクティブに更新する
    // ここでは`count`が奇数のときに`red`クラスを設定する
    class:red=move || count.get() % 2 == 1
>
    "Click me"
</button>
```

> 実際にコードを入力しながら読み進めている場合は、`index.html`へ次のような内容を追加してください。
>
> ```html
> <style>
>   .red {
>     color: red;
>   }
> </style>
> ```

CSSクラス名によっては、`view`マクロで直接解析できないことがあります。特に、ハイフンと数字などの文字が混在している場合です。その場合はタプル構文を使えます。`class=("name", value)`も、ひとつのクラスを直接更新します。

```rust
class=("button-20", move || count.get() % 2 == 1)
```

タプルの第1要素に配列を指定すれば、ひとつの条件に対して複数のクラスを設定することもできます。

```rust
class=(["button-20", "rounded"], move || count.get() % 2 == 1)
```

## 動的なスタイル

同様の`style:`構文を使うと、個々のCSSプロパティを直接更新できます。

```rust
let (count, set_count) = signal(0);

view! {
    <button
        on:click=move |_| {
            *set_count.write() += 10;
        }
        // `style`属性を設定する
        style="position: absolute"
        // `style:`で個々のCSSプロパティを切り替える
        style:left=move || format!("{}px", count.get() + 100)
        style:background-color=move || format!("rgb({}, {}, 100)", count.get(), 100)
        style:max-width="400px"
        // スタイルシートで使うCSS変数を設定する
        style=("--columns", move || count.get().to_string())
    >
        "Click to Move"
    </button>
}
```

## 動的な属性

通常の属性についても同じことが言えます。属性へ通常の文字列やプリミティブ値を渡すと、静的な値になります。シグナルを含む関数を属性へ渡すと、その値はリアクティブに更新されます。ビューへもうひとつ要素を追加してみましょう。

```rust
<progress
    max="50"
    // シグナルは関数なので、`value=count`と`value=move || count.get()`は
    // どちらを使っても同じである
    value=count
/>
```

これでカウントを設定するたびに、`<button>`の`class`が切り替わるだけでなく、`<progress>`バーの`value`も増加し、プログレスバーが前へ進みます。

## 派生シグナル

もう一段だけ深く掘り下げてみましょう。

`view`へ関数を渡すだけでリアクティブなインターフェースを作成できることは、すでにご存じでしょう。つまり、プログレスバーの動作も簡単に変更できます。たとえば、2倍の速さで進めたいとします。

```rust
<progress
    max="50"
    value=move || count.get() * 2
/>
```

しかし、この計算を複数の場所で再利用したい場合を考えてみてください。その場合は、シグナルへアクセスするクロージャである**派生シグナル**を使用できます。

```rust
let double_count = move || count.get() * 2;

/* ビューの残りをここに挿入する */
<progress
    max="50"
    // ここで一度使用する
    value=double_count
/>
<p>
    "Double Count: "
    // ここでもう一度使用する
    {double_count}
</p>
```

派生シグナルを使うと、アプリケーション内の複数箇所で使用できるリアクティブな計算値を、最小限のオーバーヘッドで作成できます。

注意：このように派生シグナルを使うと、シグナルが変化するたび（`count()`が変化したとき）に、`double_count`へアクセスする場所ごとに計算が1回実行されます。つまり、この例では2回です。これは非常に軽い計算なので問題ありません。負荷の高い計算でこの問題を解決するために設計されたメモ（memo）については、後の章で説明します。

> #### 発展的なトピック：生のHTMLを挿入する
>
> `view`マクロは、追加の属性として`inner_html`をサポートしています。この属性を使うと、任意の要素のHTMLコンテンツを直接設定できますが、その要素に渡したほかの子要素はすべて消去されます。渡したHTMLはエスケープ_されない_ことに注意してください。クロスサイトスクリプティング（XSS）攻撃を防ぐため、信頼できる入力だけが含まれていること、またはHTMLエンティティが適切にエスケープされていることを必ず確認してください。
>
> ```rust
> let html = "<p>このHTMLが挿入されます。</p>";
> view! {
>   <div inner_html=html/>
> }
> ```
>
> [`view`マクロの完全なドキュメントはこちらです](https://docs.rs/leptos/latest/leptos/macro.view.html)。

```admonish sandbox title="Live example" collapsible=true

[クリックしてCodeSandboxを開きます。](https://codesandbox.io/p/devbox/2-dynamic-attributes-0-7-wddqfp?file=%2Fsrc%2Fmain.rs%3A1%2C1-58%2C1)

<noscript>
  例を表示するにはJavaScriptを有効にしてください。
</noscript>

<template>
  <iframe src="https://codesandbox.io/p/devbox/2-dynamic-attributes-0-7-wddqfp?file=%2Fsrc%2Fmain.rs%3A1%2C1-58%2C1" width="100%" height="1000px" style="max-height: 100vh"></iframe>
</template>

```

<details>
<summary>CodeSandboxのソースコード</summary>

```rust
use leptos::prelude::*;

#[component]
fn App() -> impl IntoView {
    let (count, set_count) = signal(0);

    // 「派生シグナル」は、ほかのシグナルへアクセスする関数である
    // これを使うと、ひとつ以上の別のシグナルの値に依存する
    // リアクティブな値を作成できる
    let double_count = move || count.get() * 2;

    view! {
        <button
            on:click=move |_| {
                *set_count.write() += 1;
            }
            // class:構文は、ひとつのクラスをリアクティブに更新する
            // ここでは`count`が奇数のときに`red`クラスを設定する
            class:red=move || count.get() % 2 == 1
            class=("button-20", move || count.get() % 2 == 1)
        >
            "Click me"
        </button>
        // 注意：<br>のような自己終了タグには明示的な/が必要になる
        <br/>

        // `count`が変化するたびに、このプログレスバーを更新する
        <progress
            // 静的な属性はHTMLと同じように動作する
            max="50"

            // 属性へ関数を渡すと、その属性がリアクティブに設定される
            // シグナルは関数なので、`value=count`と`value=move || count.get()`は
            // どちらを使っても同じである
            value=count
        >
        </progress>
        <br/>

        // このプログレスバーは`double_count`を使うため、
        // 2倍の速さで進む
        <progress
            max="50"
            // 派生シグナルは関数なので、
            // DOMをリアクティブに更新できる
            value=double_count
        >
        </progress>
        <p>"Count: " {count}</p>
        <p>"Double Count: " {double_count}</p>
    }
}

fn main() {
    leptos::mount::mount_to_body(App)
}
```

</details>
</preview>
