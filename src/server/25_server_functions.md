# Server Function

おもちゃのようなアプリを超えるものを作るなら、サーバー上でコードを実行する機会が常にあります。サーバーだけで動くデータベースの読み書き、クライアントへ送りたくないライブラリを使った高コストな計算、CORS上の理由でクライアントではなくサーバーから呼ぶ必要があるAPIへのアクセス、あるいはサーバーに保存された秘密のAPIキーを必要とし、ユーザーのブラウザへ決して送るべきではないAPIへのアクセスなどです。

従来はサーバーとクライアントのコードを分離し、REST APIやGraphQL APIなどを用意して、クライアントがサーバー上のデータを取得・変更できるようにしていました。この方法でも構いませんが、複数の場所に分かれたコード（取得を行うクライアント側コードと、実際に動くサーバー側関数）を記述・保守する必要があります。さらに両者の間のAPI契約という、3つ目の管理対象も生まれます。

Leptosは、**server function**という概念を導入している現代的なフレームワークの1つです。server functionには2つの重要な特徴があります。

1. Server functionはコンポーネントコードと**同じ場所に配置**されるため、技術ではなく機能ごとに作業を整理できます。たとえば、ユーザーが選んだダーク／ライトモードをセッション間で保持し、ちらつきを避けるためサーバーレンダリング中にも適用する「ダークモード」機能があるとします。これには、クライアント上で操作可能なコンポーネントと、サーバー上の処理（Cookieの設定や、場合によってはユーザー情報のデータベース保存）が必要です。従来、この機能はコード内の2か所、「フロントエンド」と「バックエンド」に分割されがちでした。server functionなら、おそらく両方を1つの `dark_mode.rs` に書くだけで済みます。
2. Server functionは**アイソモーフィック**です。つまり、サーバーからもブラウザからも呼び出せます。これは2つのプラットフォーム向けに異なるコードを生成することで実現します。サーバーではserver functionがそのまま実行されます。ブラウザではserver functionの本体が、サーバーへfetchリクエストを送るスタブに置き換えられます。引数はリクエストへシリアライズされ、戻り値はレスポンスからデシリアライズされます。しかしどちら側でも、単純に関数として呼び出せます。データベースへ書き込む `add_todo` 関数を作り、ブラウザ内のボタンのクリックハンドラーから普通に呼び出せるのです！

## Server Functionを使う

この例はなかなかよさそうです。どのようなコードになるでしょうか？ 実際、とても簡単です。

```rust
// todo.rs

#[server]
pub async fn add_todo(title: String) -> Result<(), ServerFnError> {
    let mut conn = db().await?;

    match sqlx::query("INSERT INTO todos (title, completed) VALUES ($1, false)")
        .bind(title)
        .execute(&mut conn)
        .await
    {
        Ok(_row) => Ok(()),
        Err(e) => Err(ServerFnError::ServerError(e.to_string())),
    }
}

#[component]
pub fn BusyButton() -> impl IntoView {
	view! {
        <button on:click=move |_| {
            spawn_local(async {
                add_todo("So much to do!".to_string()).await;
            });
        }>
            "Todoを追加"
        </button>
	}
}
```

すぐにいくつかの点に気づくでしょう。

- Server functionは `sqlx` のようなサーバー専用の依存関係を使い、データベースのようなサーバー専用リソースへアクセスできます。
- Server functionは `async` です。サーバー上で同期的な処理しか行わなくても、関数シグネチャは `async` でなければなりません。ブラウザからの呼び出しは、必ず非同期になるためです。
- Server functionは `Result<T, ServerFnError>` を返します。これも、サーバー上で失敗しない処理しか行わない場合にも変わりません。`ServerFnError` のvariantには、ネットワークリクエストの過程で起こりうるさまざまな問題が含まれるためです。
- Server functionはクライアントから呼び出せます。クリックハンドラーを見てください。このコードはクライアントで_しか_実行されません。それでも、通常の非同期関数と同じように `add_todo` を呼び出せます（`Future` の実行には `spawn_local` を使います）。

```rust
move |_| {
	spawn_local(async {
		add_todo("So much to do!".to_string()).await;
	});
}
```

- Server functionは `fn` で定義するトップレベル関数です。イベントリスナー、派生signal、その他Leptosのほとんどのものとは異なり、クロージャではありません！ `fn` の呼び出しなので、アプリのリアクティブな状態や、引数として渡されていないものにはアクセスできません。これも当然です。サーバーへリクエストを送るとき、明示的に送らない限り、サーバーはクライアントの状態へアクセスできません。（そうでなければ、リクエストごとにリアクティブシステム全体をシリアライズして送る必要があります。これはよい考えではありません。）
- Server functionの引数と戻り値は、どちらもシリアライズ可能でなければなりません。これも納得できるでしょう。一般の関数の引数はシリアライズする必要がありませんが、ブラウザからserver functionを呼ぶには、引数をシリアライズしてHTTPで送る必要があります。

server functionの定義方法についても、注意点がいくつかあります。

- Server functionは、任意の場所で定義できるトップレベル関数に [`#[server]` マクロ](https://docs.rs/leptos/latest/leptos/attr.server.html)を付けて作成します。

Server functionは条件付きコンパイルを利用します。サーバー側では、引数をHTTPリクエストとして受け取り、結果をHTTPレスポンスとして返すエンドポイントを作ります。クライアント側／ブラウザ向けビルドでは、server functionの本体がHTTPリクエストを行うスタブに置き換えられます。

```admonish warning
### セキュリティに関する重要な注意

Server functionは優れた技術ですが、非常に重要な点を覚えておいてください。**Server functionは魔法ではなく、公開APIを定義するための糖衣構文です。** server functionの_本体_が公開されることはなく、単にサーバーバイナリの一部です。しかしserver functionは公開アクセス可能なAPIエンドポイントであり、戻り値はJSONなどのデータにすぎません。公開情報であるか、適切なセキュリティ対策を実装していない限り、server functionから情報を返してはいけません。対策には、受信リクエストの認証、適切な暗号化、アクセスのレート制限などが含まれます。
```

## Server Functionをカスタマイズする

デフォルトでは、server functionは引数をHTTP POSTリクエストとして（`serde_qs` を使用）、戻り値をJSONとして（`serde_json` を使用）エンコードします。このデフォルトは、WASMが無効、未対応、またはまだ読み込まれていない場合でもPOSTリクエストをネイティブに送信できる `<form>` 要素との互換性を高めるためのものです。名前の衝突を避けるため、エンドポイントはハッシュ化されたURLに配置されます。

ただしserver functionは、対応するさまざまな入出力エンコーディングや、特定のエンドポイントを設定する機能など、多くの方法でカスタマイズできます。

詳細や例については、[`#[server]` マクロ](https://docs.rs/leptos/latest/leptos/attr.server.html)と [`server_fn` crate](https://docs.rs/server_fn/latest/server_fn/)のドキュメント、およびリポジトリにある充実した [`server_fns_axum` の例](https://github.com/leptos-rs/leptos/blob/main/examples/server_fns_axum/src/app.rs)をご覧ください。

## 独自のエラーを使う

Server functionは、`FromServerFnError` traitを実装する任意の種類のエラーを返せます。
これによりエラー処理がはるかに扱いやすくなり、ドメイン固有のエラー情報をクライアントへ提供できます。

```rust
use leptos::prelude::*;
use server_fn::codec::JsonEncoding;
use serde::{Deserialize, Serialize};

#[derive(Debug, Clone, Deserialize, Serialize)]
pub enum AppError {
    ServerFnError(ServerFnErrorErr),
    DbError(String),
}

impl FromServerFnError for AppError {
    type Encoder = JsonEncoding;

    fn from_server_fn_error(value: ServerFnErrorErr) -> Self {
        AppError::ServerFnError(value)
    }
}

#[server]
pub async fn create_user(name: String, email: String) -> Result<User, AppError> {
    // データベースにユーザーを作成してみる
    match insert_user_into_db(&name, &email).await {
        Ok(user) => Ok(user),
        Err(e) => Err(AppError::DbError(e.to_string())),
    }
}
```

## 注意すべき特性

Server functionには、知っておく価値のある特性がいくつかあります。

- `isize` や `usize` のようなポインターサイズの整数型を使うと、32ビットのWASMアーキテクチャと64ビットのサーバーアーキテクチャ間の呼び出しでエラーになることがあります。サーバーが32ビットに収まらない値を返すと、デシリアライズエラーになります。この問題を軽減するには `i32` や `i64` のような固定サイズ型を使ってください。
- サーバーへ送られる引数は、デフォルトで `serde_qs` によりURLエンコードされます。これにより `<form>` 要素とうまく連携しますが、特有の挙動もあります。たとえば、現在の `serde_qs` はoptional型（[こちら](https://github.com/leptos-rs/leptos/issues/3832)または[こちら](https://github.com/leptos-rs/leptos/issues/4016)を参照）や、タプルvariantを持つenum（[こちら](https://github.com/leptos-rs/leptos/issues/4464)を参照）を常に適切に扱えるわけではありません。これらのissueに記載された回避策を使うか、[別の入力エンコーディングへ切り替える](https://docs.rs/leptos/latest/leptos/attr.server.html#named-arguments)ことができます。

## Server FunctionをLeptosと統合する

ここまでの説明は、実のところフレームワークに依存しません。（実際、Leptosのserver function crateはDioxusにも統合されています！）Server functionは、HTTPリクエストやURLエンコーディングといったWeb標準を利用し、関数のようなRPC呼び出しを定義する方法にすぎません。

しかしある意味では、ここまでの話で最後に欠けていたprimitiveも提供します。server functionは普通のRust非同期関数なので、[以前](../async/index.html)説明したLeptosの非同期primitiveと完全に統合できます。そのため、server functionをアプリケーションのほかの部分へ簡単に組み込めます。

- server functionを呼び出し、サーバーからデータを読み込む**Resource**を作成する
- `<Suspense/>` または `<Transition/>` の下でResourceを読み取り、ストリーミングSSRを有効にして、データ読み込み中にはフォールバック状態を表示する
- server functionを呼び出し、サーバー上のデータを変更する**Action**を作成する

本書の最後のセクションでは、プログレッシブエンハンスメントされたHTMLフォームを使ってこれらのserver actionを実行するパターンを紹介し、より具体的に説明します。

その前に次の数章では、ActixとAxumのサーバーフレームワークが提供する強力なextractorとの最適な統合方法など、server functionで行いたい処理の詳細を見ていきます。
