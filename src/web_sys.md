# JavaScriptとの統合：`wasm-bindgen`、`web_sys`、`HtmlElement`

Leptosは、フレームワークの世界から出ずに宣言的なウェブアプリケーションを構築できる
さまざまな道具を提供します。リアクティブシステム、`component` と `view` マクロ、Routerを
使えば、ブラウザのWeb APIと直接やり取りせずにUIを構築できます。すべてをRustで直接
記述できるのは、Rustが好きならすばらしいことです（本書をここまで読んだなら、きっと
Rustが好きでしょう）。

[`leptos-use`](https://leptos-use.rs/) が提供する優れたユーティリティ群など、
エコシステムのクレートは多くのWeb APIへLeptos固有のリアクティブラッパーを提供します。

それでもJavaScriptライブラリやWeb APIへ直接アクセスする必要がある場合は多くあります。

## `wasm-bindgen`でJSライブラリを使う

RustコードはWebAssembly（WASM）モジュールへコンパイルし、ブラウザで実行できます。しかし
WASMはブラウザAPIへ直接アクセスできません。そのためRust/WASMエコシステムでは、Rustコードと
それをホストするJavaScriptブラウザ環境とのバインディングを生成します。

[`wasm-bindgen`](https://rustwasm.github.io/docs/wasm-bindgen/) はその中心です。JSの呼び出し方を
示す注釈をRustコードへ付けるインターフェイスと、必要なJSグルーコードを生成するCLIを
提供します。`trunk` と `cargo-leptos` も内部で `wasm-bindgen` を利用しています。

Rustから呼び出したいJavaScriptライブラリがある場合は、`wasm-bindgen` の
[JSから関数をインポートする方法](https://rustwasm.github.io/docs/wasm-bindgen/examples/import-js.html)
を参照してください。個々の関数、クラス、値は比較的簡単にインポートできます。

JSライブラリの直接統合が常に簡単とは限りません。Reactなど特定のJSフレームワークへ依存する
ライブラリは特に困難です。リッチテキストエディターなどDOM状態を操作するライブラリも慎重に
使う必要があります。LeptosとJSライブラリの双方が状態の最終的な信頼できる情報源だと
想定する可能性があるため、責任範囲を明確に分けてください。

## `web-sys`でWeb APIへアクセスする

別のJSライブラリを導入せずブラウザAPIへアクセスするだけなら、[`web_sys`](https://docs.rs/web-sys/latest/web_sys/)
クレートを使えます。ブラウザが提供するWeb APIに対し、ブラウザの型・関数とRustの構造体・
メソッドを1対1で対応させたバインディングを提供します。

Web APIへアクセスする「Leptosで_X_を行うには？」という疑問には、素のJavaScriptによる
解決方法を調べ、[`web-sys`のドキュメント](https://docs.rs/web-sys/latest/web_sys/)を使って
Rustへ置き換えるのがよい方法です。

> この節を読んだ後は、[`wasm-bindgen`ガイドの`web-sys`に関する章](https://rustwasm.github.io/docs/wasm-bindgen/web-sys/index.html)も参考になります。

### featureを有効にする

`web_sys` はコンパイル時間を抑えるため、多くの機能がfeatureで制限されています。APIを
利用するには、対応するfeatureを有効にする必要があります。

各項目に必要なfeatureはドキュメントへ記載されています。たとえば
[`Element::get_bounding_rect_client`](https://docs.rs/web-sys/latest/web_sys/struct.Element.html#method.get_bounding_client_rect)
には `DomRect` と `Element` featureが必要です。

Leptosはすでに[多数のfeature](https://github.com/leptos-rs/leptos/blob/main/leptos_dom/Cargo.toml#L41)
を有効にしています。必要なものが含まれていれば、自分のアプリケーションで有効にする必要は
ありません。含まれていなければ `Cargo.toml` へ追加してください。

```toml
[dependencies.web-sys]
version = "0.3"
features = ["DomRect"]
```

JavaScript標準の発展に伴い、[WebGPU](https://docs.rs/web-sys/latest/web_sys/struct.Gpu.html) のような
完全には安定していないブラウザ機能を使いたい場合もあります。`web_sys` は頻繁に変化する
可能性のある標準へ追従するため、安定性は保証されません。

利用するには環境変数 `RUSTFLAGS=--cfg=web_sys_unstable_apis` を追加します。各コマンドへ
指定するか、リポジトリの `.cargo/config.toml` へ追加できます。

コマンドへ指定する場合：

```sh
RUSTFLAGS=--cfg=web_sys_unstable_apis cargo # ...
```

`.cargo/config.toml`へ指定する場合：

```toml
[env]
RUSTFLAGS = "--cfg=web_sys_unstable_apis"
```

## `view`から生の`HtmlElement`へアクセスする

フレームワークが宣言的なので、UIを構築するためにDOMノードを直接操作する必要はありません。
しかし、ビューの一部を表す背後のDOM要素へ直接アクセスしたい場合もあります。
[「制御されていない入力」](/view/05_forms.html?highlight=NodeRef#uncontrolled-inputs)では、
[`NodeRef`](https://docs.rs/leptos/latest/leptos/tachys/reactive_graph/node_ref/struct.NodeRef.html)
を使う方法を示しました。

`NodeRef::get` は、直接操作できる正しい型の `web-sys` 要素を返します。

たとえば、次のコードを考えてみましょう。

```rust
#[component]
pub fn App() -> impl IntoView {
    let node_ref = NodeRef::<Input>::new();

    Effect::new(move |_| {
        if let Some(node) = node_ref.get() {
            leptos::logging::log!("value = {}", node.value());
        }
    });

    view! {
        <input node_ref=node_ref/>
    }
}
```

このエフェクト内の `node` は `web_sys::HtmlInputElement` なので、適切なメソッドを自由に
呼び出せます。

ここで `.get()` は `Option` を返します。DOM要素が実際に作成されて値が設定されるまで
`NodeRef` は空だからです。エフェクトはコンポーネント実行の1 tick後に動くため、多くの場合、
エフェクト実行時には `<input>` がすでに作成されています。
