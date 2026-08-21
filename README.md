# Leptos Book 日本語版

- [公開版](#公開版)
- [翻訳元](#翻訳元)
- [ローカルでのビルド](#ローカルでのビルド)
- [GitHub Pagesへの公開](#github-pagesへの公開)
- [VS Code Dev Container（任意）](#vs-code-dev-container任意)

## 公開版

[Leptos Book 日本語版](https://tonodera.github.io/leptos-book-ja/)をGitHub Pagesで公開します。

このリポジトリは、[The Leptos Book](https://book.leptos.dev/)の非公式日本語訳です。原著の章構成とコード例を維持し、説明文を自然な日本語に翻訳しています。翻訳内容と原著に相違がある場合は、原著を正としてください。

## 翻訳元

- 原著：[leptos-rs/book](https://github.com/leptos-rs/book)
- 翻訳基準コミット：`3d65eecabfb0a4bc93d41bb803b67103828f051a`
- ライセンス：MIT License

## ローカルでのビルド

本書は [`mdbook`](https://crates.io/crates/mdbook) で構築します。Cargoを使って`mdbook`をインストールしてください。

```sh
cargo install mdbook --version 0.4.*
```

注記や警告などの表示には、mdBook preprocessorの [`mdbook-admonish`](https://crates.io/crates/mdbook-admonish) も使用します。

```sh
cargo install mdbook-admonish --version 1.*
```


インストール後、次のコマンドで本書を起動します。

```sh
mdbook serve
```

ブラウザで [`http://localhost:3000`](http://localhost:3000) を開いてください。生成だけを行う場合は`mdbook build`を実行します。成果物は`book/`へ出力されます。

## GitHub Pagesへの公開

`main`ブランチへ変更が反映されると、`.github/workflows/publish_mdbook.yml`が本書をビルドし、GitHub Pagesへ自動的にデプロイします。初回のみ、GitHubリポジトリの「Settings」→「Pages」で「Source」を「GitHub Actions」に設定してください。

## VS Code Dev Container（任意）

付属の [VS Code Dev Container](https://code.visualstudio.com/docs/devcontainers/containers)でもビルド・実行できます。依存関係のインストール、本書のビルド、ライブリロード付きの配信が自動的に行われます。

Dockerと公式の [Dev Containers](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-containers) 拡張機能をインストールし、VS Codeでプロジェクトを開いて「Reopen in Container」を選択してください。

詳しくは[VS Codeのドキュメント](https://code.visualstudio.com/docs/devcontainers/containers)を参照してください。
