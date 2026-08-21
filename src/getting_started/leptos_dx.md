# Leptosの開発者体験を改善する

Leptosを使ったWebサイトやアプリの開発体験を改善するために、いくつかできることがあります。特に、本書の例を実際に入力しながら読み進めたい場合は、数分かけて環境を設定し、開発しやすい状態にしておくとよいでしょう。

## 1）`console_error_panic_hook`を設定する

デフォルトでは、ブラウザでWASMコードを実行中にパニックが発生しても、`Unreachable executed`のような役に立たないメッセージと、WASMバイナリ内部を指すスタックトレースがブラウザにエラーとして表示されるだけです。

`console_error_panic_hook`を使うと、Rustソースコード内の行を含む、本来のRustスタックトレースを確認できます。

設定はとても簡単です。

1. プロジェクトで`cargo add console_error_panic_hook`を実行します。
2. main関数に`console_error_panic_hook::set_once();`を追加します。

> よくわからない場合は、[こちらの例](https://github.com/leptos-rs/leptos/blob/main/examples/counter/src/main.rs#L6)を参照してください。

これで、ブラウザコンソールに表示されるパニックメッセージが格段にわかりやすくなるはずです。

## 2）エディターで`#[component]`と`#[server]`の内部を自動補完する

マクロには、入力がその時点で完全に正しい場合に限り、あらゆるものからあらゆるものへ展開できるという性質があります。そのため、rust-analyzerが適切な自動補完やその他の支援機能を提供するのが難しいことがあります。

エディターでこれらのマクロを使う際に問題が起きた場合は、特定の手続きマクロを無視するようrust-analyzerに明示できます。特に`#[server]`マクロは、関数本体に注釈を付けるものの、本体内のコード自体は変換しないため、この設定が非常に役立ちます。

```admonish note 
 Leptos 0.5.3以降では、`#[component]`マクロがrust-analyzerでサポートされるようになりました。それでも問題が発生する場合は、`#[component]`もマクロの無視リストに追加するとよいでしょう（以下を参照してください）。
ただし、この設定を行うとrust-analyzerがコンポーネントのpropsを認識できなくなり、それが原因でIDEに別のエラーや警告が表示される可能性があります。
```

VS Codeの`settings.json`：

```json
"rust-analyzer.procMacro.ignored": {
	"leptos_macro": [
        // optional:
		// "component",
		"server"
	],
}
```

cargo-leptosを使用する場合のVS Codeの`settings.json`：
```json
"rust-analyzer.procMacro.ignored": {
	"leptos_macro": [
        // optional:
		// "component",
		"server"
	],
},
// if code that is cfg-gated for the `ssr` feature is shown as inactive,
// you may want to tell rust-analyzer to enable the `ssr` feature by default
//
// you can also use `rust-analyzer.cargo.allFeatures` to enable all features
"rust-analyzer.cargo.features": ["ssr"]
```

Neovim：

```lua
vim.lsp.config('rust_analyzer', {
  -- Other Configs ...
  settings = {
    ["rust-analyzer"] = {
      -- Other Settings ...
      procMacro = {
        ignored = {
          leptos_macro = {
            -- optional: --
            -- "component",
            "server",
          },
        },
      },
    },
  }
})
```

Helixの`.helix/languages.toml`：

```toml
[[language]]
name = "rust"

[language-server.rust-analyzer]
config = { procMacro = { ignored = { leptos_macro = [
	# Optional:
	# "component",
	"server"
] } } }
```

Zedの`settings.json`：

```json
{
  -- Other Settings ...
  "lsp": {
    "rust-analyzer": {
      "initialization_options": {
        "procMacro": {
          "ignored": {
            "leptos_macro": [
				// Optional:
				// "component",
				"server"
			],
          },
        },
      },
    },
  },
}
```

Sublime Text 3で、「Goto Anything...」メニューから`LSP-rust-analyzer.sublime-settings`を開いた場合：

```json
// Settings in here override those in "LSP-rust-analyzer/LSP-rust-analyzer.sublime-settings"
{
  "rust-analyzer.procMacro.ignored": {
    "leptos_macro": [
      // optional:
      // "component",
      "server"
    ],
  },
}
```
## 3）エディターのRust Analyzerでfeatureを有効にする（任意）
デフォルトでは、rust-analyzerはRustプロジェクトのデフォルトfeatureだけを対象に動作します。Leptosでは、複数のfeatureを使ってコンパイルを制御します。クライアントサイドレンダリングのプロジェクトでは各所で`csr`を使います。サーバーサイドレンダリングのアプリでは、サーバーコードに`ssr`、ブラウザだけで実行するコードに`hydrate`を含めることがあります。

これらのfeatureを有効にする方法はIDEによって異なります。以下に代表的なIDEでの設定を示します。使用しているIDEが一覧にない場合は、通常、`rust-analyzer.cargo.features`または`rust-analyzer.cargo.allFeatures`を検索すると設定を見つけられます。

VS Codeの`settings.json`：
```json
{
  "rust-analyzer.cargo.features": "all",  // Enable all features
}
```

Neovimの`init.lua`：
```lua
vim.lsp.config('rust_analyzer', {
  settings = {
    ["rust-analyzer"] = {
      cargo = {
        features = "all", -- Enable all features
      },
    },
  }
})

```
Helixの`.helix/languages.toml`、またはプロジェクトごとの`.helix/config.toml`：
```toml
[[language]]
name = "rust"

[language-server.rust-analyzer.config.cargo]
allFeatures = true
```

Zedの`settings.json`：

```json
{
  -- Other Settings ...
  "lsp": {
    "rust-analyzer": {
      "initialization_options": {
        "cargo": {
          "allFeatures": true // Enable all features
        }
      }
	}
  }
}
```

Sublime Text 3の`LSP-rust-analyzer-settings.json`にあるユーザー設定：
```json
 {
        "settings": {
            "LSP": {
                "rust-analyzer": {
                    "settings": {
                        "cargo": {
                            "features": "all"
                        }
                    }
                }
            }
        }
    }
```


## 4）`leptosfmt`を設定する（任意）

`leptosfmt`はLeptosの`view!`マクロ用フォーマッターです（通常、このマクロ内にUIコードを記述します）。`view!`マクロでは、JSXに似た「RSX」形式でUIを記述できるため、cargo-fmtはマクロ内部のコードをうまく自動整形できません。`leptosfmt`はこの整形上の問題を解決し、RSX形式のUIコードをきれいで読みやすい状態に保つクレートです。

`leptosfmt`は、コマンドラインまたはコードエディター内からインストールして使用できます。

最初に、`cargo install leptosfmt`でツールをインストールします。

コマンドラインからデフォルト設定で使用するだけなら、プロジェクトのルートで`leptosfmt ./**/*.rs`を実行します。これにより、すべてのRustファイルが`leptosfmt`で整形されます。

### Rust Analyzer対応IDEで自動実行する

エディターで`leptosfmt`を使えるように設定したい場合や、`leptosfmt`の動作をカスタマイズしたい場合は、[`leptosfmt`のGitHubリポジトリにあるREADME.md](https://github.com/bram209/leptosfmt)の手順を参照してください。

最良の結果を得るには、ワークスペースごとにエディターで`leptosfmt`を設定することが推奨されています。

### RustRoverで自動実行する

残念ながらRustRoverはRust Analyzerをサポートしていないため、`leptosfmt`を自動実行するには別の方法が必要です。ひとつの方法は、[FileWatchers](https://plugins.jetbrains.com/plugin/7177-file-watchers)プラグインを次の設定で使用することです。

- 名前：Leptosfmt
- ファイルの種類：Rust files
- プログラム：`/path/to/leptosfmt`（環境変数`$PATH`に含まれていれば、単に`leptosfmt`でも構いません）
- 引数：`$FilePath$`
- 更新する出力パス：`$FilePath$`


## 5）開発中に`--cfg=erase_components`を使う

Leptos 0.7では、型システムへの依存度を高める変更がレンダラーにいくつか加えられました。大規模なプロジェクトでは、これによってコンパイル時間が長くなることがあります。開発中にカスタム設定フラグ`--cfg=erase_components`を使うと、コンパイル時間の低下をほぼ解消できます（型情報の一部を消去することで、コンパイラーの処理量と出力するデバッグ情報を減らします。その代わり、バイナリサイズと実行時コストが増えるため、リリースモードでは使用しない方がよいでしょう）。

cargo-leptos v0.2.40以降では、開発モードで自動的に有効になります。Trunkを使用している場合、cargo-leptosを使用していない場合、または開発以外の用途でも有効にしたい場合は、コマンドライン（`RUSTFLAGS="--cfg erase_components" trunk serve`または`RUSTFLAGS="--cfg erase_components" cargo leptos watch`）か、`.cargo/config.toml`で簡単に設定できます。
```toml
# use your own native target
[target.aarch64-apple-darwin]
rustflags = [
  "--cfg",
  "erase_components",
]

[target.wasm32-unknown-unknown]
rustflags = [
   "--cfg",
   "erase_components",
]
```
