# グローバル状態管理

ここまではコンポーネント内のローカル状態だけを扱い、親子コンポーネント間で状態を連携する
方法を見てきました。しかし、アプリケーション全体で使える、より一般的なグローバル状態管理の
解決策が必要になる場合もあります。

一般には、**この章の内容は必要ありません。** 通常はすべての状態をグローバルな構造へ
保存するのではなく、それぞれがローカル状態を管理するコンポーネントを組み合わせて
アプリケーションを構築します。ただし、テーマ設定、利用者設定の保存、UIの異なる場所にある
コンポーネント間でのデータ共有など、グローバル状態管理が必要なケースもあります。

グローバル状態を扱う最良の方法は3つあります。

1. Routerを使い、URLを通じてグローバル状態を駆動する
2. コンテキストを通じてシグナルを渡す
3. Storeを使ってグローバル状態の構造体を作成する

## 選択肢1：URLをグローバル状態として使う

多くの点で、URLはグローバル状態を保存する最良の方法です。ツリー内のどこにある
コンポーネントからでもアクセスできます。`<form>` や `<a>` のように、URLを更新するための
ネイティブHTML要素もあります。また、ページを再読み込みしても、デバイスをまたいでも状態が
維持されます。URLを友人と共有したりスマートフォンからノートPCへ送ったりすれば、保存された
状態も再現されます。

この後の数章ではRouterを扱い、これらの話題をさらに詳しく説明します。

ここでは選択肢2と3だけを見ていきます。

## 選択肢2：コンテキストを通じてシグナルを渡す

[親子コンポーネント間の通信](view/08_parent_child.md)では、`provide_context` で親から子へ
シグナルを渡し、子で `use_context` を使って読み取れることを説明しました。しかし
`provide_context` は、どれだけ離れた階層間でも機能します。状態を保持するグローバルな
シグナルを作成したい場合、それを提供したコンポーネントの任意の子孫からコンテキストを通じて
アクセスできます。

コンテキスト経由で提供されたシグナルは、途中のコンポーネントではなく読み取られた場所だけを
リアクティブに更新します。そのため、離れた場所でもきめ細かな更新の利点が保たれます。

まずアプリケーションのルートでシグナルを作成し、`provide_context` を使ってすべての子と
子孫へ提供します。

```rust
#[component]
fn App() -> impl IntoView {
    // ルートに、アプリケーション内のどこからでも利用できるシグナルを作成する
    let (count, set_count) = signal(0);
    // setterは特定のコンポーネントへ渡すが、count自体はコンテキストで全体へ提供する
    provide_context(count);

    view! {
        // SetterButtonはcountを変更できる
        <SetterButton set_count/>
        // これらの利用側は読み取りだけができる
        // 必要なら`set_count`を渡して書き込み権限を与えることもできる
        <FancyMath/>
        <ListItems/>
    }
}
```

`<SetterButton/>` は、これまで何度か作成したものと同じ種類のカウンターです。

`<FancyMath/>` と `<ListItems/>` はどちらも、提供したシグナルを `use_context` で受け取り、
それを使って処理します。

```rust
/// グローバルなcountを使って「高度な」計算を行うコンポーネント
#[component]
fn FancyMath() -> impl IntoView {
    // `use_context`でグローバルなcountシグナルを取得する
    let count = use_context::<ReadSignal<u32>>()
        // 親コンポーネントで提供済みだとわかっている
        .expect("there to be a `count` signal provided");
    let is_even = move || count.get() & 1 == 0;

    view! {
        <div class="consumer blue">
            "The number "
            <strong>{count}</strong>
            {move || if is_even() {
                " is"
            } else {
                " is not"
            }}
            " even."
        </div>
    }
}
```

## 選択肢3：グローバル状態のStoreを作成する

> この内容の一部は、[Storeを使った複雑な反復処理](../view/04b_iteration.md#option-4-stores)の
> 節と重複しています。どちらも中級者向けの任意の内容なので、多少重複しても問題ないと
> 考えました。

Storeは新しいリアクティブプリミティブであり、Leptos 0.7では付属の `reactive_stores`
クレートを通じて利用できます。フレームワーク全体のバージョン変更を必要とせず開発を続けられる
よう、現在は別のクレートとして配布されています。

Storeを使うと構造体全体を包み、ほかのフィールドの変化を追跡せずに個々のフィールドを
リアクティブに読み取り、更新できます。

構造体へ `#[derive(Store)]` を追加して使用します（マクロは
`use reactive_stores::Store;` でインポートできます）。これにより、構造体を `Store<_>` で
包んだときに各フィールドのgetterを提供する拡張トレイトが作成されます。

```rust
#[derive(Clone, Debug, Default, Store)]
struct GlobalState {
    count: i32,
    name: String,
}
```

これによって `GlobalStateStoreFields` というトレイトが作成され、`Store<GlobalState>` へ
`count` と `name` メソッドが追加されます。各メソッドはリアクティブなStoreの
*フィールド*を返します。

```rust
#[component]
fn App() -> impl IntoView {
    provide_context(Store::new(GlobalState::default()));

    // 以下省略
}

/// グローバル状態のcountを更新するコンポーネント。
#[component]
fn GlobalStateCounter() -> impl IntoView {
    let state = expect_context::<Store<GlobalState>>();

    // `count`フィールドだけへリアクティブにアクセスできる
    let count = state.count();

    view! {
        <div class="consumer blue">
            <button
                on:click=move |_| {
                    *count.write() += 1;
                }
            >
                "Increment Global Count"
            </button>
            <br/>
            <span>"Count is: " {move || count.get()}</span>
        </div>
    }
}
```

このボタンをクリックすると `state.count` だけが更新されます。別の場所で `state.name` を
読み取っていても、ボタンのクリックはそこへ通知されません。これにより、トップダウンの
データフローときめ細かなリアクティブ更新の利点を組み合わせられます。

さらに詳しい例は、リポジトリ内の [`stores` の例](https://github.com/leptos-rs/leptos/blob/main/examples/stores/src/lib.rs)をご覧ください。
