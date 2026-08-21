# 子要素を投影する

コンポーネントを構築していると、複数階層のコンポーネントを通して子要素を「投影」したくなる
場合があります。

## 問題

次のコードを考えてみましょう。

```rust
pub fn NestedShow<F, IV>(fallback: F, children: ChildrenFn) -> impl IntoView
where
    F: Fn() -> IV + Send + Sync + 'static,
    IV: IntoView + 'static,
{
    view! {
        <Show
            when=|| todo!()
            fallback=|| ()
        >
            <Show
                when=|| todo!()
                fallback=fallback
            >
                {children()}
            </Show>
        </Show>
    }
}
```

これは単純です。内側の条件が `true` なら `children` を表示し、そうでなければ `fallback` を
表示します。外側の条件が `false` なら `()`、つまり何もレンダリングしません。

つまり `<NestedShow/>` の子要素を外側の `<Show/>` コンポーネントを_通して_渡し、内側の
`<Show/>` の子要素にしたいのです。これを「投影」と呼んでいます。

このコードはコンパイルできません。

```
error[E0525]: expected a closure that implements the `Fn` trait, but this closure only implements `FnOnce`
```

各 `<Show/>` は `children` を複数回構築できなければなりません。外側の `<Show/>` の子要素を
初めて構築するとき、`fallback` と `children` は内側の `<Show/>` 呼び出しへmoveされます。
そのため、外側の `<Show/>` が後で子要素を構築するときには利用できません。

## 詳細

> 解決策まで読み飛ばしても構いません。

この問題を深く理解するには、展開後の `view` マクロを見ると役立ちます。整理したものを
次に示します。

```rust
Show(
    ShowProps::builder()
        .when(|| todo!())
        .fallback(|| ())
        .children({
            // ここでchildrenとfallbackがクロージャへmoveされる
            ::leptos::children::ToChildren::to_children(move || {
                Show(
                    ShowProps::builder()
                        .when(|| todo!())
                        // ここでfallbackが消費される
                        .fallback(fallback)
                        .children({
                            // ここでchildrenがキャプチャされる
                            ::leptos::children::ToChildren::to_children(
                                move || children(),
                            )
                        })
                        .build(),
                )
            })
        })
        .build(),
)
```

すべてのコンポーネントは自身のpropsを所有します。そのため、この場合の `<Show/>` は
`fallback` と `children` への参照しかキャプチャしておらず、呼び出すことができません。

## 解決策

しかし `<Suspense/>` と `<Show/>` はどちらも `ChildrenFn` を受け取ります。つまり、その
`children` は `Fn` 型を実装し、不変参照だけで複数回呼び出せる必要があります。したがって
`children` や `fallback` を所有する必要はなく、それらへの `'static` 参照を渡せれば十分です。

この問題は [`StoredValue`](https://docs.rs/leptos/latest/leptos/reactive/owner/struct.StoredValue.html)
プリミティブで解決できます。これはリアクティブシステムへ値を保存し、所有権をフレームワークへ
渡す代わりに、シグナルと同様に `Copy` かつ `'static` な参照を受け取るものです。特定の
メソッドを通じて値へアクセスしたり変更したりできます。

この場合は非常に単純です。

```rust
pub fn NestedShow<F, IV>(fallback: F, children: ChildrenFn) -> impl IntoView
where
    F: Fn() -> IV + Send + Sync + 'static,
    IV: IntoView + 'static,
{
    let fallback = StoredValue::new(fallback);
    let children = StoredValue::new(children);

    view! {
        <Show
            when=|| todo!()
            fallback=|| ()
        >
            <Show
                // Resourceから読み取り、利用者が検証済みか確認する
                when=move || todo!()
                fallback=move || fallback.read_value()()
            >
                {children.read_value()()}
            </Show>
        </Show>
    }
}
```

最上位で `fallback` と `children` の両方を、`NestedShow` が所有するリアクティブスコープへ
保存します。これで参照をほかの階層から `<Show/>` コンポーネントまでmoveし、そこで
呼び出せます。

## 最後の注意点

この方法が動作するのは、`<Show/>` が子要素の所有権ではなく、`.read_value` で取得できる
不変参照だけを必要とするためです。

別のケースでは、所有するpropsを `ChildrenFn` を取る関数へ投影し、その関数を複数回
呼び出す必要があるかもしれません。その場合は `view` マクロの `clone:` ヘルパーが役立ちます。

次の例を考えてみましょう。

```rust
#[component]
pub fn App() -> impl IntoView {
    let name = "Alice".to_string();
    view! {
        <Outer>
            <Inner>
                <Inmost name=name.clone()/>
            </Inner>
        </Outer>
    }
}

#[component]
pub fn Outer(children: ChildrenFn) -> impl IntoView {
    children()
}

#[component]
pub fn Inner(children: ChildrenFn) -> impl IntoView {
    children()
}

#[component]
pub fn Inmost(name: String) -> impl IntoView {
    view! {
        <p>{name}</p>
    }
}
```

`name=name.clone()` としても、次のエラーが発生します。

```
cannot move out of `name`, a captured variable in an `Fn` closure
```

複数回実行する必要がある子要素の複数階層を通じてキャプチャされており、子要素の_内側へ_
クローンする明確な方法がないためです。

この場合は `clone:` 構文が便利です。`clone:name` を指定すると、`name` を `<Inner/>` の
子要素へmoveする_前に_クローンし、所有権の問題を解決します。

```rust
view! {
	<Outer>
		<Inner clone:name>
			<Inmost name=name.clone()/>
		</Inner>
	</Outer>
}
```

`view` マクロが不透明なため、これらの問題は理解やデバッグが少し難しい場合があります。
しかし一般には、必ず解決できます。
