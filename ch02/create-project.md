## プロジェクトの作成

`uzumibi new` コマンドを使って、Cloudflare Workers向けのプロジェクトを作成します。

### プロジェクトの生成

以下のコマンドで `hello-uzumibi` という名前のプロジェクトを作成します。

```bash
uzumibi new --template cloudflare hello-uzumibi
```

実行すると、プロジェクトのディレクトリが作成され、必要なファイルが生成されます。

```
Created: hello-uzumibi/Cargo.toml
Created: hello-uzumibi/package.json
Created: hello-uzumibi/wrangler.jsonc
Created: hello-uzumibi/vitest.config.js
Created: hello-uzumibi/lib/app.rb
Created: hello-uzumibi/public/assets/.gitkeep
Created: hello-uzumibi/src/index.js
Created: hello-uzumibi/wasm-app/Cargo.toml
Created: hello-uzumibi/wasm-app/build.rs
Created: hello-uzumibi/wasm-app/src/lib.rs
Created: hello-uzumibi/wasm-app/.cargo/config.toml
```

### uzumibi new コマンドのオプション

`uzumibi new` コマンドには以下のオプションがあります。

| オプション | 説明 |
|-----------|------|
| `-t, --template <TEMPLATE>` | 使用するプラットフォームテンプレート（必須） |
| `-d, --dest_dir <DIR>` | 出力先ディレクトリ（省略時はプロジェクト名と同じ） |
| `-f, --force` | 既存ファイルを確認なしで上書き |
| `--features <FEATURES>` | 有効にする機能（カンマ区切り） |

`--features` で指定できる機能は以下の通りです。

| 機能名 | 説明 |
|-------|------|
| `enable-external` | 外部サービス連携（Fetch, KV, Queue）を有効にする |
| `queue` | Cloudflare Queuesのコンシューマー処理を有効にする |

例えば、外部サービス連携付きのプロジェクトを作成する場合は以下のようにします。

```bash
uzumibi new --template cloudflare --features enable-external my-api-app
```

### プロジェクトのディレクトリ構成

生成されたプロジェクトは以下のような構成になっています。

```
hello-uzumibi/
├── Cargo.toml          # Rustワークスペースの設定
├── package.json        # Node.jsの依存関係とスクリプト
├── wrangler.jsonc      # Wrangler（Cloudflare Workers CLI）の設定
├── vitest.config.js    # テスト設定
├── lib/
│   └── app.rb          # Rubyアプリケーションコード（メイン）
├── public/
│   └── assets/         # 静的アセット（HTML, CSS, 画像など）
├── src/
│   └── index.js        # JavaScriptグルーコード（エントリーポイント）
└── wasm-app/
    ├── Cargo.toml      # WASMクレートの設定
    ├── build.rs        # ビルドスクリプト（Rubyコードのコンパイル）
    ├── src/
    │   └── lib.rs      # WASMモジュールのRustコード
    └── .cargo/
        └── config.toml # Cargoのターゲット設定
```

### 各ファイルの役割

#### `lib/app.rb`

**開発者が主に編集するファイル**です。Uzumibiのルーティングとリクエスト処理のロジックをRubyで記述します。

#### `src/index.js`

Cloudflare Workersのエントリーポイントです。HTTPリクエストを受け取り、バイナリ形式にシリアライズしてWASMモジュールに渡し、レスポンスをデシリアライズしてクライアントに返却します。通常、このファイルを編集する必要はありません。

#### `wasm-app/`

Rubyコードをmrubyバイトコードにコンパイルし、WASMモジュールとしてパッケージングするためのRustクレートです。

- `build.rs` はビルド時に `lib/app.rb` を mrubyバイトコード（`.mrb`）にコンパイルします。
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
