# `<ActionForm/>`

[`<ActionForm/>`](https://docs.rs/leptos/latest/leptos/form/fn.ActionForm.html) はserver actionを受け取り、フォーム送信時に自動的にdispatchする特殊な `<Form/>` です。これにより、JS／WASMがなくても `<form>` からserver functionを直接呼び出せます。

手順は簡単です。

1. [`#[server]` マクロ](https://docs.rs/leptos/latest/leptos/attr.server.html)を使ってserver functionを定義します（[Server Function](../server/25_server_functions.md)を参照）。
2. [`ServerAction::new()`](https://docs.rs/leptos/latest/leptos/server/struct.ServerAction.html) を使い、定義したserver functionの型を指定してActionを作成します。
3. `<ActionForm/>` を作成し、`action` propにserver actionを渡します。
4. server functionの名前付き引数を、同じ名前のフォームフィールドとして渡します。

> **注意：** HTMLフォームとしてのグレースフルデグラデーションと正しい動作を保証するため、`<ActionForm/>` が使えるのはserver functionのデフォルトであるURLエンコードされた `POST` エンコーディングだけです。

```rust
#[server]
pub async fn add_todo(title: String) -> Result<(), ServerFnError> {
    todo!()
}

#[component]
fn AddTodo() -> impl IntoView {
    let add_todo = ServerAction::<AddTodo>::new();
    // サーバーから最後に*返された*値を保持する
    let value = add_todo.value();
    // サーバーがエラーを返したか確認する
    let has_error = move || value.with(|val| matches!(val, Some(Err(_))));

    view! {
        <ActionForm action=add_todo>
            <label>
                "Todoを追加"
                // `title` は `add_todo` の `title` 引数に対応する
                <input type="text" name="title"/>
            </label>
            <input type="submit" value="追加"/>
        </ActionForm>
    }
}
```

本当にこれだけです。JS／WASMがあれば、フォームはページを再読み込みせずに送信され、最後の送信内容はActionの `.input()` signalに、処理中の状態は `.pending()` に保存される、といった動作をします。（復習が必要なら [`Action`](https://docs.rs/leptos/latest/leptos/reactive/actions/struct.Action.html) のドキュメントを参照してください。）JS／WASMがなければ、フォームはページを再読み込みして送信されます。（`leptos_axum` または `leptos_actix` の）`redirect` 関数を呼べば、正しいページへリダイレクトします。デフォルトでは、現在いるページへ戻ります。HTML、HTTP、アイソモーフィックレンダリングの力により、JS／WASMがなくても `<ActionForm/>` はそのまま動作します。

## クライアント側のバリデーション

`<ActionForm/>` は単なる `<form>` なので、`submit` イベントを発生させます。HTMLのバリデーションを使うことも、`on:submit:capture` ハンドラーに独自のクライアント側バリデーションロジックを書くこともできます。送信を止めるには `ev.prevent_default()` を呼ぶだけです。

送信されたフォームからserver functionのデータ型へのparseを試みるときは、[`FromFormData`](https://docs.rs/leptos/latest/leptos/form/trait.FromFormData.html) traitが役立ちます。

```rust
let on_submit = move |ev| {
	let data = AddTodo::from_event(&ev);
	// 単純なバリデーション例：Todoが「nope!」なら拒否する
	if data.is_err() || data.unwrap().title == "nope!" {
		// ev.prevent_default()でフォームの送信を止める
		ev.prevent_default();
	}
}

// ……`submit`ハンドラーを`ActionForm`へ追加する

<ActionForm on:submit:capture=on_submit /* ... */>
```

```admonish note
`on:submit` ではなく `on:submit:capture` を使っている点に注目してください。これにより、ブラウザのイベント処理における「バブリング」フェーズではなく「キャプチャ」フェーズで発火するイベントリスナーが追加されます。つまり、イベントハンドラーは `ActionForm` 組み込みの `submit` ハンドラーより先に実行されます。詳しくは[こちらのissue](https://github.com/leptos-rs/leptos/issues/3872)をご覧ください。
```

## 複雑な入力

ネストしたシリアライズ可能フィールドを持つstructをserver functionの引数にする場合は、`serde_qs` のインデックス記法を使います。

```rust
#[derive(serde::Serialize, serde::Deserialize, Debug, Clone)]
struct Settings {
    display_name: String,
}

#[derive(serde::Serialize, serde::Deserialize, Debug, Clone)]
struct HeftyData {
    first_name: String,
    last_name: String,
    settings: Settings,
}

#[component]
fn ComplexInput() -> impl IntoView {
    let submit = ServerAction::<VeryImportantFn>::new();

    view! {
      <ActionForm action=submit>
        <input type="text" name="hefty_arg[first_name]" value="leptos"/>
        <input
          type="text"
          name="hefty_arg[last_name]"
          value="closures-everywhere"
        />
        <input
          type="text"
          name="hefty_arg[settings][display_name]"
          value="my alias"
        />
        <input type="submit"/>
      </ActionForm>
    }
}

#[server]
async fn very_important_fn(hefty_arg: HeftyData) -> Result<(), ServerFnError> {
    assert_eq!(hefty_arg.first_name.as_str(), "leptos");
    assert_eq!(hefty_arg.last_name.as_str(), "closures-everywhere");
    aseert_eq!(hefty_arg.settings.display_name.as_str(), "my alias");
    Ok(())
}
```
