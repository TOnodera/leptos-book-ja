# シグナルを扱う

ここまでは、[`ReadSignal`](https://docs.rs/leptos/latest/leptos/reactive/signal/struct.ReadSignal.html)
のgetterと [`WriteSignal`](https://docs.rs/leptos/latest/leptos/reactive/signal/struct.WriteSignal.html)
のsetterを返す [`signal`](https://docs.rs/leptos/latest/leptos/reactive/signal/fn.signal.html) の
簡単な使用例を見てきました。

## 値の取得と設定

シグナルには、いくつかの基本操作があります。

### 取得

1. [`.read()`](https://docs.rs/leptos/latest/leptos/reactive/signal/struct.ReadSignal.html#impl-Read-for-T) は、シグナルの値へデリファレンスできる読み取りガードを返し、以後の値の変化をリアクティブに追跡します。このガードが破棄されるまではシグナルの値を更新できず、更新しようとすると実行時エラーになることに注意してください。
1. [`.with()`](https://docs.rs/leptos/latest/leptos/reactive/signal/struct.ReadSignal.html#impl-With-for-T) は関数を受け取ります。その関数はシグナルの現在値を参照（`&T`）として受け取り、シグナルを追跡します。
1. [`.get()`](https://docs.rs/leptos/latest/leptos/reactive/signal/struct.ReadSignal.html#impl-Get-for-T) はシグナルの現在値をクローンし、以後の値の変化を追跡します。

シグナルへのアクセスには `.get()` が最もよく使われます。値をクローンせず、不変参照を受け取る
メソッドを使う場合は `.read()` が便利です（`my_vec_signal.read().len()`）。参照を使って
より多くの処理を行いながら、必要以上に長くロックを保持したくない場合は `.with()` が
役立ちます。

### 設定

1. [`.write()`](https://docs.rs/leptos/latest/leptos/reactive/signal/struct.WriteSignal.html#impl-Write-for-WriteSignal%3CT,+S%3E) は、シグナルの値への可変参照となる書き込みガードを返し、購読者へ更新が必要だと通知します。このガードが破棄されるまではシグナルの値を読み取れず、読み取ろうとすると実行時エラーになります。
1. [`.update()`](https://docs.rs/leptos/latest/leptos/reactive/signal/struct.WriteSignal.html#impl-Update-for-T) は、シグナルの現在値への可変参照（`&mut T`）を受け取る関数を取り、購読者へ通知します。`.update()` はクロージャの戻り値を返しませんが、必要なら [`.try_update()`](https://docs.rs/leptos/latest/leptos/trait.SignalUpdate.html#tymethod.try_update) を使えます。たとえば `Vec<_>` から項目を削除し、その項目を受け取りたい場合です。
1. [`.set()`](https://docs.rs/leptos/latest/leptos/reactive/signal/struct.WriteSignal.html#impl-Set-for-T) はシグナルの現在値を置き換え、購読者へ通知します。

新しい値の設定には `.set()` が最も一般的で、値をその場で更新するには `.write()` が非常に
便利です。`.read()` と `.with()` の関係と同様に、意図したより長く書き込みロックを保持する
可能性を避けたい場合は `.update()` が役立ちます。

```admonish note
これらのトレイトはトレイトの合成に基づき、包括実装によって提供されます。たとえば `Read` は
`Track` と `ReadUntracked` を実装するすべての型へ実装されます。`With` は `Read` を実装する
すべての型へ、`Get` は `With` と `Clone` を実装するすべての型へ実装されます。

`Write`、`Update`、`Set` にも同様の関係があります。

ドキュメントを読む際には、この点を覚えておく価値があります。実装済みのトレイトとして
`ReadUntracked` と `Track` しか表示されていなくても、`.with()` や `.get()`（`T: Clone` の
場合）などを利用できます。
```

## シグナルを使いこなす

`.get()` と `.set()` は、`.read()` と `.write()`、または `.with()` と `.update()` を使って
実装できることに気づくかもしれません。つまり `count.get()` は
`count.with(|n| n.clone())` や `count.read().clone()` と同じであり、`count.set(1)` は
`count.update(|n| *n = 1)` や `*count.write() = 1` によって実装されています。

もちろん、`.get()` と `.set()` の方が記述しやすい構文です。

しかし、ほかのメソッドが非常に適している場面もあります。

たとえば、`Vec<String>` を保持するシグナルを考えてみましょう。

```rust
let (names, set_names) = signal(Vec::new());
if names.get().is_empty() {
	set_names(vec!["Alice".to_string()]);
}
```

ロジックとしては十分単純ですが、かなり非効率な処理が隠れています。
`names.get().is_empty()` は値をクローンすることを思い出してください。つまり
`Vec<String>` 全体をクローンし、`is_empty()` を実行した直後に、そのクローンを
破棄しています。

同様に `set_names` は値をまったく新しい `Vec<_>` へ置き換えます。これでも問題はありませんが、
元の `Vec<_>` をその場で変更することもできます。

```rust
let (names, set_names) = signal(Vec::new());
if names.read().is_empty() {
	set_names.write().push("Alice".to_string());
}
```

これで関数は `names` を参照として受け取って `is_empty()` を実行するだけになり、クローンを
避けたうえで `Vec<_>` をその場で変更します。

## スレッド安全性とスレッドローカルな値

ドキュメントを読んだり自分のアプリケーションで試したりして、シグナルへ保存する値は
`Send + Sync` でなければならないと気づいたかもしれません。リアクティブシステムが実際に
マルチスレッドをサポートしているためです。シグナルはスレッド間で送信でき、リアクティブ
グラフ全体も複数のスレッドをまたいで動作できます（Tokioのマルチスレッドexecutorを使う
Axumなどのサーバーフレームワークで[サーバーサイドレンダリング](../ssr/)を行う際に、
特に役立ちます）。通常のRustデータ型はデフォルトで `Send + Sync` なので、多くの場合、
この制約が作業へ影響することはありません。

しかし、Web Workerを使わない限りブラウザ環境はシングルスレッドであり、`wasm-bindgen` と
`web-sys` が提供するJavaScript型はすべて明示的に `!Send` です。そのため、通常の
シグナルへ保存することはできません。

そこで、各シグナルプリミティブには `!Send` データを保存できる「ローカル」版が用意されて
います。シグナルへ保存する必要がある `!Send` のブラウザ型を扱う場合にだけ使用してください。

| 標準 | ローカル |
| ---- | -------- |
| [`signal`](https://docs.rs/leptos/latest/leptos/reactive/signal/fn.signal.html) | [`signal_local`](https://docs.rs/leptos/latest/leptos/prelude/fn.signal_local.html) |
| [`RwSignal::new`](https://docs.rs/leptos/latest/leptos/prelude/struct.RwSignal.html#method.new) | [`RwSignal::new_local`](https://docs.rs/leptos/latest/leptos/prelude/struct.RwSignal.html#method.new_local) |
| [`Resource`](https://docs.rs/leptos/latest/leptos/prelude/struct.Resource.html) | [`LocalResource`](https://docs.rs/leptos/latest/leptos/prelude/struct.LocalResource.html) |
| [`Action::new`](https://docs.rs/leptos/latest/leptos/prelude/struct.Action.html#method.new) | [`Action::new_local`](https://docs.rs/leptos/latest/leptos/prelude/struct.Action.html#method.new_local), [`Action::new_unsync`](https://docs.rs/leptos/latest/leptos/prelude/struct.Action.html#method.new_unsync) |

## Nightly版の構文

`nightly` featureとnightly版の構文を使う場合、`ReadSignal` を関数として呼び出すのは
`.get()` の糖衣構文です。`WriteSignal` を関数として呼び出すのは `.set()` の糖衣構文です。
したがって、次のコードは

```rust
let (count, set_count) = signal(0);
set_count(1);
logging::log!(count());
```

次のコードと同じです。

```rust
let (count, set_count) = signal(0);
set_count.set(1);
logging::log!(count.get());
```

これは単なる糖衣構文ではありません。シグナルを意味的に関数と同じものにすることで、APIの
一貫性を高めます。詳しくは[幕間](./interlude_functions.md)をご覧ください。

## シグナルを相互に依存させる

あるシグナルを別のシグナルの値に基づいて変化させる必要がある場合について、よく質問を
受けます。これを実現するよい方法が3つと、理想的ではないものの制御された状況なら使える
方法が1つあります。

### よい選択肢

**1）BがAの関数である場合。** Aのシグナルと、Bの派生シグナルまたはメモを作成します。

```rust
// A
let (count, set_count) = signal(1);
// BはAの関数
let derived_signal_double_count = move || count.get() * 2;
// BはAの関数
let memoized_double_count = Memo::new(move |_| count.get() * 2);
```

> 派生シグナルとメモのどちらを使うべきかについては、[`Memo`](https://docs.rs/leptos/latest/leptos/reactive/computed/struct.Memo.html) のドキュメントをご覧ください。

**2）CがAと別の値Bの関数である場合。** AとBのシグナル、およびCの派生シグナルまたは
メモを作成します。

```rust
// A
let (first_name, set_first_name) = signal("Bridget".to_string());
// B
let (last_name, set_last_name) = signal("Jones".to_string());
// CはAとBの関数
let full_name = move || format!("{} {}", &*first_name.read(), &*last_name.read());
```

**3）AとBが独立したシグナルで、ときどき同時に更新される場合。** Aを更新する呼び出しを
行う際に、Bを更新する別の呼び出しも行います。

```rust
// A
let (age, set_age) = signal(32);
// B
let (favorite_number, set_favorite_number) = signal(42);
// `Clear`ボタンのクリック処理に使う
let clear_handler = move |_| {
  // AとBの両方を更新する
  set_age.set(0);
  set_favorite_number.set(0);
};
```

### どうしても必要なら……

**4）Aが変化するたびにBへ書き込むエフェクトを作成する。** いくつかの理由から、これは
公式には推奨されていません。

a）Aが更新されるたびにリアクティブ処理を完全に2周するため、必ず効率が悪くなります。
まずAを設定するとエフェクトとAに依存するほかのエフェクトが実行され、次にBを設定すると
Bに依存するエフェクトが実行されます。

b）無限ループやエフェクトの過剰な再実行を誤って作り出す可能性が高まります。このように
処理が往復するリアクティブなスパゲッティコードは2010年代初頭によく見られました。
読み書きの分離や、エフェクトからシグナルへ書き込むことを非推奨にすることで、私たちは
これを避けようとしています。

ほとんどの状況では、派生シグナルやメモに基づく明確なトップダウンのデータフローになるよう、
書き直すのが最善です。とはいえ、この方法が致命的というわけではありません。

> ここでは意図的に例を示していません。どのように動作するかは、[`Effect`](https://docs.rs/leptos/latest/leptos/reactive/effect/struct.Effect.html) のドキュメントを読んで確認してください。
