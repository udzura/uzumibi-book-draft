## Uzumibiのサポートするプラットフォーム

Uzumibiは複数のエッジコンピューティングプラットフォームに対応しています。`uzumibi new` コマンドでプロジェクトを作成する際に、 `--template` オプションでターゲットプラットフォームを選択します。

### 対応プラットフォーム一覧

| プラットフォーム | テンプレート名 | ステータス | WASMターゲット |
|---------------|------------|---------|-------------|
| Cloudflare Workers | `cloudflare` | Stable | `wasm32-unknown-unknown` |
| Fastly Compute@Edge | `fastly` | Stable | `wasm32-wasip1` |
| Spin (Fermyon Cloud) | `spin` | Stable | `wasm32-wasip1` |
| Google Cloud Run | `cloudrun` | Experimental | ネイティブ（コンテナ） |
| Service Worker | `serviceworker` | Experimental | `wasm32-unknown-unknown` |
| Web Worker | `webworker` | Experimental | `wasm32-unknown-unknown` |

### Cloudflare Workers

Cloudflare Workersは、Cloudflareのグローバルネットワーク上でサーバーレスコードを実行するプラットフォームです。Uzumibiでは最も開発が進んでおり、以下の機能をサポートしています。

- 基本的なHTTPリクエスト/レスポンス処理
- 静的アセットの配信（`public/` ディレクトリ）
- 外部HTTPリクエスト（`Uzumibi::Fetch`）
- Durable Objectによるデータ永続化（`Uzumibi::KV`）
- Cloudflare Queuesによる非同期メッセージング（`Uzumibi::Queue`）

開発にはNode.js（pnpm）とWrangler CLIを使用します。

### Fastly Compute@Edge

Fastly Compute@Edgeは、FastlyのCDNエッジノード上でWASMアプリケーションを実行するプラットフォームです。Rustで書かれたグルーコードを使ってリクエスト処理を行います。

開発にはFastly CLIとRustツールチェインを使用します。

### Spin (Fermyon Cloud)

Spinは、Fermyon社が開発するWebAssemblyマイクロサービスフレームワークです。Fermyon Cloudにデプロイしてサーバーレスアプリケーションとして実行できます。

開発にはSpin CLIとRustツールチェインを使用します。

### Google Cloud Run（Experimental）

Google Cloud Runは、コンテナベースのサーバーレスプラットフォームです。Uzumibiでは、Tokio + Hyperを使ったHTTPサーバーとしてコンテナにパッケージングされ、Cloud Run上で実行されます。

他のプラットフォームとは異なり、WASMではなくネイティブバイナリとしてビルドされます。Dockerfileが自動生成されます。

### Service Worker / Web Worker（Experimental）

ブラウザのService Worker APIやWeb Worker APIを使って、クライアントサイドでUzumibiアプリケーションを実行するための実験的なテンプレートです。

### 本書の対象プラットフォーム

本書では、最も機能が充実しており実用的な **Cloudflare Workers** を対象に解説を進めます。Uzumibiのルーティングやアプリケーションコード（`lib/app.rb`）自体はプラットフォーム非依存であるため、他のプラットフォームでも同様のコードで動作しますが、外部サービス連携の部分はプラットフォームごとに異なります。
