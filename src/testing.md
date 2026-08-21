# コンポーネントをテストする

ユーザーインターフェイスのテストは比較的難しいものの、非常に重要です。この章では、
Leptosアプリケーションをテストするための原則と方法をいくつか説明します。

## 1. 通常のRustテストでビジネスロジックをテストする

多くの場合、ロジックをコンポーネントから取り出し、個別にテストするのが合理的です。
単純なコンポーネントにはテストすべきロジックがないこともありますが、多くのケースでは
テスト可能なラッパー型を使い、通常のRustの `impl` ブロックへロジックを実装する価値が
あります。

たとえば、次のようにロジックをコンポーネントへ直接埋め込む代わりに、

```rust
#[component]
pub fn TodoApp() -> impl IntoView {
    let (todos, set_todos) = signal(vec![Todo { /* ... */ }]);
    // ⚠️ コンポーネントへ埋め込まれているためテストしにくい
    let num_remaining = move || todos.read().iter().filter(|todo| !todo.completed).sum();
}
```

ロジックを別のデータ構造へ取り出してテストできます。

```rust
pub struct Todos(Vec<Todo>);

impl Todos {
    pub fn num_remaining(&self) -> usize {
        self.0.iter().filter(|todo| !todo.completed).sum()
    }
}

#[cfg(test)]
mod tests {
    #[test]
    fn test_remaining() {
        // ...
    }
}

#[component]
pub fn TodoApp() -> impl IntoView {
    let (todos, set_todos) = signal(Todos(vec![Todo { /* ... */ }]));
    // ✅ 対応するテストが存在する
    let num_remaining = move || todos.read().num_remaining();
}
```

一般に、コンポーネント自体へ包み込むロジックが少ないほど、コードはよりRustらしくなり、
テストもしやすくなります。

## 2. エンドツーエンド（`e2e`）テストでコンポーネントをテストする

Leptosの [`examples`](https://github.com/leptos-rs/leptos/tree/main/examples) ディレクトリには、
さまざまなテストツールを使った充実したエンドツーエンドテストの例がいくつかあります。

使い方を理解する最も簡単な方法は、テスト例そのものを見ることです。

### [`counter`](https://github.com/leptos-rs/leptos/blob/main/examples/counter/tests/web.rs)で`wasm-bindgen-test`を使う

これは [`wasm-pack test`](https://rustwasm.github.io/wasm-pack/book/commands/test.html) コマンドを
使う、比較的単純な手動テストのセットアップです。

#### テスト例

```rust
#[wasm_bindgen_test]
async fn clear() {
    let document = document();
    let test_wrapper = document.create_element("section").unwrap();
    let _ = document.body().unwrap().append_child(&test_wrapper);

    // 最初にカウンターをレンダリングしてDOMへマウントする
    // 初期値10から始めることに注目
    let _dispose = mount_to(
        test_wrapper.clone().unchecked_into(),
        || view! { <SimpleCounter initial_value=10 step=1/> },
    );

    // DOMをたどってボタンを取り出す
    // IDがあれば、さらに簡単になる
    let div = test_wrapper.query_selector("div").unwrap().unwrap();
    let clear = test_wrapper
        .query_selector("button")
        .unwrap()
        .unwrap()
        .unchecked_into::<web_sys::HtmlElement>();

    // `clear`ボタンをクリックする
    clear.click();

    // リアクティブシステムは非同期システム上に構築されているため、
    // 変更はDOMへ同期的には反映されない
    // 変更を検出するため、変更のたびに短時間yieldし、ビューを更新するエフェクトを実行させる
    tick().await;

    // <div>が期待値と一致するか、`outerHTML`を使ってテストする
    assert_eq!(div.outer_html(), {
        // 値0で作成するのと同じ状態になる
        let (value, _set_value) = signal(0);

        // イベントリスナーはHTMLへレンダリングされないため削除できる
        view! {
            <div>
                <button>"Clear"</button>
                <button>"-1"</button>
                <span>"Value: " {value} "!"</span>
                <button>"+1"</button>
            </div>
        }
        // LeptosはHTML要素に対する複数のバックエンドレンダラーをサポートする
        // ここでの.into_view()は「通常のDOMレンダラーを使う」と指定する便利な方法
        .into_view()
        // ビューは遅延され、DOMツリーを記述するが、この時点ではまだ作成しない
        // .build()を呼び出すと実際にDOM要素を構築する
        .build()
        // .build()はDOM要素のスマートポインタであるElementStateを返す
        // そのため.outer_html()を呼び出し、実際のDOM要素のouterHTMLへアクセスできる
        .outer_html()
    });

    // 実はもっと簡単な方法がある……
    // 初期値0の<SimpleCounter/>と比較するだけでよい
    assert_eq!(test_wrapper.inner_html(), {
        let comparison_wrapper = document.create_element("section").unwrap();
        let _dispose = mount_to(
            comparison_wrapper.clone().unchecked_into(),
            || view! { <SimpleCounter initial_value=0 step=1/>},
        );
        comparison_wrapper.inner_html()
    });
}
```

### [`counters`でPlaywrightを使う](https://github.com/leptos-rs/leptos/tree/main/examples/counters/e2e)

このテストでは一般的なJavaScriptテストツールPlaywrightを使い、同じ例に対してエンドツー
エンドテストを実行します。フロントエンド開発経験者の多くに馴染みのあるライブラリと
テスト手法です。

#### テスト例

```js
test.describe("Increment Count", () => {
  test("should increase the total count", async ({ page }) => {
    const ui = new CountersPage(page);
    await ui.goto();
    await ui.addCounter();

    await ui.incrementCount();
    await ui.incrementCount();
    await ui.incrementCount();

    await expect(ui.total).toHaveText("3");
  });
});
```

### [`todo_app_sqlite`でのGherkin/Cucumberテスト](https://github.com/leptos-rs/leptos/blob/main/examples/todo_app_sqlite/e2e/README.md)

この流れには任意のテストツールを統合できます。次の例では、自然言語に基づくテスト
フレームワークCucumberを使います。

```
@add_todo
Feature: Add Todo

    Background:
        Given I see the app

    @add_todo-see
    Scenario: Should see the todo
        Given I set the todo as Buy Bread
        When I click the Add button
        Then I see the todo named Buy Bread

    # @allow.skipped
    @add_todo-style
    Scenario: Should see the pending todo
        When I add a todo as Buy Oranges
        Then I see the pending todo
```

これらのアクションの定義はRustコードで記述されています。

```rust
use crate::fixtures::{action, world::AppWorld};
use anyhow::{Ok, Result};
use cucumber::{given, when};

#[given("I see the app")]
#[when("I open the app")]
async fn i_open_the_app(world: &mut AppWorld) -> Result<()> {
    let client = &world.client;
    action::goto_path(client, "").await?;

    Ok(())
}

#[given(regex = "^I add a todo as (.*)$")]
#[when(regex = "^I add a todo as (.*)$")]
async fn i_add_a_todo_titled(world: &mut AppWorld, text: String) -> Result<()> {
    let client = &world.client;
    action::add_todo(client, text.as_str()).await?;

    Ok(())
}

// 以下省略
```

### さらに学ぶ

自分のアプリケーションでこれらのツールを使う方法について詳しくは、Leptosリポジトリの
CI設定も自由に参照してください。これらのテスト手法はすべて、実際のLeptosサンプル
アプリケーションに対して定期的に実行されています。
