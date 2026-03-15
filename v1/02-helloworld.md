# プロジェクトの作成とhello world

この章では、まずはUzumibiを動かすため、インストールとhello worldプロジェクトの作成、動作確認まで行います。

## uzumibi-cliのインストール

Uzumibiでの開発を始めるには、まず `uzumibi-cli` とその前提となるツール群をインストールする必要があります。

### 前提条件

以下のツールが事前にインストールされている必要があります。

#### Rustツールチェイン

UzumibiのプロジェクトビルドにはRustコンパイラが必要です。`rustup` を使ってインストールします。

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

インストール後、WASMターゲットを追加します。

```bash
rustup target add wasm32-unknown-unknown
```

バージョンを確認します。

```bash
$ rustc --version
rustc 1.93.1 (01f6ddf75 2026-02-11)
```

#### Node.js と pnpm

Cloudflare Workersでは、Wranglerを使った開発・デプロイにNode.jsが必要です。

```bash
# Node.jsのインストール（バージョンマネージャーを使う場合の例）
# fnmの場合
fnm install --lts
# nvm の場合
nvm install --lts

# pnpmのインストール
npm install -g pnpm
```

バージョンを確認します。

```bash
$ node --version
v25.6.1

$ pnpm --version
10.29.3
```

#### Wrangler CLI

Cloudflare Workersの開発・デプロイに使用するCLIツールです。プロジェクト作成後に `pnpm install` で自動的にインストールされますが、グローバルにインストールしておくと便利でしょう。

```bash
pnpm install -g wrangler
```

#### clang / ビルドツール

mrubyのコンパイラ（`mruby-conpiler2`）のビルドにCコンパイラの `clang` が必要です。

macOSの場合：

```bash
xcode-select --install
```

Linuxの場合：

```bash
# Ubuntu/Debian
sudo apt-get install clang build-essential

# Fedora
sudo dnf install clang
```

### uzumibi-cliのインストール

`uzumibi-cli` はRustのパッケージマネージャー `cargo` を使ってインストールします。

```bash
cargo install 'uzumibi-cli@^0.6'
```

インストールが完了したら、バージョンを確認します。

```bash
$ uzumibi --version
uzumibi-cli 0.6.1
```

ヘルプを表示して、利用可能なコマンドを確認できます。

```bash
$ uzumibi --help
```

これで開発環境の準備は完了です。

## プロジェクトの作成

`uzumibi new` コマンドを使って、Cloudflare Workers向けのプロジェクトを作成します。

### プロジェクトの生成

以下のコマンドで `hello-uzumibi` という名前のプロジェクトを作成します。

```bash
uzumibi new --template cloudflare hello-uzumibi
```

実行すると、プロジェクトのディレクトリが作成され、必要なファイルが生成されます。

```
Creating project 'hello-uzumibi'...
  generate  hello-uzumibi/.gitignore
  generate  hello-uzumibi/Cargo.toml
  generate  hello-uzumibi/package.json
  generate  hello-uzumibi/vitest.config.js
  generate  hello-uzumibi/wrangler.jsonc
  generate  hello-uzumibi/lib/app.rb
  generate  hello-uzumibi/public/assets/index.html
  generate  hello-uzumibi/src/index.js
  generate  hello-uzumibi/wasm-app/Cargo.toml
  generate  hello-uzumibi/wasm-app/build.rs
  generate  hello-uzumibi/wasm-app/src/lib.rs

✓ Successfully created project from template 'cloudflare'
  Run 'cd hello-uzumibi' to get started!
```

### プロジェクトのディレクトリ構成

生成されたプロジェクトは以下のような構成になっています。

```
hello-uzumibi/
├── Cargo.toml          # Rustワークスペースの設定
├── package.json        # Node.jsの依存関係とスクリプト
├── wrangler.jsonc      # Wrangler（Cloudflare Workers CLI）の設定
├── lib/
│   └── app.rb          # Rubyアプリケーションコード（メイン）
├── public/
│   └── assets/...      # 静的アセット（HTML, CSS, 画像など）
├── src/
│   └── index.js        # JavaScriptグルーコード（エントリーポイント）
└── wasm-app/
    ├── Cargo.toml      # WASMクレートの設定
    ├── build.rs        # ビルドスクリプト（Rubyコードのコンパイル）
    ├── src/
    │   └── lib.rs      # WASMモジュールのRustコード
    └── .cargo/
        └── config.toml # Cargoのターゲット設定（無い場合もあります）
```

### 各ファイルの役割

#### `lib/app.rb`

**開発者が主に編集するファイル**です。Uzumibiのルーティングとリクエスト処理のロジックをRubyで記述します。

#### `src/index.js`

Cloudflare Workersのエントリーポイントです。HTTPリクエストを受け取り、バイナリ形式にシリアライズしてWASMモジュールに渡し、レスポンスをデシリアライズしてクライアントに返却します。通常、このファイルを編集する必要はありません。

#### `wasm-app/`

Rubyコードをmrubyバイトコードにコンパイルし、WASMモジュールとしてパッケージングするためのRustクレートです。通常、このファイルを編集する必要はありません。

- `build.rs` はビルド時に `lib/app.rb` を mrubyバイトコード（`.mrb`）にコンパイルする設定が含まれています。
- `src/lib.rs` はmruby/edge VMの初期化と、エクスポート関数の定義を行います。

#### `wrangler.jsonc`

Cloudflare Workersの設定ファイルです。アプリケーション名、静的アセットの設定などが含まれています。

```jsonc
{
    "name": "hello-uzumibi",
    "main": "src/index.js",
    "compatibility_date": "2025-12-30",
    "assets": {
        "directory": "./public",
        "binding": "ASSETS"
    }
}
```

#### `package.json`

npm/pnpmのスクリプトと依存関係を定義しています。

```json
{
    "scripts": {
        "deploy": "wrangler deploy",
        "dev": "cargo build --package hello-uzumibi --target wasm32-unknown-unknown --release && cp -v -f target/wasm32-unknown-unknown/release/hello_uzumibi.wasm src/ && wrangler dev",
        "start": "wrangler dev",
        "test": "vitest"
    }
}
```

`dev` スクリプトは、Rustのビルド（WASMコンパイル）とWranglerの開発サーバー起動をまとめて行います。

### 依存関係のインストール

プロジェクトディレクトリに移動して、Node.jsの依存関係をインストールします。

```bash
cd hello-uzumibi
pnpm install
```

これで、Wranglerをはじめとする開発ツールがプロジェクトローカルにインストールされます。

## hello worldアプリケーションの実装

プロジェクトが作成できたので、実際にアプリケーションを実装していきましょう。UzumibiではRubyのコード（`lib/app.rb`）と、静的なフロントエンド（`public/` ディレクトリ）の両方を扱えます。

### フロントの実装

まず、`public/` ディレクトリに静的なHTMLファイルを配置してみましょう。`public/index.html` を作成します。

```html
<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <title>Hello Uzumibi</title>
    <style>
        body {
            font-family: sans-serif;
            max-width: 600px;
            margin: 50px auto;
            padding: 0 20px;
        }
        #result {
            margin-top: 20px;
            padding: 15px;
            background: #f0f0f0;
            border-radius: 5px;
        }
    </style>
</head>
<body>
    <h1>Hello Uzumibi!</h1>
    <p>RubyでCloudflare Workersをやっていく気持ち</p>

    <button id="greetBtn">APIを呼び出す</button>
    <div id="result"></div>

    <script>
        document.getElementById('greetBtn').addEventListener('click', async () => {
            const res = await fetch('/api/hello');
            const text = await res.text();
            document.getElementById('result').textContent = text;
        });
    </script>
</body>
</html>
```

`public/` ディレクトリに配置したファイルは、Cloudflare Workersの静的アセット機能によって自動的に配信されます。

### APIの実装

次に、`lib/app.rb` を編集して、APIエンドポイントを実装します。デフォルトで生成されたコードを以下のように書き換えます。

```ruby
class App < Uzumibi::Router
  # ルートパスは静的アセット（public/index.html）に委譲
  get "/" do |req, res|
    fetch_assets
  end

  # シンプルなAPIエンドポイント
  get "/api/hello" do |req, res|
    res.status_code = 200
    res.headers = {
      "content-type" => "text/plain",
      "x-powered-by" => "#{RUBY_ENGINE} #{RUBY_VERSION}"
    }
    res.body = "Hello from Uzumibi! Running on #{RUBY_ENGINE} #{RUBY_VERSION}\n"
    res
  end

  # 動的パラメータを使った挨拶API
  get "/api/greet/:name" do |req, res|
    name = req.params[:name]
    res.status_code = 200
    res.headers = {
      "content-type" => "application/json",
    }
    res.body = JSON.generate({ message: "Hello, #{name}!" })
    res
  end

  # POSTリクエストの処理
  post "/api/echo" do |req, res|
    res.status_code = 200
    res.headers = {
      "content-type" => "text/plain",
    }
    res.body = "Received: #{req.body.inspect}\n"
    res
  end
end

$APP = App.new
```

### コードの解説

#### ルーティング

`Uzumibi::Router` を継承したクラスで、`get`、`post` などのクラスメソッドを使ってルートを定義します。

```ruby
class App < Uzumibi::Router
  get "/パス" do |req, res|
    # ハンドラの処理
  end
end
```

各ルートハンドラのブロックには `req`（リクエスト）と `res`（レスポンス）の2つの引数が渡されます。

#### 静的アセットの配信

```ruby
get "/" do |req, res|
  fetch_assets
end
```

`fetch_assets` メソッドを呼ぶと、リクエストが `public/` ディレクトリの静的アセットに委譲されます。これにより、 `/` のパスに対応する `public/index.html` が返されます。

この時、たとえば `"/file"` にハンドラを設定した上で `/file` というリクエストが来た場合は、`public/file.html` が存在すればそれが返されます。拡張子が補完されることに留意してください。

#### レスポンスの構築

レスポンスオブジェクトに `status_code`、`headers`、`body` を設定し、最後に `res` を返します。

```ruby
get "/api/hello" do |req, res|
  res.status_code = 200
  res.headers = {
    "content-type" => "text/plain",
  }
  res.body = "Hello!\n"
  res  # 必ずresを返す
end
```

#### 動的パラメータ

URLパスに `:パラメータ名` を含めると、そのパスセグメントが動的パラメータとしてキャプチャされます。`req.params` は Hash オブジェクトなので、そこを経由してからアクセスできます。

```ruby
get "/api/greet/:name" do |req, res|
  name = req.params[:name]
  # ...
end
```

#### リクエストボディ

POSTリクエストなどのボディは `req.body` で、同期的に取得できます。リクエストの `content-type` がフォームデータ（`application/x-www-form-urlencoded`）やJSONデータ（`application/json`）の場合は自動的にパースされるので、 `req.params` から統合的にアクセスすることも可能です。

#### グローバル変数 `$APP`

```ruby
$APP = App.new
```

Uzumibi `0.6.x` 系の仕様では、最後にアプリケーションのインスタンスをグローバル変数 `$APP` に代入するルールとしています。この名前の変数がWasm側のRustコードから参照されるため、必ず設定する必要があります。

## devサーバーの立ち上げ

アプリケーションのコードが書けたら、開発サーバーを起動して動作を確認しましょう。

### 開発サーバーの起動

以下のコマンドで開発サーバーを起動します。

```bash
pnpm run dev
```

このコマンドは内部で以下の処理を順番に実行します。

1. **Rustのビルド**: `cargo` コマンドを実行し、`lib/app.rb` をmrubyバイトコードにコンパイルした上で、Wasmモジュール（`.wasm`）を生成します。
2. **Wasmファイルのコピー**: ビルドされた `.wasm` ファイルを `src/` ディレクトリにコピーします。
3. **Wrangler devの起動**: `wrangler dev` を実行して、ローカル開発サーバーを起動します。

初回のビルドはRustクレートのダウンロードとコンパイルが必要なため、数分かかることがあります。2回目以降は差分ビルドとなるため、高速に起動します。

ビルドが完了すると、以下のようなメッセージが表示されます。

```
 ⛅️ wrangler 4.73.0
───────────────────

Cloudflare collects anonymous telemetry about your usage of Wrangler. Learn more at https://github.com/cloudflare/workers-sdk/tree/main/packages/wrangler/telemetry.md
Your Worker has access to the following bindings:
Binding            Resource      Mode
env.ASSETS         Assets        local
...

⎔ Starting local server...
[wrangler:info] Ready on http://localhost:8787
```

### 動作確認

開発サーバーが起動したら、ブラウザやcurlで動作を確認します。

#### ブラウザでの確認

ブラウザで `http://localhost:8787` にアクセスすると、`public/index.html` の内容が表示されます。「APIを呼び出す」ボタンをクリックすると、`/api/hello` エンドポイントからのレスポンスが表示されます。

![](https://storage.googleapis.com/zenn-user-upload/4d91a41e016f-20260314.png)

![](https://storage.googleapis.com/zenn-user-upload/46172e036e47-20260314.png)

HTML内のJavaScriptを調整すれば、他のエンドポイントも呼び出して内容を確認することもできます。

#### curlでの確認

APIエンドポイントをcurlで呼び出して、確認してみましょう。

```bash
# ルートパス（静的アセット）
$ curl http://localhost:8787/
<!DOCTYPE html>
<html lang="ja">
...

# APIエンドポイント
$ curl http://localhost:8787/api/hello
Hello from Uzumibi! Running on mruby/edge 3.2.0

# 動的パラメータ
$ curl http://localhost:8787/api/greet/World
{"message": "Hello, World!"}

# POSTリクエスト
$ curl -X POST -d "test data" http://localhost:8787/api/echo
Received: test data
```

### コードの変更を反映する

`lib/app.rb` を変更した場合、現在のUzumibiでは自動リロードには対応していません。Wasmの再ビルドが必要なため、開発サーバーを一度停止（`Ctrl+C`）してから、再度 `pnpm run dev` を実行してください。

```bash
# Ctrl+C で停止
# コードを編集
# 再起動
pnpm run dev
```

### トラブルシューティング

#### ビルドエラー: `clang not found`

mrubyコンパイラのビルドに `clang` が必要です。macOSの場合は `xcode-select --install`、Linuxの場合は `apt install clang` でインストールしてください。

#### ビルドエラー: `wasm32-unknown-unknown target not found`

Wasmターゲットが追加されていない可能性があります。

```bash
rustup target add wasm32-unknown-unknown
```

#### Wranglerの認証エラー

初めてWranglerを使用する場合、ログインが必要なことがあります。

```bash
npx wrangler login
```

#### ポートの競合

デフォルトのポート8787が他のプロセスで使用されている場合、Wranglerは自動的に別のポートを使用します。コンソール出力に表示されるURLを確認してください。

## デプロイ

開発サーバーで動作確認ができたら、Cloudflare Workersにデプロイしてみましょう。

### Cloudflareアカウントの準備

デプロイにはCloudflareのアカウントが必要です。まだアカウントを持っていない場合は、[Cloudflareのダッシュボード](https://dash.cloudflare.com/sign-up)からサインアップしてください。無料プランでCloudflare Workersを利用できます。

### Wranglerへのログイン

Wrangler CLIからCloudflareアカウントにログインします。

```bash
npx wrangler login
```

ブラウザが開き、Cloudflareへの認証を求められます。認証が完了すると、ターミナルにログイン成功のメッセージが表示されます。

### WASMのビルド

デプロイ前に、WASMモジュールをビルドしておく必要があります。`pnpm run dev` を一度でも実行していれば、ビルド済みの `.wasm` ファイルが `src/` ディレクトリにあるはずです。もしビルドしていない場合は、以下のコマンドでビルドだけを実行します。

```bash
cargo build --package hello-uzumibi --target wasm32-unknown-unknown --release
cp target/wasm32-unknown-unknown/release/hello_uzumibi.wasm src/
```

### デプロイの実行

以下のコマンドでCloudflare Workersにデプロイします。

```bash
pnpm run deploy
```

このコマンドは内部で `wrangler deploy` を実行します。

デプロイが開始すると、以下のようなメッセージが表示されます。

```
 ⛅️ wrangler 4.73.0
───────────────────
🌀 Building list of assets...
✨ Read 3 files from the assets directory /Users/udzura/zenn-wip/hello-uzumibi/public
🌀 Starting asset upload...
```

この時、ログに出てくる容量の表示に注目してください。Uzumibiの生成するアーティファクトは、圧縮前でも650KiB程度、gzip圧縮をすれば200KiB程度になります。

```
Total Upload: 655.99 KiB / gzip: 204.28 KiB
```

Cloudflare Workersの無料プランの場合、3MBまでのアセットをアップロードでき、また推奨設定は圧縮後に1MBを切っていることと言われています。

容量が大きい場合Cloudflare Workersでは提供が難しい場合がありますが、Uzumibiとmruby/edgeのアプリケーションは、できるだけ小さなサイズに収まるようにミニマルな機能提供をするよう設計されています。したがってエッジアプリケーションの開発の上で非常に有利と言えるでしょう。

最終的に表示されたURLにアクセスすると、デプロイされたアプリケーションが動作していることを確認できます。

```
Uploaded hello-uzumibi (14.81 sec)
Deployed hello-uzumibi triggers (7.49 sec)
  https://hello-uzumibi.XXXXXX.workers.dev
Current Version ID: b7bcc75c-e557-4c39-b643-xxxxxxxx
```

### 動作確認

ブラウザで `https://hello-uzumibi.<your-subdomain>.workers.dev/` にアクセスすれば、先ほど作成したHTMLページが表示されます。

### wrangler.jsoncの設定

デプロイされるWorkerの名前や設定は `wrangler.jsonc` で管理されています。

```jsonc
{
    "name": "hello-uzumibi",        // Worker名（URLのサブドメインになる）
    "main": "src/index.js",         // エントリーポイント
    "compatibility_date": "2025-12-30",
    "assets": {
        "directory": "./public",    // 静的アセットのディレクトリ
        "binding": "ASSETS"
    }
}
```

Worker名を変更したい場合は `"name"` フィールドを編集してから再デプロイしてください。

### カスタムドメインの設定

デフォルトでは `*.workers.dev` ドメインにデプロイされますが、独自ドメインを設定することもできます。Cloudflareダッシュボードの「Workers & Pages」から設定するか、`wrangler.jsonc` に `routes` を追加してください。

詳細は[Cloudflare Workersの公式ドキュメント](https://developers.cloudflare.com/workers/configuration/routing/)を参照してください。

## この章のまとめ

この章では、Uzumibiを使ったCloudflare Workersアプリケーションの基本的な開発フローを一通り体験しました。

### 学んだこと

1. **uzumibi-cliのインストール**: Rustツールチェイン、Node.js/pnpm、Wranglerなどの前提ツールを準備し、`cargo install uzumibi-cli` でCLIをインストールしました。

2. **プロジェクトの作成**: `uzumibi new --template cloudflare` コマンドでプロジェクトを生成し、ディレクトリ構成と各ファイルの役割を確認しました。

3. **アプリケーションの実装**:
   - `public/index.html` に静的なフロントエンドページを作成しました。
   - `lib/app.rb` にRubyでAPIエンドポイントを実装しました。
   - `Uzumibi::Router` の基本的なDSL（`get`、`post`、動的パラメータなど）を学びました。

4. **開発サーバー**: `pnpm run dev` でローカル開発サーバーを起動し、ブラウザやcurlで動作を確認しました。

5. **デプロイ**: `pnpm run deploy` でCloudflare Workersにデプロイし、公開URLでアプリケーションを動作させました。

----

次の章以降では、Uzumibi on Cloudflare Workersで利用できる外部サービス連携の機能（Fetch、Durable Object、Queue）について解説します。
