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

デプロイが完了すると、以下のようなメッセージが表示されます。

```
 ⛅️ wrangler 4.54.0
-------------------

Total Upload: 1024.00 KiB / gzip: 512.00 KiB
Worker Startup Time: 15 ms
Uploaded hello-uzumibi (3.00 sec)
Deployed hello-uzumibi triggers (1.00 sec)
  https://hello-uzumibi.<your-subdomain>.workers.dev
Current Version ID: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
```

表示されたURLにアクセスすると、デプロイされたアプリケーションが動作していることを確認できます。

### 動作確認

```bash
# デプロイされたアプリケーションにアクセス
$ curl https://hello-uzumibi.<your-subdomain>.workers.dev/api/hello
Hello from Uzumibi! Running on mruby/edge 3.2.0

$ curl https://hello-uzumibi.<your-subdomain>.workers.dev/api/greet/CloudflareWorkers
{"message": "Hello, CloudflareWorkers!"}
```

ブラウザで `https://hello-uzumibi.<your-subdomain>.workers.dev/` にアクセスすれば、先ほど作成したHTMLページも表示されます。

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
