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

容量が大きい場合Cloudflare Workersでは提供が難しい場合がありますが、Uzumibiとmruby/edgeのアプリケーション、できるだけ小さなサイズに収まるようにミニマルな機能提供をするよう設計されています。したがってエッジアプリケーションの開発の上で非常に有利と言えるでしょう。

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
