# 付録：リアクティブシステムの仕組み

このライブラリをうまく使うために、リアクティブシステムが実際にどう動くかを詳しく知る必要はありません。しかし、フレームワークを高度なレベルで使い始めたら、舞台裏で起きていることを理解しておくと常に役立ちます。

使用するリアクティブprimitiveは3種類に分かれます。

- **Signal**（`ReadSignal`／`WriteSignal`、`RwSignal`、`Resource`、`Trigger`）：能動的に変更し、リアクティブな更新を引き起こせる値です。
- **計算**（`Memo`）：signal（またはほかの計算）に依存し、何らかの純粋な計算から新しいリアクティブな値を導出します。
- **Effect**：signalや計算の変更を監視して関数を実行し、何らかの副作用を起こすobserverです。

派生signalはprimitiveではない計算の一種です。単なるクロージャとして、signalに基づく繰り返し計算を複数箇所から呼べる再利用可能な関数へリファクタリングできますが、リアクティブシステム自体の中には表現されません。

ほかのprimitiveはすべて、リアクティブグラフのnodeとしてリアクティブシステム内に実在します。

リアクティブシステムの処理の大部分は、途中にMemoを挟むこともある、signalからEffectへの変更伝播です。

リアクティブシステムは、（DOMへのレンダリングやネットワークリクエストなどの）Effectが、アプリ内のRustデータ構造の更新などより桁違いに高コストだと想定します。

そのためリアクティブシステムの**主な目標**は、**Effectの実行回数をできるだけ少なくすること**です。

Leptosはリアクティブグラフを構築してこれを実現します。

> 現在のLeptosのリアクティブシステムは、JavaScript用の [Reactively](https://github.com/modderme123/reactively) ライブラリを強く参考にしています。Miloの記事[「Super-Charging Fine-Grained Reactivity」](https://dev.to/modderme123/super-charging-fine-grained-reactive-performance-47ph)では、そのアルゴリズムとfine-grained reactivity全般について、美しい図も交えた優れた解説を読めます！

## リアクティブグラフ

Signal、Memo、Effectはすべて3つの特性を共有します。

- **値**：現在の値を持ちます。signalの値、または（MemoとEffectでは）以前の実行が返した値があればその値です。
- **Source**：依存先となるほかのリアクティブprimitiveです。（signalでは空集合です。）
- **Subscriber**：それらに依存するほかのリアクティブprimitiveです。（Effectでは空集合です。）

つまり実際には、signal、Memo、Effectはリアクティブグラフの「node」という1つの一般概念に付けられた慣習的な名前にすぎません。Signalは常にsource／親を持たない「root node」です。Effectは常にsubscriberを持たない「leaf node」です。Memoは通常、sourceとsubscriberの両方を持ちます。

> 以下の例では `nightly` 構文を使います。コピー＆ペーストではなく読んでもらうための文書なので、単に冗長さを減らすためです！

### 単純な依存関係

次のコードを想像してください。

```rust
// A
let (name, set_name) = signal("Alice");

// B
let name_upper = Memo::new(move |_| name.with(|n| n.to_uppercase()));

// C
Effect::new(move |_| {
	log!("{}", name_upper());
});

set_name("Bob");
```


このリアクティブグラフは簡単に想像できます。`name` が唯一のsignal／起点node、`Effect::new` が唯一のEffect／終端nodeで、その間にMemoが1つあります。

```
A   (name)
|
B   (name_upper)
|
C   (Effect)
```

### 分岐する

もう少し複雑にしましょう。

```rust
// A
let (name, set_name) = signal("Alice");

// B
let name_upper = Memo::new(move |_| name.with(|n| n.to_uppercase()));

// C
let name_len = Memo::new(move |_| name.len());

// D
Effect::new(move |_| {
	log!("len = {}", name_len());
});

// E
Effect::new(move |_| {
	log!("name = {}", name_upper());
});
```

これもかなり単純です。source signal（`name`／`A`）が、`name_upper`／`B` と `name_len`／`C` という2つの並行経路へ分かれ、それぞれに依存するEffectがあります。

```
 __A__
|     |
B     C
|     |
E     D
```

signalを更新しましょう。

```rust
set_name("Bob");
```

すぐに次のログが出ます。

```
len = 3
name = BOB
```

もう一度行います。

```rust
set_name("Tim");
```

ログには次が表示されるはずです。

```
name = TIM
```

`len = 3` は再び記録されません。

リアクティブシステムの目標は、Effectの実行回数をできるだけ少なくすることでした。`name` を `"Bob"` から `"Tim"` へ変更すると、各Memoは再実行されます。しかし実際に値が変わった場合にのみsubscriberへ通知します。`"BOB"` と `"TIM"` は異なるため、そのEffectは再実行されます。一方、どちらの名前も長さは `3` なので、そちらは再実行されません。

### 分岐を再び合流させる

もう1つ、**ダイヤモンド問題**と呼ばれることがある例です。

```rust
// A
let (name, set_name) = signal("Alice");

// B
let name_upper = Memo::new(move |_| name.with(|n| n.to_uppercase()));

// C
let name_len = Memo::new(move |_| name.len());

// D
Effect::new(move |_| {
	log!("{} is {} characters long", name_upper(), name_len());
});
```

このグラフはどのようになるでしょう？

```
 __A__
|     |
B     C
|     |
|__D__|
```

「ダイヤモンド問題」と呼ばれる理由が分かるでしょう。拙いASCIIアートではなく直線でnodeを結べば、ダイヤモンド形になります。signalにそれぞれ依存する2つのMemoが、同じEffectへつながっています。

単純なpush型のリアクティブ実装では、このEffectが2回実行されてしまい望ましくありません。（目標はEffectの実行をできるだけ少なくすることでした。）たとえばsignalとMemoが、各依存関係を通じて変更をグラフの末端まですぐに伝播し、実質的に深さ優先でグラフをたどるリアクティブシステムを実装できます。言い換えると、`A` の更新が `B` へ通知され、`B` が `D` へ通知します。その後 `A` が `C` へ通知し、`C` がもう一度 `D` へ通知します。これは非効率（`D` が2回動く）なうえ、一時的な不整合も起こします（`D` の初回実行時には、2つ目のMemoが誤った値のままです）。

## ダイヤモンド問題を解決する

有用なリアクティブ実装なら、どれもこの問題の解決に取り組んでいます。さまざまな手法があります（優れた概説として、ここでも[Miloの記事](https://dev.to/modderme123/super-charging-fine-grained-reactive-performance-47ph)を参照してください）。

Leptosの仕組みを簡単に説明します。

リアクティブnodeは常に3つの状態のいずれかにあります。

- `Clean`：変更されていないことが分かっている
- `Check`：変更された可能性がある
- `Dirty`：確実に変更された

signalを更新すると、そのsignalを `Dirty`、すべての子孫を再帰的に `Check` としてmarkします。子孫に含まれるEffectは、再実行するqueueへ追加されます。

```
    ____A (DIRTY)___
   |               |
B (CHECK)    C (CHECK)
   |               |
   |____D (CHECK)__|
```

続いて、それらのEffectを実行します。（この時点ではすべてのEffectが `Check` とmarkされています。）計算を再実行する前に、Effectは親がdirtyかどうかを確認します。

- `D` は `B` へ移り、`Dirty` かどうかを確認します。
- しかし `B` も `Check` とmarkされています。そこで `B` も同じ処理をします。
  - `B` は `A` へ移り、`Dirty` であることを確認します。
  - sourceの1つが変わったため、`B` は再実行する必要があります。
  - `B` は再実行して新しい値を生成し、自身を `Clean` とmarkします。
  - `B` はMemoなので、以前の値と新しい値を比較します。
  - 同じなら `B` は「変更なし」、異なれば「変更あり」を返します。
- `B` が「変更あり」を返したら、`D` は確実に実行が必要だと分かり、ほかのsourceを確認する前にすぐ再実行します。
- `B` が「変更なし」を返したら、`D` は続いて `C` を確認します（`B` について説明した上の処理と同じです）。
- `B` と `C` のどちらも変わっていなければ、Effectを再実行する必要はありません。
- `B` または `C` のどちらかが変わっていれば、Effectを再実行します。

Effectは一度だけ `Check` とmarkされ、一度だけqueueへ追加されるため、実行も一度だけです。

単純な版はリアクティブな変更をグラフの末端まで押し込み、その結果Effectを2回実行する「push型」リアクティブシステムでした。この版は「push-pull型」と呼べます。`Check` 状態をグラフの末端までpushしてから、上へ向かってpullします。実際、大きなグラフでは、どのnodeを再実行すべきか正確に判断するため、上下左右を行き来することもあります。

**この重要なtrade-offに注目してください**。push型のリアクティビティはsignalの変更をより速く伝播する代わりに、MemoとEffectを過剰に再実行します。リアクティブシステムは、副作用がライブラリのRustコード内だけで行われるこの種のcache-friendlyなグラフ走査より桁違いに高コストだという（正確な）想定に基づき、Effectの再実行回数を最小化するよう設計されています。優れたリアクティブシステムの尺度は、変更を伝播する速さではなく、_過剰に通知せず_変更を伝播する速さです。

## MemoとSignal

signalは常に子へ通知することに注意してください。つまり新しい値が古い値と同じでも、signalは更新時に必ず `Dirty` とmarkされます。そうしなければsignalに `PartialEq` を要求する必要があり、一部の型では実際かなり高コストな確認になります。（たとえば明らかに値が変わった `some_vec_signal.update(|n| n.pop())` のような処理へ、不要な等値比較を追加することになります。）

一方Memoは、子へ通知する前に自身が変わったかどうかを確認します。結果を何度 `.get()` しても計算は一度しか実行しませんが、source signalが変わるたびに実行します。つまりMemoの計算が_非常に_高コストなら、入力もMemo化し、入力が変わったと確信できるときだけ再計算させたほうがよい場合があります。

## Memoと派生Signal

これはすべて優れた仕組みで、Memoも非常に便利です。しかし実際のアプリケーションの多くは、かなり浅く幅広いリアクティブグラフを持ちます。100個のsource signalと500個のEffectがあってもMemoはないか、まれにsignalとEffectの間へ3～4個ある程度です。Memoは、変更をsubscriberへ通知する回数を抑えるという役割を非常にうまく果たします。しかしここまでの説明から分かるように、次の形のoverheadがあります。

1. 高コストな場合も、そうでない場合もある `PartialEq` の確認。
2. リアクティブシステムへ別のnodeを保存するための追加メモリコスト。
3. リアクティブグラフを走査する追加の計算コスト。

計算自体がこのリアクティブ処理より安価な場合は、Memoで「過剰に包む」のを避け、単に派生signalを使うべきです。次はMemoを決して使うべきでない好例です。

```rust
let (a, set_a) = signal(1);
// どれもMemoにする意味はない
let b = move || a() + 2;
let c = move || b() % 2 == 0;
let d = move || if c() { "偶数" } else { "奇数" };

set_a(2);
set_a(3);
set_a(5);
```

Memo化すれば、厳密には `a` を `3` に設定してから `5` に設定するまでの `d` の余分な計算を省けますが、これらの計算自体がリアクティブアルゴリズムより安価です。

検討するとしても、高コストな副作用を実行する直前の最後のnodeをMemo化する程度でしょう。

```rust
let text = Memo::new(move |_| {
    d()
});
Effect::new(move |_| {
    engrave_text_into_bar_of_gold(&text());
});
```
