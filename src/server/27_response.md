# レスポンスとリダイレクト

extractorを使うと、server function内からリクエストデータへ簡単にアクセスできます。Leptosには、`ResponseOptions` 型（[Actix](https://docs.rs/leptos_actix/latest/leptos_actix/struct.ResponseOptions.html)または[Axum](https://docs.rs/leptos_axum/latest/leptos_axum/struct.ResponseOptions.html)のドキュメントを参照）と `redirect` ヘルパー関数（[Actix](https://docs.rs/leptos_actix/latest/leptos_actix/fn.redirect.html)または[Axum](https://docs.rs/leptos_axum/latest/leptos_axum/fn.redirect.html)のドキュメントを参照）を使ってHTTPレスポンスを変更する方法も用意されています。

## `ResponseOptions`

`ResponseOptions` は、最初のサーバーレンダリングのレスポンス中と、それ以降のserver function呼び出し中にcontextを介して提供されます。HTTPレスポンスのステータスコードを設定したり、Cookieを設定するためのヘッダーなどをHTTPレスポンスへ追加したりできます。

```rust
#[server]
pub async fn tea_and_cookies() -> Result<(), ServerFnError> {
    use actix_web::{
        cookie::Cookie,
        http::header::HeaderValue,
        http::{header, StatusCode},
    };
    use leptos_actix::ResponseOptions;

    // contextからResponseOptionsを取得する
    let response = expect_context::<ResponseOptions>();

    // HTTPステータスコードを設定する
    response.set_status(StatusCode::IM_A_TEAPOT);

    // HTTPレスポンスにCookieを設定する
    let cookie = Cookie::build("biscuits", "yes").finish();
    if let Ok(cookie) = HeaderValue::from_str(&cookie.to_string()) {
        response.insert_header(header::SET_COOKIE, cookie);
    }
    Ok(())
}
```

## `redirect`

HTTPレスポンスに対する一般的な変更の1つが、別のページへのリダイレクトです。ActixとAxumの統合には、これを簡単に行うための `redirect` 関数があります。

```rust
#[server]
pub async fn login(
    username: String,
    password: String,
    remember: Option<String>,
) -> Result<(), ServerFnError> {
    const INVALID_CREDENTIALS: fn() -> ServerFnError = || -> ServerFnError {
        ServerFnError::ServerError("Invalid credentials".into())
    };

    // contextからDBプールと認証プロバイダーを取得する
    let pool = pool()?;
    let auth = auth()?;

    // ユーザーが存在するか確認する
    let user: User = User::get_from_username(username, &pool)
        .await
        .ok_or_else(INVALID_CREDENTIALS)?;

    // ユーザーが正しいパスワードを入力したか確認する
    match verify(password, &user.password)? {
        // パスワードが正しければ……
        true => {
            // ユーザーをログインさせる
            auth.login_user(user.id);
            auth.remember_user(remember.is_some());

            // ホームページへリダイレクトする
            leptos_axum::redirect("/");
            Ok(())
        }
        // そうでなければエラーを返す
        false => Err(INVALID_CREDENTIALS()),
    }
}
```

このserver functionはアプリケーションから利用できます。この `redirect` は、プログレッシブエンハンスメントに対応した `<ActionForm/>` コンポーネントとうまく連携します。JS／WASMがなければ、ステータスコードとヘッダーに従ってサーバーレスポンスがリダイレクトします。JS／WASMがあれば、`<ActionForm/>` がserver functionのレスポンス内のリダイレクトを検出し、クライアント側ナビゲーションを使って新しいページへ移動します。
