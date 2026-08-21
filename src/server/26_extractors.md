# Extractor

前章で見たserver functionにより、サーバー上でコードを実行し、ブラウザでレンダリングするユーザーインターフェイスと統合する方法が分かりました。しかし、サーバーの能力を実際に最大限活用する方法については、あまり説明していませんでした。

## サーバーフレームワーク

Leptosを「フルスタック」フレームワークと呼んでいますが、「フルスタック」はいつでも不正確な呼び名です（結局のところ、ブラウザから電力会社までのすべてを意味するわけではありません）。ここでの「フルスタック」とは、Leptosアプリがブラウザとサーバーの両方で動き、両者を統合して、それぞれに固有の機能を組み合わせられるという意味です。本書でここまで見てきたように、同じRustモジュールに記述したコードで、ブラウザ上のボタンクリックからサーバー上のデータベース読み取りを実行できます。しかしLeptos自体がサーバー（あるいはデータベース、OS、ファームウェア、電気ケーブル……）を提供するわけではありません。

代わりにLeptosは、特に人気のある2つのRust Webサーバーフレームワーク、Actix Web（[`leptos_actix`](https://docs.rs/leptos_actix/latest/leptos_actix/)）とAxum（[`leptos_axum`](https://docs.rs/leptos_axum/latest/leptos_axum/)）との統合を提供します。各サーバーのrouterとの統合により、`.leptos_routes()` でLeptosアプリを既存のサーバーへ組み込み、server functionの呼び出しを簡単に処理できます。

> [Actix](https://github.com/leptos-rs/start-actix)と[Axum](https://github.com/leptos-rs/start-axum)のテンプレートをまだ見ていないなら、今が確認するよい機会です。

## Extractorを使う

ActixとAxumのハンドラーは、どちらも**extractor**という同じ強力な考え方に基づいています。ExtractorはHTTPリクエストから型付きデータを「抽出」し、サーバー固有のデータへ簡単にアクセスできるようにします。

Leptosは、各フレームワークのハンドラーによく似た便利な構文で、server function内からextractorを直接使うための `extract` ヘルパー関数を提供します。

### ActixのExtractor

[`leptos_actix` の `extract` 関数](https://docs.rs/leptos_actix/latest/leptos_actix/fn.extract.html)は、ハンドラー関数を引数に取ります。ハンドラーはActixのハンドラーと同様の規則に従います。リクエストから抽出される引数を受け取り、何らかの値を返す非同期関数です。ハンドラー関数は抽出されたデータを引数として受け取り、`async move` ブロックの本体でさらに `async` 処理を行えます。返した値は、そのままserver functionへ返されます。

```rust
use serde::Deserialize;

#[derive(Deserialize, Debug)]
struct MyQuery {
    foo: String,
}

#[server]
pub async fn actix_extract() -> Result<String, ServerFnError> {
    use actix_web::dev::ConnectionInfo;
    use actix_web::web::Query;
    use leptos_actix::extract;

    let (Query(search), connection): (Query<MyQuery>, ConnectionInfo) = extract().await?;
    Ok(format!("search = {search:?}\nconnection = {connection:?}",))
}
```

### AxumのExtractor

[`leptos_axum::extract`](https://docs.rs/leptos_axum/latest/leptos_axum/fn.extract.html) 関数の構文も非常によく似ています。

```rust
use serde::Deserialize;

#[derive(Deserialize, Debug)]
struct MyQuery {
    foo: String,
}

#[server]
pub async fn axum_extract() -> Result<String, ServerFnError> {
    use axum::{extract::Query, http::Method};
    use leptos_axum::extract;

    let (method, query): (Method, Query<MyQuery>) = extract().await?;

    Ok(format!("{method:?} and {query:?}"))
}
```

これらはサーバーの基本的なデータへアクセスする比較的単純な例です。しかし、まったく同じ `extract()` パターンを使い、ヘッダー、Cookie、データベース接続プールなどへアクセスできます。

Axumの `extract` 関数が対応するのは、stateが `()` のextractorだけです。`State` を使うextractorが必要なら、[`extract_with_state`](https://docs.rs/leptos_axum/latest/leptos_axum/fn.extract_with_state.html) を使います。この関数にはstateを渡す必要があります。そのためにはAxumの `FromRef` パターンで既存の `LeptosOptions` stateを拡張し、独自ハンドラーを使ってレンダリング中とserver function内でstateをcontextとして提供できます。

```rust
use axum::extract::FromRef;

/// AxumのSubStatesパターンを使い、stateに複数の項目を持たせるため
/// FromRefをderiveする。
#[derive(FromRef, Debug, Clone)]
pub struct AppState{
    pub leptos_options: LeptosOptions,
    pub pool: SqlitePool
}
```

[独自ハンドラーでcontextを提供する例はこちらです](https://github.com/leptos-rs/leptos/blob/19ea6fae6aec2a493d79cc86612622d219e6eebb/examples/session_auth_axum/src/main.rs#L24-L44)。

#### AxumのState

Axumにおける一般的な依存性注入のパターンは `State` を提供し、それをルートハンドラーで抽出することです。Leptosにはcontextを介した独自の依存性注入方法があります。共有するサーバーデータ（たとえばデータベース接続プール）の提供には、`State` の代わりにcontextを使える場合がよくあります。

```rust
let connection_pool = /* 何らかの共有state */;

let app = Router::new()
    .leptos_routes_with_context(
        &leptos_options,
        routes,
        move || provide_context(connection_pool.clone()),
        {
            let leptos_options = leptos_options.clone();
            move || shell(leptos_options.clone())
        },
    )
    // その他の処理
```

このcontextには、server function内で単に `use_context::<T>()` を使ってアクセスできます。

server function内で `State` を使う_必要がある_場合、たとえば `State` を必要とする既存のAxum extractorがある場合にも、Axumの [`FromRef`](https://docs.rs/axum/latest/axum/extract/derive.FromRef.html) パターンと [`extract_with_state`](https://docs.rs/leptos_axum/latest/leptos_axum/fn.extract_with_state.html) を使えます。基本的には、contextとAxum router stateの両方を介してstateを提供する必要があります。

```rust
#[derive(FromRef, Debug, Clone)]
pub struct MyData {
    pub value: usize,
    pub leptos_options: LeptosOptions,
}

let app_state = MyData {
    value: 42,
    leptos_options,
};

// ルートを持つアプリケーションを構築する
let app = Router::new()
    .leptos_routes_with_context(
        &app_state,
        routes,
        {
            let app_state = app_state.clone();
            move || provide_context(app_state.clone())
        },
        App,
    )
    .fallback(file_and_error_handler)
    .with_state(app_state);

// ...
#[server]
pub async fn uses_state() -> Result<(), ServerFnError> {
    let state = expect_context::<MyData>();
    let SomeStateExtractor(data) = extract_with_state(&state).await?;
    // TODO
}
```

#### ジェネリックなState

stateにジェネリック型を使いたい場合もあります。次の例を使いましょう。

```rust
pub struct AppState<TS: ThingService> {
    pub thing_service: Arc<TS>,
} 
```

Axumでは通常、次のようにハンドラーにジェネリック引数を使います。

```rust
pub async fn do_thing<TS: ThingService>(
    State(state): State<AppState<TS>>,
) -> Result<(), ThingError> {
    state.thing_service.do_thing()
}
```

残念ながら、現在Leptosのserver functionはジェネリック引数に対応していません。ただし、server function内で具体型を使って内部のジェネリック関数を呼ぶことで、この制限を回避できます。次のようにします。

```rust
pub async do_thing_inner<TS: ThingService>() -> Result<(), ServerFnError> {
    let state = expect_context::<AppState<TS>>(); // 動作します！
    state.thing_service.do_thing()
}

#[server]
pub async do_thing() -> Result<(), ServerFnError> {
    use crate::thing::service::Service as ConcreteThingService;
    use crate::thing::some_dep::SomeDep;

    do_thing_inner::<ConcreteThingService<SomeDep>>().await
}
```

## データ読み込みパターンに関する注意

Actixと（特に）Axumは、1往復のHTTPリクエストとレスポンスという考え方に基づいています。そのため通常、アプリケーションの「上部」（つまりレンダリング開始前）でextractorを実行し、抽出したデータを使ってレンダリング方法を決めます。`<button>` をレンダリングする前に、アプリが必要としうるすべてのデータを読み込みます。そして各ルートハンドラーは、そのルートで抽出すべきすべてのデータを把握する必要があります。

しかしLeptosはクライアントとサーバーを統合しており、すべてのデータを完全に再読み込みせず、UIの小さな部分だけをサーバーからの新しいデータで更新できることが重要です。そのためLeptosでは、データ読み込みをアプリケーションの「下」、できる限りユーザーインターフェイスの葉に近い位置へ押し下げます。`<button>` をクリックしたとき、必要なデータだけを更新できます。まさにそのためにserver functionがあります。読み込み・再読み込みするデータへ細かな粒度でアクセスできます。

`extract()` 関数を使えば、server function内でextractorを利用して両方のモデルを組み合わせられます。ルートextractorの能力をすべて利用しながら、何を抽出する必要があるかという知識を個々のコンポーネントへ分散できます。これにより、ルートのリファクタリングや再編成が容易になります。ルートが必要とするすべてのデータを事前に指定する必要はありません。
