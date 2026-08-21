# WASMバイナリサイズの最適化

WebAssemblyバイナリは、同等のアプリケーションで想定されるJavaScript bundleよりかなり大きくなります。WASM形式はストリーミングコンパイル向けに設計されているため、WASMファイルは1キロバイトあたりではJavaScriptファイルよりはるかに速くコンパイルできます。（詳しくは、WASMのストリーミングコンパイルに関する[Mozillaチームの優れた記事](https://hacks.mozilla.org/2018/01/making-webassembly-even-faster-firefoxs-new-streaming-and-tiering-compiler/)を読めます。）それでも、可能な限り小さいWASMバイナリをユーザーへ配信することが重要です。ネットワーク使用量を減らし、アプリをできるだけ早く操作可能にできるためです。

では、実践的な手順には何があるでしょうか？

## 行うべきこと

1. リリースビルドを確認していることを確かめます。（デバッグビルドははるかに、はるかに大きくなります。）
2. 速度ではなくサイズを最適化するWASM用リリースprofileを追加します。

たとえば `cargo-leptos` プロジェクトなら、`Cargo.toml` に次を追加できます。

```toml
[profile.wasm-release]
inherits = "release"
opt-level = 'z'
lto = true
codegen-units = 1

# ....

[package.metadata.leptos]
# ....
lib-profile-release = "wasm-release"
```

これにより、サーバービルドは速度に最適化したまま、リリースビルドのWASMをサイズに対して徹底的に最適化します。（サーバーを考慮しない純粋なクライアントレンダリングのアプリなら、`[profile.wasm-release]` ブロックをそのまま `[profile.release]` として使ってください。）

3. 本番環境では必ず圧縮したWASMを配信します。WASMは非常によく圧縮でき、通常は非圧縮サイズの50%未満になります。ActixやAxumから配信する静的ファイルの圧縮は簡単に有効化できます。

4. nightly Rustを使っているなら、`wasm32-unknown-unknown` targetとともに配布されるビルド済み標準ライブラリではなく、同じprofileで標準ライブラリを再ビルドできます。

そのために、プロジェクトへ `.cargo/config.toml` ファイルを作成します。

```toml
[unstable]
build-std = ["std", "panic_abort", "core", "alloc"]
build-std-features = ["panic_immediate_abort"]
```

これをSSRでも使う場合、同じCargo profileが適用される点に注意してください。targetを明示的に指定する必要があります。
```toml
[build]
target = "x86_64-unknown-linux-gnu" # または任意のtarget
```

場合によってはcfg featureの `has_std` が設定されず、`has_std` を確認する一部の依存関係でビルドエラーが起きることにも注意してください。これが原因のビルドエラーは、次を追加すると修正できます。
```toml
[build]
rustflags = ["--cfg=has_std"]
```

さらに `Cargo.toml` の `[profile.release]` へ `panic = "abort"` を追加する必要があります。これにより、同じ `build-std` とpanicの設定がサーバーバイナリにも適用されますが、望ましくない場合もあります。ここはさらに調査が必要でしょう。

5. WASMバイナリのサイズを増やす原因の1つに、`serde` のシリアライズ／デシリアライズコードがあります。Leptosはデフォルトで `serde` を使い、`Resource::new()` で作ったResourceをシリアライズ・デシリアライズします。`leptos_server` には、追加の `new_` メソッドによって別のエンコーディングを有効にするfeatureがあります。たとえば `leptos_server` crateの `miniserde` featureを有効にすると `Resource::new_miniserde()` メソッドが追加され、`serde-lite` featureは `new_serde_lite` を追加します。`miniserde` と `serde-lite` が実装するのは `serde` の機能の一部だけですが、通常は速度よりバイナリサイズを優先して最適化されます。

## 避けるべきこと

バイナリサイズを膨らませやすいcrateがあります。たとえばデフォルトfeatureを有効にした `regex` crateは、WASMバイナリへ約500KBを追加します（主な理由はUnicodeテーブルのデータを取り込む必要があるためです！）。サイズを重視する場合は正規表現全般を避けるか、さらに低いレベルへ降り、ブラウザAPIを呼び出して組み込み正規表現エンジンを使うことも検討できます。（`leptos_router` も、正規表現が必要な数少ない場面でこの方法を使っています。）

一般に、Rustが実行時性能を重視することは、小さなバイナリを重視することと相反する場合があります。たとえばRustはジェネリック関数を単相化し、呼び出しに使われるジェネリック型ごとに異なる関数のコピーを作ります。これは動的dispatchより大幅に高速ですが、バイナリサイズを増やします。Leptosは実行時性能とバイナリサイズをかなり慎重に両立させようとしていますが、多数のgenericを使うコードはバイナリサイズを増やす傾向があると気づくかもしれません。たとえば本体に多くのコードを持つジェネリックコンポーネントを4つの異なる型で呼び出すと、コンパイラーが同じコードを4つ含める可能性があります。具体型を使う内部関数やヘルパーへリファクタリングすると、性能と使いやすさを保ちながらバイナリサイズを減らせることがよくあります。

## コード分割

`cargo-leptos`、Leptosフレームワーク、routerはWASMバイナリの分割に対応しています。（この対応は2025年夏にリリースされたため、これを読む時期によっては、まだバグを修正している最中かもしれません。）

これは3つの道具の組み合わせで利用できます。`cargo leptos (serve|watch|build) --split`、[`#[lazy]`](https://docs.rs/leptos/latest/leptos/attr.lazy.html) マクロ、そして [`LazyRoute`](https://docs.rs/leptos_router/latest/leptos_router/trait.LazyRoute.html) traitと組み合わせる [`#[lazy_route]`](https://docs.rs/leptos_router/latest/leptos_router/attr.lazy_route.html) マクロです。

### `#[lazy]`

`#[lazy]` マクロは、関数を独立したWebAssembly（WASM）バイナリから遅延読み込みできることを示します。同期関数にも非同期関数にも付けられ、どちらの場合も非同期関数を生成します。遅延読み込みされる関数を初めて呼ぶと、独立したコード断片がサーバーから読み込まれて呼び出されます。それ以降は、追加の読み込み手順なしで呼び出されます。

```rust
#[lazy]
fn lazy_synchronous_function() -> String {
    "Hello, lazy world!".to_string()
}

#[lazy]
async fn lazy_async_function() -> String {
    /* 非同期処理を必要とする何かを行う */
    "こんにちは、遅延非同期の世界！".to_string()
}

async fn use_lazy_functions() {
    // 同期関数が非同期へ変換されている
    let value1 = lazy_synchronous_function().await;

    // 非同期関数は非同期のまま
    let value1 = lazy_async_function().await;
}
```

これは単発の遅延関数に役立ちます。しかし、遅延読み込みはrouterと組み合わせたときに最も力を発揮します。

### `#[lazy_route]`

遅延ルートを使うと、ルートのviewのコードを分割し、ナビゲーション中にそのルートのデータと並行して遅延読み込みできます。ネストしたroutingにより、遅延読み込みされる複数のルートもネストできます。それぞれが独自のデータと遅延viewを並行して読み込みます。

データ読み込みを（遅延読み込みされる）viewから分離すると、遅延viewの読み込みを待ってからデータ読み込みを始める「ウォーターフォール」を防げます。

```rust
use leptos::prelude::*;
use leptos_router::{lazy_route, LazyRoute};

// ルートの定義
#[derive(Debug)]
struct BlogListingRoute {
    titles: Resource<Vec<String>>
}

#[lazy_route]
impl LazyRoute for BlogListingRoute {
    fn data() -> Self {
        Self {
            titles: Resource::new(|| (), |_| async {
                vec![/* TODO：ブログ記事を読み込む */]
            })
        }
    }

    // この関数はdata()と並行して遅延読み込みされる
    fn view(this: Self) -> AnyView {
        let BlogListingRoute { titles } = this;

        // ……ここで`posts` ResourceをSuspenseなどとともに使い、
        // viewに.into_any()を呼んでAnyViewを返せる
    }
}
```

### 例と詳細情報

詳しい解説は[こちらのYouTube動画](https://www.youtube.com/watch?v=w5fhcoxQnII)で、完全な [`lazy_routes`](https://github.com/leptos-rs/leptos/blob/main/examples/lazy_routes/src/app.rs) の例はリポジトリで確認できます。
