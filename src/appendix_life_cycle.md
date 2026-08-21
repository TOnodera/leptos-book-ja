# 付録：Signalのライフサイクル

Leptosを中級レベルで使うと、よく3つの疑問が生じます。
1. コンポーネントのライフサイクルへ接続し、mount時やunmount時にコードを実行するにはどうすればよいでしょうか？
2. signalがいつdisposeされるかを知るにはどうすればよく、dispose済みのsignalへアクセスすると、ときどきpanicするのはなぜでしょうか？
3. signalが `Copy` で、明示的にcloneせずクロージャやほかの構造体へmoveできるのはなぜでしょうか？

この3つの答えは密接に関係し、それぞれやや複雑です。この付録では答えを理解するための背景を示し、アプリケーションのコードとその実行方法について正しく推論できるようにします。

## コンポーネントツリーと決定木

次の単純なLeptosアプリを考えます。

```rust
use leptos::logging::log;
use leptos::prelude::*;

#[component]
pub fn App() -> impl IntoView {
    let (count, set_count) = signal(0);

    view! {
        <button on:click=move |_| *set_count.write() += 1>"+1"</button>
        {move || if count.get() % 2 == 0 {
            view! { <p>"偶数なら問題ありません。"</p> }.into_any()
        } else {
            view! { <InnerComponent count/> }.into_any()
        }}
    }
}

#[component]
pub fn InnerComponent(count: ReadSignal<usize>) -> impl IntoView {
    Effect::new(move |_| {
        log!("countは奇数で、値は{}です", count.get());
    });

    view! {
        <OddDuck/>
        <p>{count}</p>
    }
}

#[component]
pub fn OddDuck() -> impl IntoView {
    view! {
        <p>"あなたは変わり者ですね。"</p>
    }
}
```

このアプリはcounterボタンを表示し、値が偶数なら1つのメッセージ、奇数なら別のメッセージを表示するだけです。奇数の場合は、値をコンソールへも記録します。

この単純なアプリケーションを図示する方法の1つは、ネストしたコンポーネントのツリーを描くことです。
```
App 
|_ InnerComponent
   |_ OddDuck
```

もう1つの方法は、判断箇所のツリーを描くことです。
```
root
|_ countは偶数か？
   |_ はい
   |_ いいえ
```

2つを組み合わせると、完全には対応しないことに気づきます。決定木は `InnerComponent` で作ったviewを3つに分割し、`InnerComponent` の一部を `OddDuck` コンポーネントと組み合わせます。
```
判断                コンポーネント      データ  副作用
root                <App/>              (count) <button>をレンダリング
|_ countは偶数か？  <InnerComponent/>
   |_ はい                                      偶数用の<p>をレンダリング
   |_ いいえ                                    countのログ記録を開始
                    <OddDuck/>                  奇数用の<p>をレンダリング
                                                奇数用の<p>をレンダリング（<InnerComponent/>内！）
```

この表から、次のことに気づきます。
1. コンポーネントツリーと決定木は対応しません。「countは偶数か？」という判断が `<InnerComponent/>` を3つの部分（常に変わらない部分、偶数の場合、奇数の場合）へ分け、そのうち1つを `<OddDuck/>` コンポーネントと統合します。
2. 決定木と副作用の一覧は完全に対応します。各副作用は特定の判断箇所で作成されます。
3. 決定木とデータのツリーも対応します。表にsignalが1つしかないため分かりにくいものの、複数の判断を含むことも、1つも含まないこともある関数であるコンポーネントとは異なり、signalは常に決定木の特定の行で作成されます。

重要なのは、データの構造と副作用の構造がアプリケーションの実際の機能へ影響することです。コンポーネントの構造は記述上の都合にすぎません。どのコンポーネントがどの `<p>` タグをレンダリングしたか、どのコンポーネントが値を記録するEffectを作ったかは、気にする必要も、気にすべきでもありません。重要なのは、適切な時点で起こることだけです。

Leptosでは、*コンポーネントは存在しません。* つまり、便利なのでアプリケーションをコンポーネントのツリーとして記述でき、同じく便利なのでコンポーネントを軸にしたデバッグツールやログ機能も提供しています。しかし実行時にコンポーネントは存在しません。コンポーネントは変更検出やレンダリングの単位ではなく、単なる関数呼び出しです。アプリケーション全体を1つの大きなコンポーネントに書いても、100個のコンポーネントへ分割しても、実行時の挙動には影響しません。コンポーネントは実際には存在しないからです。

一方、決定木は*実在します*。そして非常に重要です！

## 決定木、レンダリング、所有権

各判断箇所は、何らかのリアクティブな文です。つまり、時間とともに変わりうるsignalまたは関数です。signalや関数をrendererへ渡すと、rendererはそれを自動的にEffectで囲み、含まれるsignalをsubscribeして、時間とともにviewを適切に更新します。

つまりアプリケーションがレンダリングされると、決定木を完全に反映する、ネストしたEffectのツリーが作成されます。擬似コードでは次のようになります。
```rust
// root
let button = /* <button>を一度レンダリングする */;

// rendererが`move || if count() ...`をEffectで囲む
Effect::new(|_| {
    if count.get() % 2 == 0 {
        let p = /* 偶数用の<p>をレンダリングする */;
    } else {
        // ユーザーがcountを記録するEffectを作成した
        Effect::new(|_| {
            log!("countは奇数で、値は{}です", count.get());
        });

        let p1 = /* OddDuckの<p>をレンダリングする */;
        let p2 = /* 2つ目の<p>をレンダリングする */;

        // rendererが2つ目の<p>を更新するEffectを作成する
        Effect::new(|_| {
            // signalで<p>の内容を更新する
            p2.set_text_content(count.get());
        });
    }
})
```

各リアクティブ値は独自のEffectで囲まれ、DOMを更新するか、signalの変更によるほかの副作用を実行します。しかし、これらのEffectを永遠に動かす必要はありません。たとえば `count` が奇数から偶数へ戻ると、2つ目の `<p>` は存在しなくなるため、それを更新し続けるEffectも不要です。Effectは永遠に動く代わりに、作成元の判断が変わったときにcancelされます。より正確に言うと、作成時に実行中だったEffectが再実行されるたびに、そのEffectはcancelされます。条件分岐内で作成され、Effectの再実行が同じ分岐を通れば、もう一度作成されます。通らなければ作成されません。

リアクティブシステム自体の視点では、アプリケーションの「決定木」は実際にはリアクティブな「所有権ツリー」です。簡単に言えば、リアクティブな「owner」は現在実行中のEffectまたはMemoです。その内部で作成されたEffectを所有し、それらがさらに自身の子を所有します。Effectは再実行される前に、まず子を「cleanup」してから再び動きます。

ここまでのモデルはS.jsやSolidなどのJavaScriptフレームワークのリアクティブシステムと共通で、所有権の概念はEffectを自動的にcancelするために存在します。

Leptosでは所有権へ、これに似た2つ目の意味を加えます。リアクティブownerは、cancelするために子Effectを所有するだけでなく、disposeするためにsignal（Memoなど）も所有します。

## 所有権と `Copy` Arena

これこそLeptosをRust UIフレームワークとして使えるようにする革新です。UIでは共有可変性が中心になるため、従来RustでのUI状態管理は困難でした。（単純なcounterボタンだけでも問題が分かります。counterの値を示すテキストnodeを設定する不変アクセスと、クリックハンドラー内の可変アクセスの両方が必要ですが、Rustはまさにそれを防ぐよう設計されています。すべてのRust UIフレームワークはこの事実を前提に設計されます！）従来Rustでイベントハンドラーのようなものを使うには、内部可変性を持つ共有メモリ（`Rc<RefCell<_>>`、`Arc<Mutex<_>>`）を介して通信するprimitiveか、channelを介して共有メモリと通信する方法に頼ります。どちらもイベントリスナーへmoveするため、明示的な `.clone()` が必要になることがよくあります。動作はしますが、非常に不便でもあります。

Leptosは代わりに、signalへ一種のArena Allocationを一貫して使ってきました。signal自体は本質的に、別の場所で保持されるデータ構造へのindexです。自身ではreference countingを行わない、コピーコストの低い整数型なので、明示的にcloneせず、あちこちへコピーしたりイベントリスナーへmoveしたりできます。

Rustのlifetimeやreference countingの代わりに、所有権ツリーがこれらのsignalのライフサイクルを決めます。

すべてのEffectが所有元の親Effectに属し、ownerの再実行時に子がcancelされるのと同じように、すべてのsignalもownerに属し、親の再実行時にdisposeされます。

ほとんどの場合、これでまったく問題ありません。上の例で `<OddDuck/>` が別のsignalを作り、UIの一部の更新に使うとしましょう。多くの場合、そのsignalはコンポーネント内のlocal stateとして使うか、別のコンポーネントへpropとして渡します。決定木の上へ持ち上げ、アプリケーションの別の場所で使うのはまれです。`count` が偶数へ戻れば不要になり、disposeできます。

ただし、ここから2つの問題が生じる可能性があります。

### Signalはdispose後にも使えてしまう

保持している `ReadSignal` や `WriteSignal` は単なる整数です。たとえばアプリケーションで3番目のsignalなら3です。（いつものように現実は少し複雑ですが、大差はありません。）その数値をあちこちへコピーし、「signal 3を取得して」と指定できます。ownerがcleanupするとsignal 3の*値*は無効になりますが、あちこちへコピーした数値3は無効にできません。（完全なgarbage collectorなしには！）つまりsignalを決定木の「上」へ押し戻し、作成位置より概念的に「上位」の場所へ保存すると、dispose後にもアクセスできてしまいます。

dispose後のsignalを*更新*しようとしても、実際には悪いことは起こりません。存在しないsignalを更新しようとしたという警告がフレームワークから出るだけです。しかし*アクセス*しようとした場合、panic以外に筋の通った答えはありません。返せる値が存在しないためです。（`.get()` と `.with()` には `try_` 版があり、signalがdispose済みなら単に `None` を返します。）

### 上位scopeで作ったSignalをdisposeしなければleakする

反対も成り立ち、特に `RwSignal<Vec<RwSignal<_>>>` のようなsignalのcollectionを扱うときに問題になります。上位でsignalを作り、下位のコンポーネントへ渡すと、上位のownerがcleanupされるまでdisposeされません。

たとえば各Todoに新しい `RwSignal<Todo>` を作り、`RwSignal<Vec<RwSignal<Todo>>>` へ保存して `<Todo/>` へ渡すTodoアプリでは、一覧からTodoを削除してもsignalは自動的にdisposeされません。手動でdisposeしなければ、ownerが生存する限り「leak」します。（詳しくは [TodoMVCの例](https://github.com/leptos-rs/leptos/blob/main/examples/todomvc/src/lib.rs#L77-L85)を参照してください。）

これはsignalを作ってcollectionへ保存し、手動でdisposeせずにcollectionから削除した場合にだけ問題になります。

### Reference CountingされるSignalで問題を解決する

0.7では、Arena Allocationされる各primitiveに対し、reference countingされる同等物が導入されました。各 `RwSignal` に対して `ArcRwSignal` があります（`ArcReadSignal`、`ArcWriteSignal`、`ArcMemo` などもあります）。

これらのメモリとdisposeは、所有権ツリーではなくreference countingで管理されます。

そのため、Arena Allocation版ではleakしたりdispose後に使われたりする状況でも、安全に利用できます。

これはsignalのcollectionを作るときに特に便利です。たとえば `RwSignal<_>` の代わりに `ArcRwSignal<_>` を作り、表の各行で `RwSignal<_>` へ変換できます。

より具体的な例として、[`counters` の例](https://github.com/leptos-rs/leptos/blob/main/examples/counters/src/lib.rs)にある `ArcRwSignal<i32>` の利用方法を参照してください。

## すべてを結びつける

最初に挙げた疑問への答えも、これで理解できるでしょう。

### コンポーネントのライフサイクル

コンポーネントは実際には存在しないため、コンポーネントのライフサイクルもありません。しかし所有権のライフサイクルがあり、それを使って同じことを実現できます。
- *mount前*：コンポーネント本体でコードを実行するだけで、「コンポーネントがmountする前」に動きます。
- *mount時*：`create_effect` はコンポーネントの残りより1tick後に動くため、viewがDOMへmountされるのを待つ必要があるEffectに役立ちます。
- *unmount時*：`on_cleanup` を使うと、現在のownerが再実行前にcleanupしている間に動かすコードをリアクティブシステムへ渡せます。ownerは「判断」を囲むため、これはコンポーネントのunmount時に `on_cleanup` が動くことを意味します。何かをunmountできるなら、rendererはそれをunmountするEffectを作成しているはずです！

### dispose済みSignalの問題

一般に、この問題は所有権ツリーの下位でsignalを作り、どこか上位へ保存した場合にだけ起こります。問題に遭遇したら、signalの作成を親へ「hoist」してから、作成したsignalを下へ渡してください。必要なら削除時にdisposeすることも忘れないでください！

### `Copy` なsignal

`Copy` 可能なwrapper型（signal、`StoredValue` など）のシステム全体は、UIの各部分のライフサイクルに近い近似として所有権ツリーを使います。実質的には、コードブロックに基づくRust言語のlifetimeシステムと並行して、UIの区画に基づくlifetimeシステムを設けています。常にコンパイル時に完全に検査できるわけではありませんが、総合的には利点のほうが大きいと考えています。
