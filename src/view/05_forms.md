# フォームと入力

フォームとフォーム入力は、対話的なアプリケーションに欠かせない要素です。Leptosで入力を
扱う基本的なパターンは2つあります。React、SolidJS、または同様のフレームワークに
馴染みがあれば見覚えがあるでしょう。**制御された（controlled）**入力と、
**制御されていない（uncontrolled）**入力です。

## 制御された入力

「制御された入力」では、フレームワークが入力要素の状態を制御します。`input` イベントが
発生するたびに、現在の状態を保持するローカルシグナルを更新し、そのシグナルによって入力の
`value` プロパティが更新されます。

覚えておくべき重要な点が2つあります。

1. `input` イベントは要素が（ほぼ）変更されるたびに発生しますが、`change` イベントは
   （おおむね）入力からフォーカスを外したときに発生します。通常は `on:input` を使うことに
   なるでしょうが、どちらを選ぶかは自由です。
2. `value` _属性_ が設定するのは入力の初期値だけです。つまり、入力を開始するまでしか
   入力要素を更新しません。一方、`value` _プロパティ_ は入力開始後も更新を続けます。
   このため、通常は `prop:value` を設定します（`<input type="checkbox">` の `checked`
   と `prop:checked` についても同様です）。

```rust
let (name, set_name) = signal("Controlled".to_string());

view! {
    <input type="text"
        // :targetを追加すると、発生したイベントの対象要素へ
        // 型付きでアクセスできる
        on:input:target=move |ev| {
            // .value()はHTML入力要素の現在値を返す
            set_name.set(ev.target().value());
        }

        // `prop:`構文を使うと、属性ではなくDOMプロパティを更新できる
        prop:value=name
    />
    <p>"Name is: " {name}</p>
}
```

> #### `prop:value`が必要なのはなぜ？
>
> ウェブブラウザは、グラフィカルユーザーインターフェイスを描画するプラットフォームとして、
> 現存する中で最も広く普及し、安定しています。また、30年にわたる歴史の中で驚くほど高い
> 後方互換性を維持してきました。当然ながら、その結果としていくつか風変わりな点があります。
>
> そのひとつが、HTML属性とDOM要素のプロパティが区別されていることです。つまり、HTMLから
> 解析され、`.setAttribute()` でDOM要素へ設定できる「属性」と、解析されたHTML要素を表す
> JavaScriptクラスのフィールドである「プロパティ」とが別物になっています。
>
> `<input value=...>` の場合、`value` _属性_ の設定は入力の初期値を設定するものと定義され、
> `value` _プロパティ_ の設定は現在値を設定します。`about:blank` を開き、ブラウザの
> コンソールで次のJavaScriptを1行ずつ実行すると理解しやすいでしょう。
>
> ```js
> // inputを作成してDOMへ追加する
> const el = document.createElement("input");
> document.body.appendChild(el);
>
> el.setAttribute("value", "test"); // inputを更新する
> el.setAttribute("value", "another test"); // inputをもう一度更新する
>
> // inputへ文字を入力し、何文字か削除するなどの操作を行う
>
> el.setAttribute("value", "one more time?");
> // 何も変化しないはず。ここでは「初期値」を設定しても効果がない
>
> // しかし……
> el.value = "But this works";
> ```
>
> ほかの多くのフロントエンドフレームワークは属性とプロパティを同一視するか、値を正しく設定
> するために入力要素だけを特別扱いしています。Leptosもそうするべきかもしれません。しかし
> 今のところ私は、属性とプロパティのどちらを設定するかについて最大限の制御を利用者へ
> 提供し、ブラウザの実際の挙動を覆い隠すのではなく、できる限り伝える方を好んでいます。

### `bind:`で制御された入力を簡潔にする

ウェブ標準に従い、「シグナルからの読み取り」と「シグナルへの書き込み」を明確に分けるのは
よいことですが、この方法で制御された入力を作ると、必要以上に定型コードが多いように
感じる場合があります。

Leptosには、シグナルを入力へ自動的に結び付けるための特別な `bind:` 構文もあります。
これは上の「制御された入力」パターンとまったく同じ処理、すなわちシグナルを更新する
イベントリスナーと、シグナルから値を読み取る動的プロパティの作成を行います。テキスト入力
には `bind:value`、チェックボックスには `bind:checked`、ラジオボタンのグループには
`bind:group` を使えます。

```rust
let (name, set_name) = signal("Controlled".to_string());
let email = RwSignal::new("".to_string());
let favorite_color = RwSignal::new("red".to_string());
let spam_me = RwSignal::new(true);

view! {
    <input type="text"
        bind:value=(name, set_name)
    />
    <input type="email"
        bind:value=email
    />
    <label>
        "Please send me lots of spam email."
        <input type="checkbox"
            bind:checked=spam_me
        />
    </label>
    <fieldset>
        <legend>"Favorite color"</legend>
        <label>
            "Red"
            <input
                type="radio"
                name="color"
                value="red"
                bind:group=favorite_color
            />
        </label>
        <label>
            "Green"
            <input
                type="radio"
                name="color"
                value="green"
                bind:group=favorite_color
            />
        </label>
        <label>
            "Blue"
            <input
                type="radio"
                name="color"
                value="blue"
                bind:group=favorite_color
            />
        </label>
    </fieldset>
    <p>"Your favorite color is " {favorite_color} "."</p>
    <p>"Name is: " {name}</p>
    <p>"Email is: " {email}</p>
    <Show when=move || spam_me.get()>
        <p>"You’ll receive cool bonus content!"</p>
    </Show>
}
```

## 制御されていない入力

「制御されていない入力」では、ブラウザが入力要素の状態を制御します。値を保持するシグナルを
継続的に更新する代わりに、値を取得したいときに
[`NodeRef`](https://docs.rs/leptos/latest/leptos/tachys/reactive_graph/node_ref/struct.NodeRef.html)
を使って入力へアクセスします。

この例では、`<form>` が `submit` イベントを発生させたときだけフレームワークへ通知します。
各HTML要素に対応する多数の型を提供する
[`leptos::html`](https://docs.rs/leptos/latest/leptos/html/index.html) モジュールを
使っている点に注目してください。

```rust
let (name, set_name) = signal("Uncontrolled".to_string());

let input_element: NodeRef<html::Input> = NodeRef::new();

view! {
    <form on:submit=on_submit> // on_submitは後で定義する
        <input type="text"
            value=name
            node_ref=input_element
        />
        <input type="submit" value="Submit"/>
    </form>
    <p>"Name is: " {name}</p>
}
```

ここまで読めば、このビューはほぼ説明不要でしょう。次の2点に注目してください。

1. 制御された入力の例とは異なり、`prop:value` ではなく `value` を使います。入力の初期値
   だけを設定し、その後の状態の制御はブラウザへ任せるためです（代わりに `prop:value` を
   使うこともできます）。
2. `NodeRef` に値を設定するため、`node_ref=...` を使います（古い例では `_ref` を使って
   いる場合があります。両者は同じものですが、`node_ref` の方がrust-analyzerの
   サポートに優れています）。

`NodeRef` はリアクティブなスマートポインタの一種です。これを使って背後にあるDOMノードへ
アクセスできます。要素がレンダリングされると、その値が設定されます。

```rust
let on_submit = move |ev: SubmitEvent| {
    // ページの再読み込みを止める！
    ev.prevent_default();

    // ここで入力から値を取り出す
    let value = input_element
        .get()
        // イベントハンドラーが発火するのはビューがDOMへマウントされた後だけなので、
        // `NodeRef`は`Some`になる
        .expect("<input> should be mounted")
        // `leptos::HtmlElement<html::Input>`は
        // `web_sys::HtmlInputElement`への`Deref`を実装している。
        // そのため`HtmlInputElement::value()`を呼び出して
        // 入力の現在値を取得できる
        .value();
    set_name.set(value);
};
```

`on_submit` ハンドラーは入力の値へアクセスし、それを使って `set_name.set()` を呼び出します。
`NodeRef` に保存されたDOMノードへアクセスするには、関数として呼び出す（または `.get()` を
使う）だけです。戻り値は `Option<leptos::HtmlElement<html::Input>>` ですが、要素がすでに
マウントされていることはわかっています（そうでなければ、どうやってこのイベントを
発生させたのでしょう！）。そのため、ここでは安全にunwrapできます。

`NodeRef` によって正しい型のHTML要素へアクセスできるため、続いて `.value()` を呼び出し、
入力から値を取得できます。

`leptos::HtmlElement` の使い方について詳しくは、
[`web_sys`と`HtmlElement`](../web_sys.md)をご覧ください。また、このページの最後にある
CodeSandboxの完全な例も参照してください。

## 特別なケース：`<textarea>`と`<select>`

混乱を招きやすいフォーム要素が2つあります。それぞれ異なる点に注意が必要です。

### `<textarea>`

`<input>` とは異なり、`<textarea>` 要素はHTMLの `value` 属性をサポートしていません。
代わりに、HTMLの子要素にあるプレーンテキストノードを初期値として受け取ります。

そのため、初期値をサーバーでレンダリングし、ブラウザ上でも値をリアクティブにしたい場合は、
初期テキストノードを子として渡すと同時に、`prop:value` で現在値を設定できます。

```rust
view! {
    <textarea
        prop:value=move || some_value.get()
        on:input:target=move |ev| some_value.set(ev.target().value())
    >
        {some_value}
    </textarea>
}
```

### `<select>`

同様に `<select>` 要素も、それ自体の `value` プロパティを通じて制御できます。
指定された値を持つ `<option>` が選択されます。

```rust
let (value, set_value) = signal(0i32);
view! {
  <select
    on:change:target=move |ev| {
      set_value.set(ev.target().value().parse().unwrap());
    }
    prop:value=move || value.get().to_string()
  >
    <option value="0">"0"</option>
    <option value="1">"1"</option>
    <option value="2">"2"</option>
  </select>
  // 選択肢を順番に切り替えるボタン
  <button on:click=move |_| set_value.update(|n| {
    if *n == 2 {
      *n = 0;
    } else {
      *n += 1;
    }
  })>
    "Next Option"
  </button>
}
```

```admonish sandbox title="制御されたフォームと制御されていないフォームのCodeSandbox" collapsible=true

[クリックしてCodeSandboxを開きます。](https://codesandbox.io/p/devbox/5-forms-0-7-l5hktg?file=%2Fsrc%2Fmain.rs&workspaceId=478437f3-1f86-4b1e-b665-5c27a31451fb)

<noscript>
  例を表示するにはJavaScriptを有効にしてください。
</noscript>

<template>
  <iframe src="https://codesandbox.io/p/devbox/5-forms-0-7-l5hktg?file=%2Fsrc%2Fmain.rs&workspaceId=478437f3-1f86-4b1e-b665-5c27a31451fb" width="100%" height="1000px" style="max-height: 100vh"></iframe>
</template>

```

<details>
<summary>CodeSandboxのソースコード</summary>

```rust
use leptos::{ev::SubmitEvent};
use leptos::prelude::*;

#[component]
fn App() -> impl IntoView {
    view! {
        <h2>"Controlled Component"</h2>
        <ControlledComponent/>
        <h2>"Uncontrolled Component"</h2>
        <UncontrolledComponent/>
    }
}

#[component]
fn ControlledComponent() -> impl IntoView {
    // 値を保持するシグナルを作成する
    let (name, set_name) = signal("Controlled".to_string());

    view! {
        <input type="text"
            // 入力が変化するたびにイベントを発生させる
            // イベント名の後へ:targetを追加すると、ev.target()で
            // 正しい型の要素へアクセスできる
            on:input:target=move |ev| {
                set_name.set(ev.target().value());
            }

            // `prop:`構文を使うと、属性ではなくDOMプロパティを更新できる
            //
            // 重要：`value`「属性」が設定するのは、変更を加えるまでの
            // 初期値だけである。`value`「プロパティ」は現在値を設定する。
            // これはDOM特有の挙動であり、私が考案したものではない。
            // ほかのフレームワークはこの違いを覆い隠しているが、
            // ブラウザの実際の動作へアクセスできることの方が重要だと思う
            //
            // 要するに、フォーム入力にはprop:valueを使う
            prop:value=name
        />
        <p>"Name is: " {name}</p>
    }
}

#[component]
fn UncontrolledComponent() -> impl IntoView {
    // <input>の型をインポートする
    use leptos::html::Input;

    let (name, set_name) = signal("Uncontrolled".to_string());

    // NodeRefを使って入力要素への参照を保存する
    // 要素が作成されると値が設定される
    let input_element: NodeRef<Input> = NodeRef::new();

    // フォームの`submit`イベント発生時に実行される
    // <input>の値をシグナルへ保存する
    let on_submit = move |ev: SubmitEvent| {
        // ページの再読み込みを止める！
        ev.prevent_default();

        // ここで入力から値を取り出す
        let value = input_element.get()
            // イベントハンドラーが発火するのはビューがDOMへマウントされた後だけなので、
            // `NodeRef`は`Some`になる
            .expect("<input> to exist")
            // `NodeRef`はDOM要素の型に対する`Deref`を実装している
            // そのため`HtmlInputElement::value()`を呼び出して
            // 入力の現在値を取得できる
            .value();
        set_name.set(value);
    };

    view! {
        <form on:submit=on_submit>
            <input type="text"
                // ここでは`value`「属性」で初期値だけを設定し、
                // その後の状態の維持はブラウザへ任せる
                value=name

                // この入力への参照を`input_element`へ保存する
                node_ref=input_element
            />
            <input type="submit" value="Submit"/>
        </form>
        <p>"Name is: " {name}</p>
    }
}

// この`main`関数がアプリケーションのエントリーポイントになる
// コンポーネントを<body>へマウントするだけである
// `fn App`として定義したので、テンプレート内で<App/>として使える
fn main() {
    leptos::mount::mount_to_body(App)
}
```

</details>
</preview>
