# 幕間：スタイリング

ウェブサイトやアプリケーションを作成すると、すぐにスタイリングの問題へ行き着きます。
小さなアプリケーションなら、ひとつのCSSファイルで十分でしょう。しかし規模が大きくなると、
素のCSSは次第に管理しにくくなると多くの開発者が感じます。

Angular、Vue、Svelteなどには、CSSを特定のコンポーネントへ限定する仕組みが組み込まれて
います。小さなコンポーネント向けのスタイルが全体へ影響するのを防ぎ、アプリケーション全体の
スタイルを管理しやすくします。ReactやSolidなどはCSSスコープを組み込まず、エコシステムの
ライブラリへ任せています。Leptosは後者です。フレームワーク自体はCSSについて何も規定
しませんが、スタイリングライブラリを構築できる道具とプリミティブを提供します。

素のCSSから始め、Leptosアプリケーションをスタイリングする方法をいくつか紹介します。

## 素のCSS

### Trunkによるクライアントサイドレンダリング

`trunk` を使うとCSSファイルや画像をサイトと一緒にバンドルできます。`index.html` の
`<head>` で定義し、Trunkアセットとして追加します。たとえば `style.css` を追加するには
`<link data-trunk rel="css" href="./style.css"/>` タグを記述します。

詳しくはTrunkの[アセットに関するドキュメント](https://trunk-rs.github.io/trunk/guide/assets/index.html)をご覧ください。

### `cargo-leptos`によるサーバーサイドレンダリング

`cargo-leptos` のテンプレートはデフォルトでSASSを使ってCSSファイルをバンドルし、
`/pkg/{project_name}.css` へ出力します。CSSファイルを追加するには `style.scss` へ
インポートするか、`public` ディレクトリへ置きます。たとえば `public/foo.css` は
`/foo.css` として配信されます。

コンポーネント内でスタイルシートを読み込むには、[`Stylesheet`](https://docs.rs/leptos_meta/latest/leptos_meta/fn.Stylesheet.html) コンポーネントを使えます。

## TailwindCSS：ユーティリティファーストCSS

[TailwindCSS](https://tailwindcss.com/) は人気のユーティリティファーストCSSライブラリです。
インラインのユーティリティクラスでアプリケーションをスタイリングし、専用CLIがファイル内の
Tailwindクラス名を走査して必要なCSSをバンドルします。

これにより、次のようなコンポーネントを記述できます。

```rust
#[component]
fn Home() -> impl IntoView {
    let (count, set_count) = signal(0);

    view! {
        <main class="my-0 mx-auto max-w-3xl text-center">
            <h2 class="p-6 text-4xl">"Welcome to Leptos with Tailwind"</h2>
            <p class="px-10 pb-10 text-left">"Tailwind will scan your Rust files for Tailwind class names and compile them into a CSS file."</p>
            <button
                class="bg-sky-600 hover:bg-sky-700 px-5 py-3 text-white rounded-lg"
                on:click=move |_| *set_count.write() += 1
            >
                {move || if count.get() == 0 {
                    "Click me!".to_string()
                } else {
                    count.get().to_string()
                }}
            </button>
        </main>
    }
}
```

Tailwindとの統合は最初の設定が少し複雑ですが、[クライアントサイドレンダリングの`trunk`
アプリケーション](https://github.com/leptos-rs/leptos/tree/main/examples/tailwind_csr)と、
[サーバーレンダリングの`cargo-leptos`アプリケーション](https://github.com/leptos-rs/leptos/tree/main/examples/tailwind_actix)
の例があります。`cargo-leptos` にはTailwind CLIの代わりに使える
[組み込みのTailwindサポート](https://github.com/leptos-rs/cargo-leptos#site-parameters)もあります。

## Stylers：コンパイル時のCSS抽出

[Stylers](https://github.com/abishekatp/stylers) は、コンポーネント本体でスコープ付きCSSを
宣言できる、コンパイル時CSSスコープライブラリです。コンパイル時にCSSファイルへ抽出し、
アプリケーションへインポートします。そのためWASMバイナリのサイズを増やしません。

次のようなコンポーネントを記述できます。

```rust
use stylers::style;

#[component]
pub fn App() -> impl IntoView {
    let styler_class = style! { "App",
        ##two{
            color: blue;
        }
        div.one{
            color: red;
            content: raw_str(r#"\hello"#);
            font: "1.3em/1.2" Arial, Helvetica, sans-serif;
        }
        div {
            border: 1px solid black;
            margin: 25px 50px 75px 100px;
            background-color: lightblue;
        }
        h2 {
            color: purple;
        }
        @media only screen and (max-width: 1000px) {
            h3 {
                background-color: lightblue;
                color: blue
            }
        }
    };

    view! { class = styler_class,
        <div class="one">
            <h1 id="two">"Hello"</h1>
            <h2>"World"</h2>
            <h2>"and"</h2>
            <h3>"friends!"</h3>
        </div>
    }
}
```

## Stylance：CSSファイルに記述するスコープ付きCSS

StylersはRustコードへCSSをインラインで記述し、コンパイル時に抽出してスコープを設定します。
[Stylance](https://github.com/basro/stylance-rs) はコンポーネントと同じ場所のCSSファイルへ
CSSを記述し、そのファイルをコンポーネントへインポートしてCSSクラスのスコープを限定します。

編集したCSSファイルをブラウザですぐに更新できるため、`trunk` と `cargo-leptos` の
ライブリロード機能と相性がよい方法です。

```rust
import_style!(style, "app.module.scss");

#[component]
fn HomePage() -> impl IntoView {
    view! {
        <div class=style::jumbotron/>
    }
}
```

Rustを再コンパイルせず、CSSを直接編集できます。

```css
.jumbotron {
  background: blue;
}
```


## Styled：実行時のCSSスコープ



[Styled](https://github.com/eboody/styled) はLeptosと統合しやすい、実行時CSSスコープ
ライブラリです。コンポーネント関数の本体でスコープ付きCSSを宣言し、実行時に適用します。



```rust

use styled::style;



#[component]

pub fn MyComponent() -> impl IntoView {

    let styles = style!(

      div {

        background-color: red;

        color: white;

      }

    );



    styled::view! { styles,

        <div>"This text should be red with white text."</div>

    }

}

```

## コントリビューションを歓迎します

Leptosはウェブサイトやアプリケーションのスタイリング方法を規定しませんが、それを簡単にする
道具の作成は喜んで支援します。この一覧へ追加したいCSSまたはスタイリング手法を開発している
場合は、ぜひお知らせください！
