# ガイド：Island

Leptos 0.5では、新しい `islands` featureが導入されました。このガイドでは、Island Architectureを使ったデモアプリを実装しながら、islands featureと中心的な概念を説明します。

## Island Architecture

主要なJavaScriptフロントエンドフレームワーク（React、Vue、Svelte、Solid、Angular）はどれも、クライアントレンダリングするシングルページアプリ（SPA）を構築するフレームワークとして生まれました。最初のページ読み込みをHTMLへレンダリングしてハイドレートし、それ以降のナビゲーションはクライアントで直接処理します。（それが「シングルページ」の由来です。後でクライアント側routingが行われても、すべてはサーバーからの1回のページ読み込みを起点に起こります。）これらのフレームワークは後に、初期読み込み時間、SEO、ユーザー体験を改善するためサーバーサイドレンダリングを追加しました。

つまりデフォルトではアプリ全体が操作可能です。同時に、ハイドレートするためアプリ全体をJavaScriptとしてクライアントへ送らなければなりません。Leptosも同じパターンに従ってきました。

> 詳しくは[サーバーサイドレンダリング](./ssr/22_life_cycle.md)の章で読めます。

しかし、反対方向から進めることもできます。完全に操作可能なアプリをサーバー上でHTMLへレンダリングし、ブラウザでハイドレートする代わりに、普通のHTMLページから始めて小さな対話領域を追加します。これは2010年代以前のWebサイトやアプリで伝統的だった形式です。ブラウザがサーバーへ一連のリクエストを送り、レスポンスとして各新規ページのHTMLを受け取ります。「シングルページアプリ」（SPA）の台頭後、この手法は対比して「マルチページアプリ」（MPA）と呼ばれることがあります。

近年「Island Architecture」という言葉が、サーバーレンダリングしたHTMLページの「海」から始め、ページの随所へ対話性の「島」を追加する手法を表すために登場しました。

> ### 関連資料
>
> このガイドの残りでは、LeptosでIslandを使う方法を見ていきます。この手法全般の背景については、次の記事も参照してください。
>
> - Jason Miller, [“Islands Architecture”](https://jasonformat.com/islands-architecture/), Jason Miller
> - Ryan Carniato, [“Islands & Server Components & Resumability, Oh My!”](https://dev.to/this-is-learning/islands-server-components-resumability-oh-my-319d)
> - [“Islands Architectures”](https://www.patterns.dev/posts/islands-architecture) on patterns.dev
> - [Astro Islands](https://docs.astro.build/en/concepts/islands/)

## Islands Modeを有効にする

新しい `cargo-leptos` アプリから始めましょう。

```bash
cargo leptos new --git leptos-rs/start-axum
```

> この例では、ActixとAxumに実質的な違いはないはずです。

次のコマンドをバックグラウンドで実行し、

```bash
cargo leptos build
```

その間にエディターを起動してコードを書き続けます。

最初に `Cargo.toml` へ `islands` featureを追加します。これは `leptos` crateだけに追加すれば十分です。

```toml
leptos = { version = "0.7", features = ["islands"] }
```

次に `src/lib.rs` からexportされる `hydrate` 関数を変更します。`leptos::mount::hydrate_body(App)` を呼ぶ行を削除し、次に置き換えます。

```rust
leptos::mount::hydrate_islands();
```

これはアプリケーション全体を実行して、作成されたviewをハイドレートする代わりに、個々のIslandを順番にハイドレートします。

`app.rs` の `shell` 関数では、`HydrationScripts` コンポーネントへ `islands=true` も追加する必要があります。

```rust
<HydrationScripts options islands=true/>
```

それでは `cargo leptos watch` を起動し、[`http://localhost:3000`](http://localhost:3000)（または設定した場所）を開きます。

ボタンをクリックすると……

何も起こりません！

完璧です。

```admonish note
スターターテンプレートの `hydrate()` 関数定義には `use app::*;` が含まれています。Islands Modeへ切り替えると、importしたmainの `App` 関数を使わなくなるので、これを削除できると思うかもしれません。（実際、削除しなければRustのlintツールが警告することもあります！）

しかしworkspace構成では、これが問題を起こすことがあります。各関数のentrypointを個別にexportするため `wasm-bindgen` を使っています。私の経験では、workspace構成で `frontend` crate内の何も `app` crateを実際に使っていない場合、それらのbindingは正しく生成されません。[詳しくはこちらのdiscussion](https://github.com/leptos-rs/leptos/issues/2083#issuecomment-1868053733)をご覧ください。
```

## Islandを使う

何も起こらないのは、アプリのメンタルモデルを完全に反転させたからです。デフォルトで操作可能にしてすべてをハイドレートするのではなく、アプリはデフォルトで普通のHTMLとなり、対話性を明示的に有効化する必要があります。

これはWASMバイナリサイズへ大きく影響します。リリースモードでコンパイルすると、このアプリのWASMは（非圧縮で）わずか24KBです。Islands Modeでない場合は274KBです。（274KBは「Hello, world!」にしてはかなり大きいものです。実態は、このデモで使っていないクライアント側routing関連のコードすべてです。）

ページ全体が静的なので、ボタンをクリックしても何も起こりません。

では、どうすれば何かを起こせるでしょう？

`HomePage` コンポーネントをIslandにしましょう！

操作できない版は次のとおりでした。

```rust
#[component]
fn HomePage() -> impl IntoView {
    // ボタンを更新するリアクティブな値を作成する
    let count = RwSignal::new(0);
    let on_click = move |_| *count.write() += 1;

    view! {
        <h1>"Leptosへようこそ！"</h1>
        <button on:click=on_click>"クリック：" {count}</button>
    }
}
```

操作可能な版は次のとおりです。

```rust
#[island]
fn HomePage() -> impl IntoView {
    // ボタンを更新するリアクティブな値を作成する
    let count = RwSignal::new(0);
    let on_click = move |_| *count.write() += 1;

    view! {
        <h1>"Leptosへようこそ！"</h1>
        <button on:click=on_click>"クリック：" {count}</button>
    }
}
```

今度はボタンをクリックすると動きます！

`#[island]` マクロは `#[component]` マクロとまったく同じように動作しますが、Islands Modeでは対象を操作可能なIslandとして指定します。もう一度バイナリサイズを確認すると、リリースモードの非圧縮で166KBです。完全に静的な24KB版よりずっと大きいものの、完全にハイドレートする355KB版よりはるかに小さくなっています。

ここでページのソースを開くと、`HomePage` Islandが特殊な `<leptos-island>` HTML要素としてレンダリングされ、ハイドレートに使うコンポーネントが指定されていることが分かります。

```html
<leptos-island data-component="HomePage_7432294943247405892">
  <h1>Welcome to Leptos!</h1>
  <button>
    Click Me:
    <!>0
  </button>
</leptos-island>
```

この `<leptos-island>` 内部のコードだけがWASMへコンパイルされ、ハイドレーション時にもそのコードだけが実行されます。

## Islandを効果的に使う

WASMへコンパイルしてブラウザへ送る必要があるのは、`#[island]` 内のコード_だけ_だと覚えておいてください。つまりIslandは、できるだけ小さく具体的にすべきです。たとえば `HomePage` は、通常のコンポーネントとIslandへ分割したほうが適切です。

```rust
#[component]
fn HomePage() -> impl IntoView {
    view! {
        <h1>"Welcome to Leptos!"</h1>
        <Counter/>
    }
}

#[island]
fn Counter() -> impl IntoView {
    // ボタンを更新するリアクティブな値を作成する
    let (count, set_count) = signal(0);
    let on_click = move |_| *set_count.write() += 1;

    view! {
        <button on:click=on_click>"クリック：" {count}</button>
    }
}
```

これで `<h1>` をクライアントbundleへ含めたり、ハイドレートしたりする必要はありません。今は無意味な違いに見えるかもしれません。しかし `HomePage` 自体へ操作不能なHTMLコンテンツをどれだけ追加しても、WASMバイナリサイズはまったく変わりません。

通常のハイドレーションモードでは、WASMバイナリサイズはアプリの大きさ／複雑さに応じて増えます。Islands Modeでは、アプリ内の対話性の量に応じて増えます。Islandの外側へ操作不能なコンテンツをいくら追加しても、バイナリサイズは増えません。

## 強力な能力を引き出す

WASMバイナリサイズが50%減るのは魅力的です。しかし、本当の利点は何でしょうか？

重要なのは、次の2つの事実を組み合わせたときです。

1. `#[component]` 関数内のコードは、Island内で使わない限りサーバーで_のみ_実行されます。\*
2. childrenとpropは、WASMバイナリへ含めることなくサーバーからIslandへ渡せます。

つまりサーバー専用コードをコンポーネント本体で直接実行し、その結果をchildrenへ直接渡せます。完全にハイドレートするアプリではserver functionとSuspenseの複雑な組み合わせが必要な一部の処理も、Islandではインラインに書けます。

> \* 「Island内で使わない限り」という部分が重要です。`#[component]` コンポーネントがサーバーだけで動くという意味では_ありません_。正確には「共有コンポーネント」であり、`#[island]` の本体で使われる場合にのみWASMバイナリへコンパイルされます。Island内で使わなければブラウザでは実行されません。

このデモの残りでは、3つ目の事実も利用します。

3. 本来は独立したIsland同士の間でcontextを渡せます。

counterのデモの代わりに、もう少し面白いものを作りましょう。サーバー上のファイルからデータを読み取るタブ式インターフェイスです。

## サーバーのchildrenをIslandへ渡す

Islandの最も強力な点の1つは、サーバーレンダリングしたchildrenについてIsland側が何も知る必要なく、Islandへ渡せることです。Islandは自身のコンテンツをハイドレートしますが、渡されたchildrenはハイドレートしません。

ReactのDan Abramovが（非常によく似たRSCの文脈で）表現したように、Islandは実際には島ではなくドーナツです。いわば「ドーナツの穴」へサーバー専用コンテンツを直接渡し、操作不能なサーバーHTMLの海に_両側_を囲まれた、対話性を持つ小さな環礁を作れます。

> 以下のデモコードでは、すべてのサーバーコンテンツを水色の「海」、すべてのIslandを薄緑の「陸地」として表示するスタイルを追加しました。説明しているものを思い浮かべる助けになれば幸いです！

デモを続け、`Tabs` コンポーネントを作ります。タブの切り替えには対話性が必要なので、もちろんIslandにします。まずは単純なものから始めます。

```rust
#[island]
fn Tabs(labels: Vec<String>) -> impl IntoView {
    let buttons = labels
        .into_iter()
        .map(|label| view! { <button>{label}</button> })
        .collect_view();
    view! {
        <div style="display: flex; width: 100%; justify-content: space-between;">
            {buttons}
        </div>
    }
}
```

おっと、エラーが発生します。

```
error[E0463]: can't find crate for `serde`
  --> src/app.rs:43:1
   |
43 | #[island]
   | ^^^^^^^^^ can't find crate
```

修正は簡単です。`cargo add serde --features=derive` を実行しましょう。`#[island]` マクロは `labels` propをシリアライズ・デシリアライズする必要があるため、ここで `serde` を取り込もうとします。

続いて `HomePage` を更新して `Tabs` を使います。

```rust
#[component]
fn HomePage() -> impl IntoView {
	// これらのファイルを読み込む
    let files = ["a.txt", "b.txt", "c.txt"];
	// タブのラベルにはファイル名をそのまま使う
	let labels = files.iter().copied().map(Into::into).collect();
    view! {
        <h1>"Leptosへようこそ！"</h1>
        <p>"以下のタブをクリックしてレシピを読んでください。"</p>
        <Tabs labels/>
    }
}
```

DOMインスペクターを見ると、Islandが次のようになっていることが分かります。

```html
<leptos-island
  data-component="Tabs_1030591929019274801"
  data-props='{"labels":["a.txt","b.txt","c.txt"]}'
>
  <div style="display: flex; width: 100%; justify-content: space-between;;">
    <button>a.txt</button>
    <button>b.txt</button>
    <button>c.txt</button>
    <!---->
  </div>
</leptos-island>
```

`labels` propはJSONへシリアライズされ、Islandのハイドレートに使えるようHTML属性へ保存されます。

いくつかタブを追加しましょう。今のところ `Tab` Islandは非常に単純です。

```rust
#[island]
fn Tab(index: usize, children: Children) -> impl IntoView {
    view! {
        <div>{children()}</div>
    }
}
```

現時点では、各タブはchildrenを囲む `<div>` にすぎません。

`Tabs` コンポーネントもchildrenを受け取ります。今はすべて表示しましょう。

```rust
#[island]
fn Tabs(labels: Vec<String>, children: Children) -> impl IntoView {
    let buttons = labels
        .into_iter()
        .map(|label| view! { <button>{label}</button> })
        .collect_view();
    view! {
        <div style="display: flex; width: 100%; justify-content: space-around;">
            {buttons}
        </div>
        {children()}
    }
}
```

それでは `HomePage` へ戻り、タブ領域へ入れるタブの一覧を作ります。

```rust
#[component]
fn HomePage() -> impl IntoView {
    let files = ["a.txt", "b.txt", "c.txt"];
    let labels = files.iter().copied().map(Into::into).collect();
	let tabs = move || {
        files
            .into_iter()
            .enumerate()
            .map(|(index, filename)| {
                let content = std::fs::read_to_string(filename).unwrap();
                view! {
                    <Tab index>
                        <h2>{filename.to_string()}</h2>
                        <p>{content}</p>
                    </Tab>
                }
            })
            .collect_view()
    };

    view! {
        <h1>"Leptosへようこそ！"</h1>
        <p>"以下のタブをクリックしてレシピを読んでください。"</p>
        <Tabs labels>
            <div>{tabs()}</div>
        </Tabs>
    }
}
```

えっ……何でしょう？

Leptosを使い慣れているなら、このようなことはできないと知っているでしょう。コンポーネント本体のコードはすべて、サーバー（HTMLへレンダリングするため）とブラウザ（ハイドレートするため）の両方で動く必要があります。そのため `std::fs` を単純には呼べません。ブラウザはローカルファイルシステム（ましてサーバーのファイルシステム！）へアクセスできないためpanicします。アクセスできたらセキュリティ上の悪夢です！

しかし……待ってください。今はIslands Modeです。この `HomePage` コンポーネントは_本当に_サーバー上でしか動きません。したがって実際、このような普通のサーバーコードをそのまま使えます。

> **これは愚かな例でしょうか？** はい！ `.map()` 内で3つの異なるローカルファイルを同期的に読み取るのは、実際のアプリではよい選択ではありません。ここでの要点は、これが間違いなくサーバー専用コンテンツであると示すことだけです。

プロジェクトのルートへ `a.txt`、`b.txt`、`c.txt` という3ファイルを作り、好きな内容を記述してください。

ページを再読み込みすると、ブラウザに内容が表示されるはずです。ファイルを編集してもう一度再読み込みすれば、内容も更新されます。

データへのアクセス方法やコンテンツのレンダリング方法をIsland側が知る必要なく、`#[component]` から `#[island]` のchildrenへサーバー専用コンテンツを渡せます。

**これは非常に重要です。** サーバーの `children` をIslandへ渡せるため、Islandを小さく保てます。理想的には、ページの大きなまとまり全体を `#[island]` で囲みたくはありません。そのまとまりを、`#[island]` にできる操作可能な部分と、`children` としてIslandへ渡せる追加のサーバーコンテンツへ分割します。これにより、ページ内の操作可能な部分に含まれる操作不能な小区分をWASMバイナリの外に置けます。

## Island間でcontextを渡す

これはまだ本当の「タブ」ではありません。常にすべてのタブを表示しているだけです。`Tabs` と `Tab` コンポーネントへ単純なロジックを追加しましょう。

`Tabs` を変更して単純な `selected` signalを作ります。読み取り側をcontextで提供し、いずれかのボタンがクリックされるたびにsignalの値を設定します。

```rust
#[island]
fn Tabs(labels: Vec<String>, children: Children) -> impl IntoView {
    let (selected, set_selected) = signal(0);
    provide_context(selected);

    let buttons = labels
        .into_iter()
        .enumerate()
        .map(|(index, label)| view! {
            <button on:click=move |_| set_selected.set(index)>
                {label}
            </button>
        })
        .collect_view();
// ...
```

さらに `Tab` Islandを変更し、そのcontextを使って自身を表示または非表示にします。

```rust
#[island]
fn Tab(index: usize, children: Children) -> impl IntoView {
    let selected = expect_context::<ReadSignal<usize>>();
    view! {
        <div
            style:background-color="lightgreen"
            style:padding="10px"
            style:display=move || if selected.get() == index {
                "block"
            } else {
                "none"
            }
        >
            {children()}
        </div>
    }
}
```

これでタブは期待どおりに動きます。`Tabs` はcontextを介して各 `Tab` へsignalを渡し、`Tab` はそれを使って開くべきかどうかを判断します。

> これが `HomePage` で `let tabs = move ||` を関数にし、`{tabs()}` のように呼び出した理由です。この方法でタブを遅延作成すると、各 `Tab` が `selected` contextを探す時点では、`Tabs` Islandがすでにそのcontextを提供しています。

完成したタブのデモは非圧縮で約200KBです。世界最小のデモではありませんが、最初に使ったクライアント側routingありの「Hello, world」よりはるかに小さいままです！ 試しにIslands Modeを使わず、`#[server]` 関数と `Suspense` で同じデモを構築すると400KBを超えました。今回もバイナリサイズを約50%削減できたわけです。しかも、このアプリに含まれるサーバー専用コンテンツはごくわずかです。サーバー専用コンポーネントやページを追加しても、この200KBは増えないことを思い出してください。

## 概要

このデモはかなり基本的に見えるかもしれません。実際そうです。しかし、すぐに得られる重要な知見がいくつかあります。

- **WASMバイナリサイズを50%削減**し、クライアントで操作可能になるまでの時間と初期読み込み時間を測定可能なほど改善できます。
- **データのシリアライズコストを削減できます。** Resourceを作りクライアントで読み取るには、ハイドレーションに使えるようデータをシリアライズする必要があります。`Suspense` 内でHTMLを作るためにも同じデータを読んでいれば、「二重データ」になります。つまり、まったく同じデータがHTMLとしてレンダリングされると同時にJSONとしてシリアライズされ、レスポンスサイズと所要時間が増えます。
- サーバーで動く通常のネイティブRust関数のように、`#[component]` 内で**サーバー専用APIを簡単に使えます**。Islands Modeでは、実際そのとおりだからです！
- サーバーデータを読み込むための **`#[server]`／`create_resource`／`Suspense` のboilerplateを削減**できます。

## 今後の検討

`islands` featureは、現在フロントエンドWebフレームワークが探求している最先端の取り組みを反映しています。現状のIsland手法は（最近View Transitionsへ対応する前の）Astroによく似ています。従来型のサーバーレンダリングするマルチページアプリを構築し、対話性を持つIslandを非常に自然に統合できます。

簡単に追加できる小さな改善もあります。たとえばAstroのView Transitions手法によく似た次のことを実現できます。

- それ以降のナビゲーションをサーバーから取得し、HTML documentを新しいものへ置き換えることで、Islandアプリへクライアント側routingを追加する
- View Transitions APIを使い、古いdocumentと新しいdocumentの間へアニメーション付きtransitionを追加する
- 明示的に永続化するIsland、つまり一意のID（view内のコンポーネントに `persist:searchbar` など）を付け、現在の状態を失わず古いdocumentから新しいdocumentへコピーできるIslandへ対応する

ほかにも、私が[まだ納得していない](https://github.com/leptos-rs/leptos/issues/1830)、より大きなアーキテクチャ変更があります。

## 追加情報

詳しい議論については、[`islands` の例](https://github.com/leptos-rs/leptos/blob/main/examples/islands/src/app.rs)、[roadmap](https://github.com/leptos-rs/leptos/issues/1830)、[Hacker Newsのデモ](https://github.com/leptos-rs/leptos/tree/leptos_0.6/examples/hackernews_islands_axum)をご覧ください。

## デモコード

```rust
use leptos::prelude::*;

#[component]
pub fn App() -> impl IntoView {
    view! {
        <main style="background-color: lightblue; padding: 10px">
            <HomePage/>
        </main>
    }
}

/// アプリケーションのホームページをレンダリングする。
#[component]
fn HomePage() -> impl IntoView {
    let files = ["a.txt", "b.txt", "c.txt"];
    let labels = files.iter().copied().map(Into::into).collect();
    let tabs = move || {
        files
            .into_iter()
            .enumerate()
            .map(|(index, filename)| {
                let content = std::fs::read_to_string(filename).unwrap();
                view! {
                    <Tab index>
                        <div style="background-color: lightblue; padding: 10px">
                            <h2>{filename.to_string()}</h2>
                            <p>{content}</p>
                        </div>
                    </Tab>
                }
            })
            .collect_view()
    };

    view! {
        <h1>"Leptosへようこそ！"</h1>
        <p>"以下のタブをクリックしてレシピを読んでください。"</p>
        <Tabs labels>
            <div>{tabs()}</div>
        </Tabs>
    }
}

#[island]
fn Tabs(labels: Vec<String>, children: Children) -> impl IntoView {
    let (selected, set_selected) = signal(0);
    provide_context(selected);

    let buttons = labels
        .into_iter()
        .enumerate()
        .map(|(index, label)| {
            view! {
                <button on:click=move |_| set_selected.set(index)>
                    {label}
                </button>
            }
        })
        .collect_view();
    view! {
        <div
            style="display: flex; width: 100%; justify-content: space-around;\
            background-color: lightgreen; padding: 10px;"
        >
            {buttons}
        </div>
        {children()}
    }
}

#[island]
fn Tab(index: usize, children: Children) -> impl IntoView {
    let selected = expect_context::<ReadSignal<usize>>();
    view! {
        <div
            style:background-color="lightgreen"
            style:padding="10px"
            style:display=move || if selected.get() == index {
                "block"
            } else {
                "none"
            }
        >
            {children()}
        </div>
    }
}
```
