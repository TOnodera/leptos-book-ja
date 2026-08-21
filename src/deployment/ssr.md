# フルスタックSSRアプリのデプロイ

LeptosのフルスタックSSRアプリは、さまざまなサーバーまたはコンテナホスティングサービスへデプロイできます。本番稼働させる最も簡単な方法は、VPSサービスを使い、VM内でLeptosをネイティブ実行することかもしれません（[詳しくはこちら](https://github.com/leptos-rs/start-axum?tab=readme-ov-file#executing-a-server-on-a-remote-machine-without-the-toolchain)）。別の方法として、Leptosアプリをコンテナ化し、任意のコロケーションサーバーやクラウドサーバー上で [Podman](https://podman.io/) または [Docker](https://www.docker.com/) を使って実行できます。

デプロイ構成とホスティングサービスには非常に多くの種類があり、一般にLeptos自体は使用するデプロイ構成に依存しません。デプロイ先の多様性を踏まえ、このページでは次を扱います。

- Leptos SSRアプリで使う [`Containerfile`（または `Dockerfile`）の作成](#containerfileを作成する)
- `Dockerfile` を使った[クラウドサービスへのデプロイ](#クラウドへのデプロイ)（[Fly.io](#flyioへデプロイする)など）
- Leptosの[サーバーレスruntimeへのデプロイ](#サーバーレスruntimeへデプロイする)（[AWS Lambda](#aws-lambda)や、[DenoとCloudflareのようなJSホスト型WASM runtime](#denoとcloudflare-workers)など）
- [まだLeptos SSRに対応していないプラットフォーム](#leptos対応に取り組んでいるプラットフォーム)

_注意：Leptosは特定のデプロイ方法やホスティングサービスの利用を推奨していません。_

## Containerfileを作成する

`cargo-leptos` で構築したフルスタックアプリのデプロイでは、PodmanまたはDockerビルドによるデプロイに対応するクラウドホスティングサービスを使う方法が最も一般的です。次はLeptosのWebサイトのデプロイに使っているものを基にした `Containerfile`／`Dockerfile` の例です。

### Debian

```dockerfile
# Get started with a build env with Rust nightly
FROM rustlang/rust:nightly-trixie as builder

# If you’re using stable, use this instead
# FROM rust:1.92.0-trixie as builder # See current official Rust tags here: https://hub.docker.com/_/rust

# Install cargo-binstall, which makes it easier to install other
# cargo extensions like cargo-leptos
RUN wget https://github.com/cargo-bins/cargo-binstall/releases/latest/download/cargo-binstall-x86_64-unknown-linux-musl.tgz
RUN tar -xvf cargo-binstall-x86_64-unknown-linux-musl.tgz
RUN cp cargo-binstall /usr/local/cargo/bin

# Install required tools
RUN apt-get update -y \
  && apt-get install -y --no-install-recommends clang

# Install cargo-leptos
RUN cargo binstall cargo-leptos -y

# Add the WASM target
RUN rustup target add wasm32-unknown-unknown

# Make an /app dir, which everything will eventually live in
RUN mkdir -p /app
WORKDIR /app
COPY . .

# Build the app
RUN cargo leptos build --release -vv

FROM debian:trixie-slim as runtime
WORKDIR /app
RUN apt-get update -y \
  && apt-get install -y --no-install-recommends openssl ca-certificates \
  && apt-get autoremove -y \
  && apt-get clean -y \
  && rm -rf /var/lib/apt/lists/*

# -- NB: update binary name from "leptos_start" to match your app name in Cargo.toml --
# Copy the server binary to the /app directory
COPY --from=builder /app/target/release/leptos_start /app/

# /target/site contains our JS/WASM/CSS, etc.
COPY --from=builder /app/target/site /app/site

# Copy Cargo.toml if it’s needed at runtime
COPY --from=builder /app/Cargo.toml /app/

# Set any required env variables and
ENV RUST_LOG="info"
ENV LEPTOS_SITE_ADDR="0.0.0.0:8080"
ENV LEPTOS_SITE_ROOT="site"
EXPOSE 8080

# -- NB: update binary name from "leptos_start" to match your app name in Cargo.toml --
# Run the server
CMD ["/app/leptos_start"]
```

### Alpine

```dockerfile
# Get started with a build env with Rust nightly
FROM rustlang/rust:nightly-alpine as builder

RUN apk update && \
    apk add --no-cache bash curl npm libc-dev binaryen

RUN npm install -g sass

RUN curl --proto '=https' --tlsv1.3 -LsSf https://github.com/leptos-rs/cargo-leptos/releases/latest/download/cargo-leptos-installer.sh | sh

# Add the WASM target
RUN rustup target add wasm32-unknown-unknown

WORKDIR /work
COPY . .

RUN cargo leptos build --release -vv

FROM rustlang/rust:nightly-alpine as runner

WORKDIR /app

COPY --from=builder /work/target/release/leptos_start /app/
COPY --from=builder /work/target/site /app/site
COPY --from=builder /work/Cargo.toml /app/

ENV RUST_LOG="info"
ENV LEPTOS_SITE_ADDR="0.0.0.0:8080"
ENV LEPTOS_SITE_ROOT=./site
EXPOSE 8080

CMD ["/app/leptos_start"]
```

> 詳細： [Leptosアプリ用の `gnu` と `musl` のビルドファイル](https://github.com/leptos-rs/leptos/issues/1152#issuecomment-1634916088)。

## リバースプロキシに関する注意

Leptosアプリを直接公開することもできますが、通常はリバースプロキシの背後へ置くほうが適切です。これにより、SSL／TLS、圧縮、セキュリティヘッダーをRustバイナリではなく専用の層で処理できます。

人気のあるリバースプロキシはいくつかあります。CaddyはHTTPS証明書の自動管理によってよく選ばれ、要件や習熟度に応じてNginx、Traefik、Apacheも広く使われています。

Caddyを使う場合、ドメインをコンテナ名またはIPへ向けるだけの簡単な設定にできます。

```Caddyfile
# Simple setup
example.com {
    reverse_proxy leptos-app:8080
}

# Advanced: Basic auth and HSTS headers
app.example.com {
    # Protect a staging site with basic auth
    basic_auth {
        admin $2a$14$CIW9S... 
    }
    
    header {
        Strict-Transport-Security "max-age=31536000; includeSubDomains; preload"
    }

    reverse_proxy leptos-app:8080
}
```
詳しくは、[Caddy Reverse Proxy Quick-start](https://caddyserver.com/docs/quick-starts/reverse-proxy)や[Caddyfileの概念に関するドキュメント](https://caddyserver.com/docs/caddyfile)などを参照してください。

## クラウドへのデプロイ

### Fly.ioへデプロイする

Leptos SSRアプリをデプロイする選択肢の1つが [Fly.io](https://fly.io/) のようなサービスです。LeptosアプリのDockerfile定義を受け取り、起動の速いmicro VMで実行します。Flyはプロジェクトで使えるさまざまなストレージとマネージドDBも提供します。次の例では、まず使い始められるよう、単純なLeptosスターターアプリをデプロイします。必要になったら、[Fly.ioのストレージの扱いについてはこちら](https://fly.io/docs/database-storage-guides/)を参照してください。

まずアプリケーションのルートに `Dockerfile` を作成し、上で示した内容を記述します。Dockerfileの例にあるバイナリ名を自分のアプリケーション名へ変更し、必要に応じてほかの部分も調整してください。

また、`flyctl` CLIツールがインストールされ、[Fly.io](https://fly.io/) のアカウントが設定済みであることを確認します。macOS、Linux、Windows WSLへ `flyctl` をインストールするには次を実行します。

```sh
curl -L https://fly.io/install.sh | sh
```

問題がある場合や、ほかのプラットフォームへインストールする場合は、[こちらの完全な手順](https://fly.io/docs/hands-on/install-flyctl/)を参照してください。

続いてFly.ioへログインします。

```sh
fly auth login
```

そして次のコマンドでアプリを手動起動します。

```sh
fly launch
```

`flyctl` CLIツールが、Fly.ioへアプリをデプロイする手順を案内します。

```admonish note
デフォルトでは、Fly.ioは一定期間トラフィックが来ないmachineを自動停止します。Fly.ioの軽量VMは高速に起動しますが、Leptosアプリの遅延を最小限にし、常にすばやく応答させたい場合は、生成された `fly.toml` ファイルで `min_machines_running` をデフォルトの0から1へ変更してください。

[詳しくはFly.ioドキュメントのこちらのページをご覧ください](https://fly.io/docs/apps/autostart-stop/)。
```

GitHub Actionsでデプロイを管理する場合は、[Fly.io](https://fly.io/) のWeb UIで新しいアクセストークンを作成する必要があります。

「Account」>「Access Tokens」を開いて「github_actions」などの名前でtokenを作成します。続いてプロジェクトのGitHubリポジトリを開き、「Settings」>「Secrets and Variables」>「Actions」をクリックし、「FLY_API_TOKEN」という名前の「New repository secret」を作成してtokenを追加します。

Fly.ioへのデプロイ用 `fly.toml` 設定ファイルを生成するには、まずプロジェクトのソースディレクトリ内で次を実行します。

```sh
fly launch --no-deploy
```

これにより新しいFlyアプリを作成し、サービスへ登録します。新しい `fly.toml` ファイルをGitへcommitしてください。

GitHub Actionsのデプロイworkflowを設定するには、次の内容を `.github/workflows/fly_deploy.yml` ファイルへコピーします。

```admonish example collapsible=true

	# For more details, see: https://fly.io/docs/app-guides/continuous-deployment-with-github-actions/

	name: Deploy to Fly.io
	on:
	push:
		branches:
		- main
	jobs:
	deploy:
		name: Deploy app
		runs-on: ubuntu-latest
		steps:
		- uses: actions/checkout@v4
		- uses: superfly/flyctl-actions/setup-flyctl@master
		- name: Deploy to fly
			id: deployment
			run: |
			  flyctl deploy --remote-only | tail -n 1 >> $GITHUB_STEP_SUMMARY
			env:
			  FLY_API_TOKEN: ${{ secrets.FLY_API_TOKEN }}

```

次にGitHubの `main` ブランチへcommitすると、プロジェクトがFly.ioへ自動的にデプロイされます。

[サンプルリポジトリはこちら](https://github.com/diversable/fly-io-leptos-ssr-test-deploy)です。

### Railway

クラウドデプロイの別のproviderに [Railway](https://railway.app/) があります。RailwayはGitHubと統合し、コードを自動的にデプロイします。

すぐに使い始められる、規約を取り入れたコミュニティテンプレートがあります。

[![Deploy on Railway](https://railway.app/button.svg)](https://railway.app/template/pduaM5?referralCode=fZ-SY1)

このテンプレートには依存関係を最新に保つRenovateの設定があり、デプロイ前にコードをテストするGitHub Actionsにも対応しています。

Railwayにはクレジットカード不要の無料枠があり、Leptosが必要とするリソースは少ないため、無料枠を長く利用できるはずです。

[サンプルリポジトリはこちら](https://github.com/marvin-bitterlich/leptos-railway)です。

## サーバーレスruntimeへデプロイする

LeptosはAWS LambdaのようなFaaS（Function as a Service）、つまり「サーバーレス」runtimeに加え、[Deno](https://deno.com/deploy)やCloudflareのような [WinterCG](https://wintercg.org/) 互換JS runtimeへのデプロイにも対応します。ただしサーバーレス環境では、VMやコンテナ形式のデプロイと比べてSSRアプリで利用できる機能に制約があることに注意してください（以下を参照）。

### AWS Lambda

[Cargo Lambda](https://www.cargo-lambda.info/) ツールを利用すると、Leptos SSRアプリをAWS Lambdaへデプロイできます。Axumをサーバーとして使うスターターテンプレートが [leptos-rs/start-aws](https://github.com/leptos-rs/start-aws) にあり、その手順はLeptos＋Actix-webサーバー向けにも応用できます。スターターリポジトリにはCI/CD用GitHub Actions scriptのほか、Lambda関数の設定とクラウドデプロイに必要な認証情報を取得する手順も含まれます。

ただし、LambdaのようなFaaSサービスではリクエストごとに環境が同一とは限らないため、一部のネイティブなサーバー機能が動作しないことを覚えておいてください。特に [`start-aws` のドキュメント](https://github.com/leptos-rs/start-aws#state)では、AWS Lambdaはサーバーレスプラットフォームなので、長期間存続するstateの管理には一層注意が必要だと説明しています。ディスクへの書き込みやstate extractorの使用は、複数のリクエストにわたって確実には動作しません。代わりに、Lambda関数から問い合わせられるデータベースやその他のmicroserviceが必要です。

もう1つ考慮すべき要因が、FaaSの「コールドスタート」時間です。ユースケースと利用するFaaSプラットフォームによっては、遅延要件を満たせない場合があります。リクエスト速度を最適化するため、1つの関数を常時稼働させる必要があるかもしれません。

### DenoとCloudflare Workers

現在Leptos-Axumは、DenoやCloudflare Workersなど、JavaScriptがホストするWebAssembly runtimeでの実行に対応しています。この選択肢ではソースコードの設定を一部変更する必要があります（たとえば `Cargo.toml` で `crate-type = ["cdylib"]` を使ってアプリを定義し、`leptos_axum` の「wasm」featureを有効にします）。[Leptos HackerNewsのJS-fetchサンプル](https://github.com/leptos-rs/leptos/tree/leptos_0.6/examples/hackernews_js_fetch)は必要な変更を示し、Deno runtimeでアプリを動かす方法を実演します。また、JSホスト型WASM runtime用の `Cargo.toml` を設定するときは、[`leptos_axum` crateのドキュメント](https://docs.rs/leptos_axum/latest/leptos_axum/#js-fetch-integration)も参考になります。

JSホスト型WASM runtimeの初期設定は難しくありませんが、より重要な制約を覚えておく必要があります。アプリはクライアントだけでなくサーバーでもWebAssembly（`wasm32-unknown-unknown`）へコンパイルされるため、使用するすべてのcrateがWASM互換でなければなりません。Rust ecosystemのすべてのcrateがWASMに対応しているわけではないため、アプリの要件によっては致命的な制約になる場合も、ならない場合もあります。

サーバー側WASMの制約を受け入れられるなら、現時点で最適な出発点は、Leptos公式GitHubリポジトリの[DenoでLeptosを実行する例](https://github.com/leptos-rs/leptos/tree/leptos_0.6/examples/hackernews_js_fetch)を確認することです。

## Leptos対応に取り組んでいるプラットフォーム

### Spin Serverless WASIへデプロイする（Leptos SSR）

最近はサーバー上のWebAssemblyが勢いを増しており、オープンソースのサーバーレスWebAssemblyフレームワークSpinの開発者は、Leptosのネイティブ対応に取り組んでいます。LeptosとSpinのSSR統合はまだ初期段階ですが、試せる動作例があります。

Leptos SSRとSpinを連携させる完全な手順は、[Fermyonブログの記事](https://www.fermyon.com/blog/leptos-spin-get-started)にあります。記事を飛ばして、動作するスターターリポジトリをすぐ試したい場合は[こちら](https://github.com/diversable/leptos-spin-ssr-test)をご覧ください。

### Shuttle.rsへデプロイする

複数のLeptosユーザーから、Rustと親和性の高い [Shuttle.rs](https://www.shuttle.rs/) サービスでLeptosアプリをデプロイできるか質問がありました。残念ながら現在、Shuttle.rsサービスはLeptosを正式にはサポートしていません。

ただしShuttle.rsの方々は将来のLeptos対応に取り組んでいます。進捗を追いたい場合は、[こちらのGitHub issue](https://github.com/shuttle-hq/shuttle/issues/1002#issuecomment-1853661643)に注目してください。

また、ShuttleをLeptosで動かす試みも行われていますが、現時点ではShuttle cloudへのデプロイはまだ期待どおりに動作しません。自分で調査したり修正へ貢献したりしたい場合は、こちらで作業内容を確認できます：[Shuttle.rs用Leptos Axumスターターテンプレート](https://github.com/Rust-WASI-WASM/shuttle-leptos-axum)。
