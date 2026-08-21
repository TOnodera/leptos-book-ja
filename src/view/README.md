# 第1部：ユーザーインターフェースを構築する

本書の第1部では、Leptosを使ってクライアント側にユーザーインターフェースを構築する方法を見ていきます。内部では、LeptosとTrunkが短いJavaScriptコードをバンドルします。このコードがWebAssemblyへコンパイルされたLeptosのUIを読み込み、CSR（クライアントサイドレンダリング）Webサイトのインタラクティブな動作を実現します。

第1部では、LeptosとRustでリアクティブなユーザーインターフェースを構築するために必要な基本ツールを紹介します。第1部を読み終えるころには、ブラウザ上でレンダリングされる軽快な同期型Webサイトを構築し、GitHub PagesやVercelなど、任意の静的サイトホスティングサービスへデプロイできるようになっているはずです。

```admonish info
本書を最大限に活用するため、掲載されている例を実際に入力しながら読み進めることをお勧めします。
[Getting Started](https://book.leptos.dev/getting_started/)と[Leptos DX](https://book.leptos.dev/getting_started/leptos_dx.html)の章では、ブラウザ上でのWASMエラー処理を含め、LeptosとTrunkを使った基本的なプロジェクトの設定方法を説明しました。
Leptosでの開発を始めるには、その基本設定だけで十分です。

より多機能なテンプレートから始めたい場合は、[leptos-rsの`start-trunk`](https://github.com/leptos-rs/start-trunk)テンプレートリポジトリを自由に利用してください。このテンプレートでは、後ほど本書で扱うルーティング、ページのheadへの`<Title>`タグと`<Meta>`タグの挿入など、実際のLeptosプロジェクトで目にする基本設定や便利な機能を確認できます。

`start-trunk`テンプレートを使うには、`Trunk`と`cargo-generate`をインストールしておく必要があります。それぞれ`cargo install trunk`と`cargo install cargo-generate`を実行するとインストールできます。

テンプレートを使ってプロジェクトを設定するには、次のコマンドを実行します。

`cargo generate --git https://github.com/leptos-rs/start-trunk`

続いて、新しく作成されたアプリのディレクトリで次のコマンドを実行し、アプリの開発を開始します。

`trunk serve --port 3000 --open`

ファイルを変更するとTrunkサーバーがアプリを再読み込みするため、比較的スムーズに開発を進められます。

```
