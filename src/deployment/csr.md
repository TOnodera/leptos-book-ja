# クライアントサイドレンダリングするアプリのデプロイ

クライアントサイドレンダリングだけを使うアプリを、開発サーバー兼ビルドツールとしてTrunkを使って構築してきたなら、手順はとても簡単です。

```bash
trunk build --release
```

`trunk build` は `dist/` ディレクトリへ複数のビルド成果物を作成します。アプリのデプロイに必要なのは、`dist` をオンラインのどこかへ公開することだけです。一般的なJavaScriptアプリケーションのデプロイとよく似ています。

Leptos CSRアプリをさまざまなホスティングサービス向けに設定・デプロイする方法を示すサンプルリポジトリを、いくつか用意しました。

_注意：Leptosは特定のホスティングサービスの利用を推奨していません。静的サイトのデプロイに対応する任意のサービスを自由に使ってください。_

例：

- [Github Pages](#github-pages)
- [Vercel](#vercel)
- [Spin (serverless WebAssembly)](#spin---serverless-webassembly)
- [Netlify](#netlify)

## GitHub Pages

Leptos CSRアプリをGitHub Pagesへデプロイするのは簡単です。まずGitHubリポジトリの設定を開き、左側のメニューで「Pages」をクリックします。ページの「Build and deployment」セクションで「Source」を「GitHub Actions」へ変更します。続いて、次の内容を `.github/workflows/gh-pages-deploy.yml` などのファイルへコピーします。

```admonish example collapsible=true

    name: Release to Github Pages

    on:
      push:
        branches: [main]
      workflow_dispatch:

    permissions:
      contents: write # for committing to gh-pages branch.
      pages: write
      id-token: write

    # Allow only one concurrent deployment, skipping runs queued between the run in-progress and latest queued.
    # However, do NOT cancel in-progress runs as we want to allow these production deployments to complete.
    concurrency:
      group: "pages"
      cancel-in-progress: false

    jobs:
      Github-Pages-Release:

        timeout-minutes: 10

        environment:
          name: github-pages
          url: ${{ steps.deployment.outputs.page_url }}

        runs-on: ubuntu-latest

        steps:
          - uses: actions/checkout@v4 # repo checkout

          # Install Rust Nightly Toolchain, with Clippy & Rustfmt
          - name: Install nightly Rust
            uses: dtolnay/rust-toolchain@nightly
            with:
              components: clippy, rustfmt

          - name: Add WASM target
            run: rustup target add wasm32-unknown-unknown

          - name: lint
            run: cargo clippy & cargo fmt


          # If using tailwind...
          # - name: Download and install tailwindcss binary
          #   run: npm install -D tailwindcss && npx tailwindcss -i <INPUT/PATH.css> -o <OUTPUT/PATH.css>  # run tailwind


          - name: Download and install Trunk binary
            run: wget -qO- https://github.com/trunk-rs/trunk/releases/download/v0.18.4/trunk-x86_64-unknown-linux-gnu.tar.gz | tar -xzf-

          - name: Build with Trunk
            # "${GITHUB_REPOSITORY#*/}" evaluates into the name of the repository
            # using --public-url something will allow trunk to modify all the href paths like from favicon.ico to repo_name/favicon.ico .
            # this is necessary for github pages where the site is deployed to username.github.io/repo_name and all files must be requested
            # relatively as favicon.ico. if we skip public-url option, the href paths will instead request username.github.io/favicon.ico which
            # will obviously return error 404 not found.
            run: ./trunk build --release --public-url "${GITHUB_REPOSITORY#*/}"

          # Copy index.html to 404.html for SPA routing
          # Will allow routing to work if client enters from any route
          # - name: Copy index.html to 404.html
          #   run: cp dist/index.html dist/404.html

          # Deploy to gh-pages branch
          # - name: Deploy 🚀
          #   uses: JamesIves/github-pages-deploy-action@v4
          #   with:
          #     folder: dist


          # Deploy with Github Static Pages

          - name: Setup Pages
            uses: actions/configure-pages@v5
            with:
              enablement: true
              # token:

          - name: Upload artifact
            uses: actions/upload-pages-artifact@v3
            with:
              # Upload dist dir
              path: './dist'

          - name: Deploy to GitHub Pages 🚀
            id: deployment
            uses: actions/deploy-pages@v4

```

GitHub Pagesへのデプロイについて詳しくは、[こちらのサンプルリポジトリ](https://github.com/diversable/deploy_leptos_csr_to_gh_pages)をご覧ください。

## Vercel

### 手順1：Vercelを設定する

VercelのWeb UIで……

1. 新しいプロジェクトを作成します。
2. 次を確認します。
   - 「Build Command」はOverrideを有効にしたまま空にします。
   - 「Output Directory」をdist（Trunkビルドのデフォルト出力先）へ変更し、Overrideを有効にします。

<img src="./image.png" />

### 手順2：GitHub Actions用のVercel認証情報を追加する

注意：プレビューとデプロイの両Actionで、GitHub Secretsに設定したVercel認証情報が必要です。

1. 「Account Settings」>「Tokens」を開いて新しいtokenを作成し、[Vercel Access Token](https://vercel.com/guides/how-do-i-use-a-vercel-api-access-token)を取得します。下の手順5で使うため保存してください。

2. `npm i -g vercel` コマンドで [Vercel CLI](https://vercel.com/cli) をインストールし、`vercel login` を実行してアカウントへログインします。

3. プロジェクトのフォルダー内で `vercel link` を実行し、新しいVercelプロジェクトを作成します。CLIで「Link to an existing project?」と尋ねられたらyesと答え、手順1で作った名前を入力します。新しい `.vercel` フォルダーが作成されます。

4. 生成された `.vercel` フォルダーで `project.json` ファイルを開き、次の手順で使う「projectId」と「orgId」を保存します。

5. GitHubでリポジトリの「Settings」>「Secrets and Variables」>「Actions」を開き、次を[Repository secrets](https://docs.github.com/en/actions/security-guides/encrypted-secrets)として追加します。
   - 手順1のVercel Access Tokenを `VERCEL_TOKEN` secretとして保存する
   - `.vercel/project.json` の「projectID」を `VERCEL_PROJECT_ID` として追加する
   - `.vercel/project.json` の「orgId」を `VERCEL_ORG_ID` として追加する

<i>完全な手順は[「How can I use Github Actions with Vercel」](https://vercel.com/guides/how-can-i-use-github-actions-with-vercel)を参照してください。</i>

### 手順3：GitHub Actionsのscriptを追加する

最後に、デプロイ用とPRプレビュー用の2ファイルを、以下または[サンプルリポジトリの `.github/workflows/` フォルダー](https://github.com/diversable/vercel-leptos-CSR-deployment/tree/leptos_0.6/.github/workflows)から自分のGitHub workflowsフォルダーへコピー＆ペーストします。その後は、次のcommitまたはPRで自動的にデプロイされます。

<i>本番デプロイ用script：`vercel_deploy.yml`</i>

```admonish example collapsible=true

	name: Release to Vercel

	on:
	push:
		branches:
		- main
	env:
	CARGO_TERM_COLOR: always
	VERCEL_ORG_ID: ${{ secrets.VERCEL_ORG_ID }}
	VERCEL_PROJECT_ID: ${{ secrets.VERCEL_PROJECT_ID }}

	jobs:
	Vercel-Production-Deployment:
		runs-on: ubuntu-latest
		environment: production
		steps:
		- name: git-checkout
			uses: actions/checkout@v3

		- uses: dtolnay/rust-toolchain@nightly
			with:
			components: clippy, rustfmt
		- uses: Swatinem/rust-cache@v2
		- name: Setup Rust
			run: |
			rustup target add wasm32-unknown-unknown
			cargo clippy
			cargo fmt --check

		- name: Download and install Trunk binary
			run: wget -qO- https://github.com/trunk-rs/trunk/releases/download/v0.18.2/trunk-x86_64-unknown-linux-gnu.tar.gz | tar -xzf-


		- name: Build with Trunk
			run: ./trunk build --release

		- name: Install Vercel CLI
			run: npm install --global vercel@latest

		- name: Pull Vercel Environment Information
			run: vercel pull --yes --environment=production --token=${{ secrets.VERCEL_TOKEN }}

		- name: Deploy to Vercel & Display URL
			id: deployment
			working-directory: ./dist
			run: |
			vercel deploy --prod --token=${{ secrets.VERCEL_TOKEN }} >> $GITHUB_STEP_SUMMARY
			echo $GITHUB_STEP_SUMMARY

```

<i>プレビューデプロイ用script：`vercel_preview.yml`</i>

```admonish example collapsible=true

	# For more info re: vercel action see:
	# https://github.com/amondnet/vercel-action

	name: Leptos CSR Vercel Preview

	on:
	pull_request:
		branches: [ "main" ]

	workflow_dispatch:

	env:
	CARGO_TERM_COLOR: always
	VERCEL_ORG_ID: ${{ secrets.VERCEL_ORG_ID }}
	VERCEL_PROJECT_ID: ${{ secrets.VERCEL_PROJECT_ID }}

	jobs:
	fmt:
		name: Rustfmt
		runs-on: ubuntu-latest
		steps:
		- uses: actions/checkout@v4
		- uses: dtolnay/rust-toolchain@nightly
			with:
			components: rustfmt
		- name: Enforce formatting
			run: cargo fmt --check

	clippy:
		name: Clippy
		runs-on: ubuntu-latest
		steps:
		- uses: actions/checkout@v4
		- uses: dtolnay/rust-toolchain@nightly
			with:
			components: clippy
		- uses: Swatinem/rust-cache@v2
		- name: Linting
			run: cargo clippy -- -D warnings

	test:
		name: Test
		runs-on: ubuntu-latest
		needs: [fmt, clippy]
		steps:
		- uses: actions/checkout@v4
		- uses: dtolnay/rust-toolchain@nightly
		- uses: Swatinem/rust-cache@v2
		- name: Run tests
			run: cargo test

	build-and-preview-deploy:
		runs-on: ubuntu-latest
		name: Build and Preview

		needs: [test, clippy, fmt]

		permissions:
		pull-requests: write

		environment:
		name: preview
		url: ${{ steps.preview.outputs.preview-url }}

		steps:
		- name: git-checkout
			uses: actions/checkout@v4

		- uses: dtolnay/rust-toolchain@nightly
		- uses: Swatinem/rust-cache@v2
		- name: Build
			run: rustup target add wasm32-unknown-unknown

		- name: Download and install Trunk binary
			run: wget -qO- https://github.com/trunk-rs/trunk/releases/download/v0.18.2/trunk-x86_64-unknown-linux-gnu.tar.gz | tar -xzf-


		- name: Build with Trunk
			run: ./trunk build --release

		- name: Preview Deploy
			id: preview
			uses: amondnet/vercel-action@v25.1.1
			with:
			vercel-token: ${{ secrets.VERCEL_TOKEN }}
			github-token: ${{ secrets.GITHUB_TOKEN }}
			vercel-org-id: ${{ secrets.VERCEL_ORG_ID }}
			vercel-project-id: ${{ secrets.VERCEL_PROJECT_ID }}
			github-comment: true
			working-directory: ./dist

		- name: Display Deployed URL
			run: |
			echo "Deployed app URL: ${{ steps.preview.outputs.preview-url }}" >> $GITHUB_STEP_SUMMARY


```

詳しくは[こちらのサンプルリポジトリ](https://github.com/diversable/vercel-leptos-CSR-deployment)をご覧ください。

## Spin - サーバーレスWebAssembly

別の選択肢は、Spinのようなサーバーレスプラットフォームです。[Spin](https://github.com/fermyon/spin) はオープンソースで、自分のインフラストラクチャ（たとえばKubernetes内）でも実行できますが、本番環境でSpinを始める最も簡単な方法はFermyon Cloudを使うことです。

まず[こちらの手順でSpin CLIをインストール](https://developer.fermyon.com/spin/v2/install)し、まだであればLeptos CSRプロジェクト用のGitHubリポジトリを作成します。

1. 「Fermyon Cloud」>「User Settings」を開きます。未ログインなら「Login With GitHub」ボタンを選びます。

2. 「Personal Access Tokens」で「Add a Token」を選びます。名前に「gh_actions」と入力し、「Create Token」をクリックします。

3. Fermyon Cloudにtokenが表示されたら、コピーボタンでクリップボードへコピーします。

4. GitHubリポジトリで「Settings」>「Secrets and Variables」>「Actions」を開き、Fermyon Cloudのtokenを変数名「FERMYON_CLOUD_TOKEN」で「Repository secrets」へ追加します。

5. 以下のGitHub Actions scriptを `.github/workflows/<SCRIPT_NAME>.yml` ファイルへコピー＆ペーストします。

6. 「preview」と「deploy」のscriptを有効にすると、GitHub ActionsはPull Requestでプレビューを生成し、「main」ブランチの更新時に自動デプロイします。

<i>本番デプロイ用script：`spin_deploy.yml`</i>

```admonish example collapsible=true

	# For setup instructions needed for Fermyon Cloud, see:
	# https://developer.fermyon.com/cloud/github-actions

	# For reference, see:
	# https://developer.fermyon.com/cloud/changelog/gh-actions-spin-deploy

	# For the Fermyon gh actions themselves, see:
	# https://github.com/fermyon/actions

	name: Release to Spin Cloud

	on:
	push:
		branches: [main]
	workflow_dispatch:

	permissions:
	contents: read
	id-token: write

	# Allow only one concurrent deployment, skipping runs queued between the run in-progress and latest queued.
	# However, do NOT cancel in-progress runs as we want to allow these production deployments to complete.
	concurrency:
	group: "spin"
	cancel-in-progress: false

	jobs:
	Spin-Release:

		timeout-minutes: 10

		environment:
		name: production
		url: ${{ steps.deployment.outputs.app-url }}

		runs-on: ubuntu-latest

		steps:
		- uses: actions/checkout@v4 # repo checkout

		# Install Rust Nightly Toolchain, with Clippy & Rustfmt
		- name: Install nightly Rust
			uses: dtolnay/rust-toolchain@nightly
			with:
			components: clippy, rustfmt

		- name: Add WASM & WASI targets
			run: rustup target add wasm32-unknown-unknown && rustup target add wasm32-wasi

		- name: lint
			run: cargo clippy & cargo fmt


		# If using tailwind...
		# - name: Download and install tailwindcss binary
		#   run: npm install -D tailwindcss && npx tailwindcss -i <INPUT/PATH.css> -o <OUTPUT/PATH.css>  # run tailwind


		- name: Download and install Trunk binary
			run: wget -qO- https://github.com/trunk-rs/trunk/releases/download/v0.18.2/trunk-x86_64-unknown-linux-gnu.tar.gz | tar -xzf-


		- name: Build with Trunk
			run: ./trunk build --release


		# Install Spin CLI & Deploy

		- name: Setup Spin
			uses: fermyon/actions/spin/setup@v1
			# with:
			# plugins:


		- name: Build and deploy
			id: deployment
			uses: fermyon/actions/spin/deploy@v1
			with:
			fermyon_token: ${{ secrets.FERMYON_CLOUD_TOKEN }}
			# key_values: |-
				# abc=xyz
				# foo=bar
			# variables: |-
				# password=${{ secrets.SECURE_PASSWORD }}
				# apikey=${{ secrets.API_KEY }}

		# Create an explicit message to display the URL of the deployed app, as well as in the job graph
		- name: Deployed URL
			run: |
			echo "Deployed app URL: ${{ steps.deployment.outputs.app-url }}" >> $GITHUB_STEP_SUMMARY

```

<i>プレビューデプロイ用script：`spin_preview.yml`</i>

```admonish example collapsible=true

	# For setup instructions needed for Fermyon Cloud, see:
	# https://developer.fermyon.com/cloud/github-actions


	# For the Fermyon gh actions themselves, see:
	# https://github.com/fermyon/actions

	# Specifically:
	# https://github.com/fermyon/actions?tab=readme-ov-file#deploy-preview-of-spin-app-to-fermyon-cloud---fermyonactionsspinpreviewv1

	name: Preview on Spin Cloud

	on:
	pull_request:
		branches: ["main", "v*"]
		types: ['opened', 'synchronize', 'reopened', 'closed']
	workflow_dispatch:

	permissions:
	contents: read
	pull-requests: write

	# Allow only one concurrent deployment, skipping runs queued between the run in-progress and latest queued.
	# However, do NOT cancel in-progress runs as we want to allow these production deployments to complete.
	concurrency:
	group: "spin"
	cancel-in-progress: false

	jobs:
	Spin-Preview:

		timeout-minutes: 10

		environment:
		name: preview
		url: ${{ steps.preview.outputs.app-url }}

		runs-on: ubuntu-latest

		steps:
		- uses: actions/checkout@v4 # repo checkout

		# Install Rust Nightly Toolchain, with Clippy & Rustfmt
		- name: Install nightly Rust
			uses: dtolnay/rust-toolchain@nightly
			with:
			components: clippy, rustfmt

		- name: Add WASM & WASI targets
			run: rustup target add wasm32-unknown-unknown && rustup target add wasm32-wasi

		- name: lint
			run: cargo clippy & cargo fmt


		# If using tailwind...
		# - name: Download and install tailwindcss binary
		#   run: npm install -D tailwindcss && npx tailwindcss -i <INPUT/PATH.css> -o <OUTPUT/PATH.css>  # run tailwind


		- name: Download and install Trunk binary
			run: wget -qO- https://github.com/trunk-rs/trunk/releases/download/v0.18.2/trunk-x86_64-unknown-linux-gnu.tar.gz | tar -xzf-


		- name: Build with Trunk
			run: ./trunk build --release


		# Install Spin CLI & Deploy

		- name: Setup Spin
			uses: fermyon/actions/spin/setup@v1
			# with:
			# plugins:


		- name: Build and preview
			id: preview
			uses: fermyon/actions/spin/preview@v1
			with:
			fermyon_token: ${{ secrets.FERMYON_CLOUD_TOKEN }}
			github_token: ${{ secrets.GITHUB_TOKEN }}
			undeploy: ${{ github.event.pull_request && github.event.action == 'closed' }}
			# key_values: |-
				# abc=xyz
				# foo=bar
			# variables: |-
				# password=${{ secrets.SECURE_PASSWORD }}
				# apikey=${{ secrets.API_KEY }}


		- name: Display Deployed URL
			run: |
			echo "Deployed app URL: ${{ steps.preview.outputs.app-url }}" >> $GITHUB_STEP_SUMMARY

```

詳しくは[こちらのサンプルリポジトリ](https://github.com/diversable/leptos-spin-CSR)をご覧ください。

# Netlify

Leptos CSRアプリをNetlifyへデプロイするには、プロジェクトを作成し、プロジェクトのルートへ2つの簡単な設定ファイルを追加するだけです。後者から始めましょう。

## 設定ファイル

プロジェクトのルートへ、次の内容で `netlify.toml` ファイルを作成します。

```toml
[build]
command = "rustup target add wasm32-unknown-unknown && cargo install trunk --locked && trunk build --release"
publish = "dist"

[build.environment]
RUST_VERSION = "stable"

[[redirects]]
from = "/*"
to = "/index.html"
status = 200
```

プロジェクトのルートへ、次の内容で `rust-toolchain.toml` ファイルを作成します。

```toml
[toolchain]
channel = "stable"
targets = ["wasm32-unknown-unknown"]
```

## デプロイ

1. Gitリポジトリを接続して[プロジェクトをNetlifyへ追加](https://docs.netlify.com/start/add-new-project/)します。
2. Netlifyが `netlify.toml` の設定を自動的に検出します。
3. 追加の環境変数が必要なら、[Netlifyの環境変数設定](https://docs.netlify.com/build/environment-variables/overview/)で設定します。

`rust-toolchain.toml` は、ビルド過程で正しいRust toolchainとWASM targetを利用できるようにします。`netlify.toml` のリダイレクト規則は、すべてのパスに `index.html` を配信し、SPAのルートが正しく動作するようにします。
