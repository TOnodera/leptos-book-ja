# `cargo-leptos`の紹介

ここまではブラウザでコードを実行し、Trunkでビルドとローカル開発環境を管理してきました。
SSRを追加するには、サーバー上でもアプリケーションコードを実行する必要があります。
ネイティブコードへコンパイルしてサーバーで動くものと、WASMへコンパイルしてブラウザで動く
ものという2つのバイナリを構築し、サーバーはWASM版と初期化用JavaScriptを配信します。

不可能ではありませんが、複雑になります。開発体験を改善するため、
[`cargo-leptos`](https://github.com/leptos-rs/cargo-leptos) ビルドツールを作りました。
変更時のサーバー側とクライアント側の再コンパイルを管理し、Tailwind、SASS、テストなどの
組み込みサポートも提供します。

使い始めるには次を実行します。

```bash
cargo install --locked cargo-leptos
```

新しいプロジェクトを作成するには、次のいずれかを実行します。

```bash
# Actixテンプレート
cargo leptos new --git https://github.com/leptos-rs/start-actix
```

or

```bash
# Axumテンプレート
cargo leptos new --git https://github.com/leptos-rs/start-axum
```

ブラウザで実行するWebAssemblyへRustコードをコンパイルできるよう、`wasm32-unknown-unknown`
ターゲットを追加してください。
```bash
rustup target add wasm32-unknown-unknown
```

作成したディレクトリへ `cd` し、次を実行します。

```bash
cargo leptos watch
```

コンパイル完了後、ブラウザで [`http://localhost:3000`](http://localhost:3000) を開けます。

`cargo-leptos` には多くの追加機能と組み込みツールがあります。詳しくは
[`README`](https://github.com/leptos-rs/cargo-leptos/blob/main/README.md)をご覧ください。

では `localhost:3000` を開くと何が起きるのでしょうか。続きを見ていきましょう。
