# `<For/>`でより複雑なデータを反復処理する

この章では、入れ子になったデータ構造の反復処理をもう少し詳しく掘り下げます。反復処理を扱う前章と関連する内容ですが、いまはもっと単純なテーマに集中したければ、読み飛ばして後から戻ってきても構いません。

## 問題

キーが変化しない限り、フレームワークは行内の項目を再レンダリングしないと説明しました。最初は筋が通っているように思えますが、思わぬ落とし穴になることがあります。

行の各項目が何らかのデータ構造になっている例を考えましょう。たとえば、キーと値を持つJSON配列から項目が取得されるとします。

```rust
#[derive(Debug, Clone)]
struct DatabaseEntry {
    key: String,
    value: i32,
}
```

行を反復処理して各行を表示する単純なコンポーネントを定義します。

```rust
#[component]
pub fn App() -> impl IntoView {
    // 3行のデータから始める
    let (data, set_data) = signal(vec![
        DatabaseEntry {
            key: "foo".to_string(),
            value: 10,
        },
        DatabaseEntry {
            key: "bar".to_string(),
            value: 20,
        },
        DatabaseEntry {
            key: "baz".to_string(),
            value: 15,
        },
    ]);
    view! {
        // クリックしたら各行の値を2倍にする
        <button on:click=move |_| {
            set_data.update(|data| {
                for row in data {
                    row.value *= 2;
                }
            });
            // シグナルの新しい値をログへ出力する
            leptos::logging::log!("{:?}", data.get());
        }>
            "Update Values"
        </button>
        // 行を反復処理して各値を表示する
        <For
            each=move || data.get()
            key=|state| state.key.clone()
            let(child)
        >
            <p>{child.value}</p>
        </For>
    }
}
```

> ここで使っている`let(child)`構文に注目してください。前章では`children` propsを指定した`<For/>`を紹介しました。実際には`view`マクロから抜けることなく、`<For/>`コンポーネントの子としてこの値を直接作成できます。上の`let(child)`と`<p>{child.value}</p>`の組み合わせは、次のコードと同じです。
>
> ```rust
> children=|child| view! { <p>{child.value}</p> }
> ```
>
> 必要に応じて、データのパターンを分割代入することもできます。
>
> ```rust
> <For
>     each=move || data.get()
>     key=|state| state.key.clone()
>     let(DatabaseEntry { key, value })
> >
> ```

`Update Values`ボタンをクリックしても……何も起きません。正確には、シグナルは更新され、新しい値もログへ出力されますが、各行の`{child.value}`は更新されません。

リアクティブにするためのクロージャを追加し忘れたのでしょうか。`{move || child.value}`を試してみましょう。

……いいえ。やはり何も起きません。

問題は、各行がキーの変化時にしか再レンダリングされないことです。各行の値は更新しましたが、どの行のキーも更新していないため、何も再レンダリングされません。また、`child.value`の型はリアクティブな`ReadSignal<i32>`などではなく、通常の`i32`です。つまり、クロージャで包んでも、この行の値は決して更新されません。

考えられる解決策は4つあります。

1. データ構造が変化したときに必ず更新されるよう`key`を変更する
2. リアクティブになるよう`value`を変更する
3. 各行を直接使わず、データ構造のリアクティブなスライスを取得する
4. `Store`を使う

## 選択肢1：キーを変更する

各行が再レンダリングされるのはキーが変化したときだけです。先ほどの行はキーが変化しなかったため再レンダリングされませんでした。それなら、キーを強制的に変化させてみましょう。

```rust
<For
	each=move || data.get()
	key=|state| (state.key.clone(), state.value)
	let(child)
>
	<p>{child.value}</p>
</For>
```

`key`へキーと値の両方を含めました。これにより、行の値が変化するたびに`<For/>`は完全に新しい行として扱い、以前の行を置き換えます。

### 長所

とても簡単な方法です。`DatabaseEntry`で`PartialEq`、`Eq`、`Hash`をderiveすれば、さらに単純に`key=|state| state.clone()`と記述できます。

### 短所

**4つの選択肢のなかで最も効率が悪い方法です。** 行の値が変化するたびに以前の`<p>`要素を破棄し、まったく新しい要素へ置き換えます。テキストノードをきめ細かく更新するのではなく、変化のたびに行全体を実際に再レンダリングするため、行のUIが複雑になるほどコストも高くなります。

また、`<For/>`がキーのコピーを保持できるよう、データ構造全体をクローンすることになります。複雑な構造では、すぐに望ましくない方法となります。

## 選択肢2：入れ子のシグナル

値に対してきめ細かなリアクティビティが必要なら、各行の`value`をシグナルで包む方法があります。

```rust
#[derive(Debug, Clone)]
struct DatabaseEntry {
    key: String,
    value: RwSignal<i32>,
}
```

`RwSignal<_>`はgetterとsetterをひとつのオブジェクトへまとめた「読み書き可能なシグナル」です。getterとsetterを別々に格納するより構造体へ保存しやすいため、ここではこれを使います。

```rust
#[component]
pub fn App() -> impl IntoView {
    // 3行のデータから始める
    let (data, _set_data) = signal(vec![
        DatabaseEntry {
            key: "foo".to_string(),
            value: RwSignal::new(10),
        },
        DatabaseEntry {
            key: "bar".to_string(),
            value: RwSignal::new(20),
        },
        DatabaseEntry {
            key: "baz".to_string(),
            value: RwSignal::new(15),
        },
    ]);
    view! {
        // クリックしたら各行の値を2倍にする
        <button on:click=move |_| {
            for row in &*data.read() {
                row.value.update(|value| *value *= 2);
            }
            // シグナルの新しい値をログへ出力する
            leptos::logging::log!("{:?}", data.get());
        }>
            "Update Values"
        </button>
        // 行を反復処理して各値を表示する
        <For
            each=move || data.get()
            key=|state| state.key.clone()
            let(child)
        >
            <p>{child.value}</p>
        </For>
    }
}
```

このバージョンは正しく動作します！ ブラウザのDOMインスペクターで確認すると、
以前のバージョンとは異なり、個々のテキストノードだけが更新されていることがわかります。
シグナルはビューに渡してもリアクティビティを維持するため、シグナルをそのまま
`{child.value}` に渡すことで正しく動作します。

`set_data.update()` を `data.read()` に変更した点にも注目してください。`.read()` は、
シグナルの値をクローンせずに参照する方法です。ここで更新しているのは内側の値だけで、
値のリスト自体ではありません。各シグナルがそれぞれの状態を保持しているので、実際には
`data` シグナルを更新する必要がなく、不変の参照を返す `.read()` で十分です。

> 実際、このバージョンは `data` を更新しないため、`<For/>` は前章で扱った静的なリストと
> 本質的に同じであり、通常のイテレータでも構いません。ただし、将来行を追加または削除する
> 場合には `<For/>` が役立ちます。

### 長所

これは最も効率的な選択肢であり、フレームワーク全体の考え方にも素直に沿っています。
時間とともに変化する値をシグナルで包むことで、インターフェイスがその変化に応答できます。

### 短所

APIなど、自分では制御できないデータソースからデータを受け取る場合、ネストした
リアクティビティは扱いにくいことがあります。各フィールドをシグナルで包むためだけに、
別の構造体を作りたくない場合もあるでしょう。

## 選択肢3：メモ化したスライス

Leptosには [`Memo`](https://docs.rs/leptos/latest/leptos/reactive/computed/struct.Memo.html)
というプリミティブがあります。これは派生計算を作成し、その値が変化した場合にだけ
リアクティブな更新を引き起こします。

これを使うと、大きなデータ構造のフィールドをシグナルで包まなくても、サブフィールドに
対するリアクティブな値を作成できます。
[`<ForEnumerate/>`](https://docs.rs/leptos/latest/leptos/control_flow/fn.ForEnumerate.html)
と組み合わせれば、変更されたデータ値だけを再レンダリングできます。

アプリケーションの大部分は最初の（動作しなかった）バージョンのままにして、`<For/>`
の部分だけを次のように変更します。

```rust
<ForEnumerate
    each=move || data.get()
    key=|state| state.key.clone()
    children=move |index, _| {
        let value = Memo::new(move |_| {
            data.with(|data| data.get(index.get()).map(|d| d.value).unwrap_or(0))
        });
        view! {
            <p>{value}</p>
        }
    }
/>
```

ここにはいくつかの違いがあります。

- `For` ではなく `ForEnumerate` を使うため、`index` シグナルを利用できます。
- `view` 以外のコードを実行しやすくするため、`children` プロパティを明示的に使っています。
- `value` メモを定義し、ビュー内で使っています。この `value` は、各行へ渡される `child` を
  実際には使用しません。代わりにインデックスを使って元の `data` を参照し、値を取得します。

これで `data` が変化するたびに各メモが再計算されます。その値が変化していれば、行全体を
再レンダリングすることなく、対応するテキストノードだけを更新します。

**注意**：以前のバージョンの例のように、列挙したイテレータと `For` を組み合わせて
この処理を行うのは安全ではありません。

```rust
<For
    each=move || data.get().into_iter().enumerate()
    key=|(_, state)| state.key.clone()
    children=move |(index, _)| {
        let value = Memo::new(move |_| {
            data.with(|data| data.get(index).map(|d| d.value).unwrap_or(0))
        });
        view! {
            <p>{value}</p>
        }
    }
/>
```

この場合、`data` 内の値の変化には反応しますが、順序の変化には反応しません。Memoは常に、
作成時の `index` を使い続けるためです。項目を移動すると、レンダリング結果に重複した項目が
生じます。

### 長所

データ自体をシグナルで包まなくても、シグナルで包むバージョンと同じきめ細かな
リアクティビティを得られます。

### 短所

ネストしたシグナルを使う場合と比べ、`<ForEnumerate/>` ループ内で行ごとにメモを設定する
処理は少し複雑です。たとえば、`data[index.get()]` ではなく `data.get(index.get())` を使い、
パニックの可能性を防ぐ必要があります。行が削除された直後に、このメモが一度だけ再実行
される場合があるためです（各行のメモと `<ForEnumerate/>` 全体が同じ `data` シグナルに
依存しており、同一のシグナルに依存する複数のリアクティブな値について、実行順序は
保証されていません）。

また、メモはリアクティブな変化をメモ化しますが、値を確認するための同じ計算は毎回
再実行する必要があります。そのため、このように特定の箇所だけを更新する場合は、
ネストしたリアクティブなシグナルの方が依然として効率的です。

## 選択肢4：ストア

> この内容の一部は、[こちら](../15_global_state.md#option-3-create-a-global-state-store)の
> ストアを使ったグローバル状態管理の節と重複しています。どちらも中級者向けの任意の内容
> なので、多少重複しても問題ないと考えました。

Leptos 0.7では、「ストア」という新しいリアクティブプリミティブが導入されました。
ストアは、この章でここまで説明してきた問題に対処するために設計されています。やや実験的な
機能であるため、`Cargo.toml` に `reactive_stores` という依存関係を追加する必要があります。

ストアを使うと、上記の選択肢のようにネストしたシグナルやメモを手作業で作らなくても、
構造体の個々のフィールドや `Vec<_>` などのコレクション内の各項目へ、きめ細かく
リアクティブにアクセスできます。

ストアは `Store` deriveマクロを基盤としており、このマクロが構造体の各フィールドに対する
ゲッターを作成します。ゲッターを呼び出すと、その特定のフィールドへリアクティブに
アクセスできます。値を読み取ると、そのフィールドと親・子だけが追跡されます。更新時にも
そのフィールドと親・子だけへ通知され、兄弟フィールドには通知されません。つまり、`value`
を変更しても `key` には通知されない、ということです。

上の例で使ったデータ型をストア向けに変更してみましょう。

ストアの最上位は必ず構造体である必要があるため、`rows` フィールドだけを持つ `Data`
ラッパーを作成します。

```rust
#[derive(Store, Debug, Clone)]
pub struct Data {
    #[store(key: String = |row| row.key.clone())]
    rows: Vec<DatabaseEntry>,
}

#[derive(Store, Debug, Clone)]
struct DatabaseEntry {
    key: String,
    value: i32,
}
```

`rows` フィールドに `#[store(key)]` を追加すると、ストアのフィールドへキー付きで
アクセスできるようになります。これは、以下の `<For/>` コンポーネントで役立ちます。
`<For/>` で使うものと同じ `key` を、そのまま使用できます。

`<For/>` コンポーネントはとても単純です。

```rust
<For
    each=move || data.rows()
    key=|row| row.read().key.clone()
    children=|child| {
        let value = child.value();
        view! { <p>{move || value.get()}</p> }
    }
/>
```

`rows` はキー付きフィールドなので `IntoIterator` を実装しており、`each` プロパティには
単純に `move || data.rows()` を使えます。ネストしたシグナルのバージョンで
`move || data.get()` が行っていたのと同様に、`rows` リストへのあらゆる変更に反応します。

`key` フィールドでは `.read()` を呼び出して行の現在値へアクセスし、`key` フィールドを
クローンして返します。

`children` プロパティ内で `child.value()` を呼び出すと、このキーを持つ行の `value`
フィールドへリアクティブにアクセスできます。行の並べ替え、追加、削除が行われても、
キー付きストアフィールドが同期を保つため、この `value` は常に正しいキーと対応します。

更新ボタンのハンドラーでは、`rows` の項目を反復処理し、それぞれを更新します。

```rust
for row in data.rows().iter_unkeyed() {
    *row.value().write() *= 2;
}
```

### 長所

ネストしたシグナルやメモ化したスライスを手作業で作成しなくても、ネストしたシグナル版や
メモ版と同じ、きめ細かなリアクティビティを得られます。特殊なネストしたリアクティブ型ではなく、
deriveマクロを付けた通常のデータ（構造体と `Vec<_>`）を扱えます。

個人的には、ここで紹介した中ではストア版が最も優れていると思います。最新のAPIなので、
それも当然かもしれません。私たちは数年にわたってこれらの問題を考えてきましたが、ストアには
そこで得た知見の一部が取り入れられています！

### 短所

一方で、これは最も新しいAPIでもあります。この文を書いている時点（2024年12月）で、
ストアがリリースされてからまだ数週間しか経っていません。未解決のバグやエッジケースが
まだ残っているはずです。

### 完全な例

以下がストアを使った完全な例です。さらに充実した別の例は
[こちら](https://github.com/leptos-rs/leptos/blob/main/examples/stores/src/lib.rs)、
本書での詳しい説明は[こちら](../15_global_state.md)にあります。

```
use reactive_stores::Store;

#[derive(Store, Debug, Clone)]
pub struct Data {
    #[store(key: String = |row| row.key.clone())]
    rows: Vec<DatabaseEntry>,
}

#[derive(Store, Debug, Clone)]
struct DatabaseEntry {
    key: String,
    value: i32,
}

#[component]
pub fn App() -> impl IntoView {
    // 行を保持するシグナルの代わりに、Dataのストアを作成する
    let data = Store::new(Data {
        rows: vec![
            DatabaseEntry {
                key: "foo".to_string(),
                value: 10,
            },
            DatabaseEntry {
                key: "bar".to_string(),
                value: 20,
            },
            DatabaseEntry {
                key: "baz".to_string(),
                value: 15,
            },
        ],
    });

    view! {
        // クリックしたら各行を更新し、値を2倍にする
        <button on:click=move |_| {
            // 反復可能なストアフィールドの項目を反復処理できるようにする
            use reactive_stores::StoreFieldIterator;

            // rows()を呼び出して行へアクセスする
            for row in data.rows().iter_unkeyed() {
                *row.value().write() *= 2;
            }
            // ストアの新しい値をログへ出力する
            leptos::logging::log!("{:?}", data.get());
        }>
            "Update Values"
        </button>
        // 行を反復処理して各値を表示する
        <For
            each=move || data.rows()
            key=|row| row.read().key.clone()
            children=|child| {
                let value = child.value();
                view! { <p>{move || value.get()}</p> }
            }
        />
    }
}
```
