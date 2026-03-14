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

ブラウザで `http://localhost:8787` にアクセスすると、`public/index.html` が表示されます。「APIを呼び出す」ボタンをクリックすると、`/api/hello` エンドポイントからのレスポンスが表示されます。

#### curlでの確認

APIエンドポイントをcurlで呼び出してみましょう。

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
