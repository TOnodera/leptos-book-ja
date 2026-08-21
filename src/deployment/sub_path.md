# ルート以外のパスへデプロイする

ここまでのデプロイ手順では、アプリケーションをドメインのルートパス（`/`）へデプロイすることを前提としていました。しかし、`/my-app` のようなルート以外のパスへデプロイすることもできます。

ルート以外のパスへデプロイする場合は、新しいベースパスをアプリケーションの各部分へ伝えるため、いくつかの手順が必要です。

## Routerの `base` を更新する

[`<Router/>`](https://docs.rs/leptos_router/latest/leptos_router/components/fn.Router.html) コンポーネントには、routingのベースパスを指定する `base` propがあります。たとえば `/`、`/about`、`/contact` の3ページを持つアプリケーションを `/my-app` へデプロイし、ルートを `/my-app`、`/my-app/about`、`/my-app/contact` にするなら、`base` propを `/my-app` に設定します。

```rust
<Router base="/my-app">
    <Routes fallback=|| "Not found.">
        <Route path=path!("/") view=Home/>
        <Route path=path!("/about") view=About/>
        <Route path=path!("/contact") view=Contact/>
    </Routes>
</Router>
```

リバースプロキシを使う場合、実際には `/my-app` を配信していても、サーバーは `/` を配信していると*認識*することがよくあります。一方、ブラウザ内のrouterにはURLが `/my-app` と見えます。この場合は条件付きコンパイルで `base` propを条件に応じて設定します。
```rust
let base = if cfg!(feature = "hydrate") {
    "/my-app"
} else {
    "/"
};
// ...
<Router base> // ...
```

## `<HydrationScripts root/>` を更新する

サーバーレンダリングを使う場合、[`<HydrationScripts/>`](https://docs.rs/leptos/latest/leptos/hydration/fn.HydrationScripts.html) コンポーネントがアプリをハイドレートするJS／WASMの読み込みを担当します。このコンポーネントには、ハイドレーション用scriptのベースパスを指定する独自の `root` propがあります。scriptもサブディレクトリから配信するなら、そのベースパスを `root` propに含めてください。

## Server FunctionのURLを更新する

server functionはデフォルトで `/` へリクエストを送ります。server functionのハンドラーを別のパスへ配置している場合は、[`set_server_url`](https://docs.rs/leptos/latest/leptos/server_fn/client/fn.set_server_url.html) で設定できます。

## Trunkの設定

Trunkでクライアントサイドレンダリングを行う場合、`--public-url` で公開URLを設定する方法は[Trunkのドキュメント](https://trunk-rs.github.io/trunk/guide/assets/index.html#directives)を参照してください。
