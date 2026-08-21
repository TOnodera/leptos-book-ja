# はじめる

Leptosを始めるには、大きく分けて2つの方法があります。

1. **[Trunk](https://trunk-rs.github.io/trunk/)を使ったクライアントサイドレンダリング（CSR）** — Leptosで軽快なWebサイトを作りたいだけの場合や、既存のサーバーまたはAPIと組み合わせたい場合に適した選択肢です。
   CSRモードでは、TrunkがLeptosアプリをWebAssembly（WASM）へコンパイルし、一般的なJavaScript製シングルページアプリケーション（SPA）と同じようにブラウザ上で実行します。Leptos CSRには、ビルド時間が短く反復的な開発サイクルを素早く回せること、考え方が比較的単純であること、アプリのデプロイ方法に多くの選択肢があることなどの利点があります。一方、CSRアプリにも欠点はあります。サーバーサイドレンダリング方式と比べてエンドユーザーの初回読み込みが遅く、JavaScript製SPAにつきもののSEO上の課題はLeptos CSRアプリにも当てはまります。また、内部では自動生成された短いJavaScriptコードを使ってLeptosのWASMバンドルを読み込むため、CSRアプリを正しく表示するにはクライアント端末でJavaScriptが有効になっている_必要があります_。ソフトウェアエンジニアリング全般に言えるように、ここにも検討すべきトレードオフがあります。

2. **[`cargo-leptos`](https://github.com/leptos-rs/cargo-leptos)を使ったフルスタックのサーバーサイドレンダリング（SSR）** — フロントエンドとバックエンドの両方をRustで動かし、CRUD型のWebサイトや独自のWebアプリを構築したい場合に適した選択肢です。
   LeptosのSSRでは、サーバー上でアプリをHTMLとしてレンダリングしてブラウザへ送信します。その後、WebAssemblyを使ってHTMLに動作を組み込み、アプリをインタラクティブにします。この処理を「ハイドレーション」と呼びます。サーバー側では、Leptos SSRアプリを[Actix-web](https://docs.rs/leptos_actix/latest/leptos_actix/)または[Axum](https://docs.rs/leptos_axum/latest/leptos_axum/)のどちらか好きなサーバーライブラリと密接に統合できます。そのため、それぞれのコミュニティが提供するクレートを活用してLeptosサーバーを構築できます。
   LeptosでSSRを選ぶ利点には、Webアプリの初回読み込み時間を最短にし、SEOスコアを最適化しやすいことがあります。また、Leptosの「サーバー関数」という機能を使えば、クライアントコードからサーバー上の関数を透過的に呼び出せるため、サーバーとクライアントの境界をまたぐ処理も大幅に単純化できます（この機能については後ほど詳しく説明します）。とはいえ、フルスタックSSRがよいことずくめというわけではありません。Rustコードを変更するたびにサーバーとクライアントの両方を再コンパイルする必要があるため、開発時の反復サイクルが遅くなることや、ハイドレーションに伴う複雑さが加わることが欠点です。

本書を最後まで読み終えるころには、プロジェクトの要件に応じてどのトレードオフを受け入れ、CSRとSSRのどちらを選ぶべきかを判断できるようになっているはずです。

本書の第1部では、クライアントサイドレンダリングを行うLeptosサイトから始めます。`Trunk`を使ってJavaScriptとWASMのバンドルをブラウザへ配信し、リアクティブなUIを構築します。

第2部では`cargo-leptos`を紹介し、フルスタックSSRモードでLeptosの能力を余すところなく活用していきます。

```admonish note
JavaScriptの世界から来た方で、クライアントサイドレンダリング（CSR）やサーバーサイドレンダリング（SSR）といった用語に馴染みがなければ、ほかのフレームワークになぞらえると違いを理解しやすくなります。

LeptosのCSRモードは、React（またはSolidJSのような「シグナル」ベースのフレームワーク）を使う場合に似ています。どのようなサーバー側技術スタックとも組み合わせられる、クライアント側UIの構築に重点を置きます。

LeptosのSSRモードを使うことは、Reactの世界でNext.jsのようなフルスタックフレームワーク（またはSolidの「SolidStart」フレームワーク）を使う場合に似ています。SSRを利用すると、サーバー上でレンダリングしてからクライアントへ送るWebサイトやアプリを構築できます。SSRはサイトの読み込み性能やアクセシビリティの改善に役立つだけでなく、フロントエンドとバックエンドで異なる言語へ頭を切り替える必要がないため、ひとりの開発者がクライアント側とサーバー側の*両方*を担当しやすくなります。

Leptosフレームワークは、ReactのようにCSRモードでUIだけを作ることも、Next.jsのようにフルスタックSSRモードで使用することもできます。後者では、UIとサーバーの両方をRustというひとつの言語で構築できます。

```

## Hello World! LeptosのCSR開発環境を準備する

最初に、Rustがインストールされ、最新の状態になっていることを確認してください（手順が必要な場合は[こちら](https://www.rust-lang.org/tools/install)を参照してください）。

まだインストールしていなければ、LeptosのCSRサイトを実行するための「Trunk」ツールを、コマンドラインで次のように実行してインストールします。

```bash
cargo install --locked trunk
```

続いて、基本的なRustプロジェクトを作成します。

```bash
cargo init leptos-tutorial
```

新しく作成した`leptos-tutorial`プロジェクトへ`cd`で移動し、`leptos`を依存関係に追加します。

```bash
cargo add leptos --features=csr
```

RustがコードをWebAssemblyへコンパイルしてブラウザ上で実行できるよう、`wasm32-unknown-unknown`ターゲットを追加しておきます。

```bash
rustup target add wasm32-unknown-unknown
```

`leptos-tutorial`ディレクトリのルートに、簡単な`index.html`を作成します。

```html
<!DOCTYPE html>
<html>
  <head></head>
  <body></body>
</html>
```

そして、`main.rs`に簡単な「Hello, world!」を追加します。

```rust
use leptos::prelude::*;

fn main() {
    leptos::mount::mount_to_body(|| view! { <p>"Hello, world!"</p> })
}
```

この時点で、ディレクトリ構成はおおむね次のようになります。

```
leptos_tutorial
├── src
│   └── main.rs
├── Cargo.toml
├── index.html
```

次に、`leptos-tutorial`ディレクトリのルートで`trunk serve --open`を実行します。
Trunkによってアプリが自動的にコンパイルされ、既定のブラウザで開かれるはずです。
`main.rs`を編集すると、Trunkがソースコードを再コンパイルしてページをライブリロードします。

LeptosとTrunkが支える、RustとWebAssembly（WASM）によるUI開発の世界へようこそ！

```admonish note
Windowsを使用している場合、`trunk serve --open`が動作しないことがあります。`--open`で問題が起きたときは、単に`trunk serve`を使用し、ブラウザのタブを手動で開いてください。
```

---

それでは、Leptosを使った最初の本格的なアプリケーション構築に入る前に、Leptosでの開発を少し楽にするために知っておきたいことをいくつか紹介します。
