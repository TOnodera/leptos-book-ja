# はじめに

本書は、Webフレームワーク[Leptos](https://github.com/leptos-rs/leptos)の入門書です。
ブラウザ上でレンダリングされるシンプルなアプリケーションから始め、サーバーサイドレンダリングとハイドレーションを備えたフルスタックアプリケーションへと進みながら、アプリケーション構築に必要な基本概念を説明します。

本書では、きめ細かなリアクティビティや現代的なWebフレームワークの詳細について、事前知識があることを前提としていません。一方で、Rustプログラミング言語、HTML、CSS、DOM、および基本的なWeb APIには馴染みがあるものとします。

Leptosに最もよく似ているのは、[Solid](https://www.solidjs.com)（JavaScript）や[Sycamore](https://sycamore-rs.netlify.app/)（Rust）などのフレームワークです。React（JavaScript）、Svelte（JavaScript）、Yew（Rust）、Dioxus（Rust）といったほかのフレームワークにも共通点があるため、いずれかを知っていればLeptosを理解しやすくなるでしょう。

APIの各部分についての詳しいドキュメントは、[Docs.rs](https://docs.rs/leptos/latest/leptos/)で参照できます。

> 本書のソースコードは[こちら](https://github.com/leptos-rs/book)で公開されています。誤字の修正や説明の明確化を目的としたPRは、いつでも歓迎されています。
