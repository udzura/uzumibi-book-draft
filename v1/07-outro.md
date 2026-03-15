# 終わりに: Uzumibiの参考情報

本章では、本編では収録できなかった補足的な情報を掲載します。

## Uzumibiのサポートするプラットフォーム

Uzumibiは複数のエッジコンピューティングプラットフォームに対応しています。`uzumibi new` コマンドでプロジェクトを作成する際に、 `--template` オプションでターゲットプラットフォームを選択します。

### 対応プラットフォーム一覧

| プラットフォーム | テンプレート名 | ステータス | WASMターゲット |
|---------------|------------|---------|-------------|
| Cloudflare Workers | `cloudflare` | Beta | `wasm32-unknown-unknown` |
| Fastly Compute@Edge | `fastly` | Experimental | `wasm32-wasip1` |
| Spin (Fermyon Cloud) | `spin` | Experimental | `wasm32-wasip1` |
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

開発にはNode.js（pnpm）とWrangler CLI、Rustツールチェインを使用します。

### Fastly Compute@Edge

Fastly Compute@Edgeは、FastlyのCDNエッジノード上でWASMアプリケーションを実行するプラットフォームです。Rustで書かれたグルーコードを使ってリクエスト処理を行います。

開発にはFastly CLIとRustツールチェインを使用します。

### Spin (Fermyon Cloud)

Spinは、Fermyon社が開発するWebAssemblyマイクロサービスフレームワークです。Fermyon Cloudにデプロイしてサーバーレスアプリケーションとして実行できます。

開発にはSpin CLIとRustツールチェインを使用します。

### Google Cloud Run（Experimental）

Google Cloud Runは、コンテナベースのサーバーレスプラットフォームです。Uzumibiでは、Tokio + Hyperを使ったHTTPサーバーとしてコンテナにパッケージングされ、Cloud Run上で実行可能な状態になります。

他のプラットフォームとは異なり、Wasmではなく**ネイティブバイナリ**としてビルドされます。Dockerfileも自動で添付しますので、それをもとにコンテナを作成できます。

ローカルホストでも実行可能な形でプロジェクトが作成されるので、開発自体は、Dockerがインストールされていれば可能です。Rustツールチェインもあれば便利でしょう。

### Service Worker / Web Worker（Experimental）

ブラウザのService Worker APIやWeb Worker APIを使って、クライアントサイドでUzumibiアプリケーションを実行するための実験的なテンプレートです。

Wasmの作成にRustツールチェインが必要です。

### 免責事項

:::message
各プラットフォーム、ソフトウェアの名称、商標はそれぞれの運営企業に所属します。本書では、紹介目的でそれらの知的資産を引用しています。
:::

## まだUzumibiではできないこと

### 複数ファイルのアプリケーション記述

現在のUzumibiでは、Rubyのアプリケーションコードは **単一ファイル** （`lib/app.rb` や `lib/consumer.rb`）に記述する必要があります。

通常のRuby開発では、`require` や `require_relative` を使ってコードを複数ファイルに分割し、モジュール化するのが一般的です。しかし、Uzumibiではmruby/edgeの制約により、これらのファイル読み込み機能は利用できません。現在のところは、1ファイルに全コードを記述する、ビルドスクリプトでファイルを結合する等の形で対応してください。

複数ファイルのサポートは、Uzumibiの今後の開発で優先的に取り組む予定の機能です。

### mruby/edge自体の言語機能の制限

mruby/edgeはmrubyのサブセットを実装しているため、CRuby（標準のRuby）やフルセットのmrubyと比べていくつかの制限があります。

- **一部の標準ライブラリ**: `File`、`IO`、`Socket` などのOSリソースに依存するライブラリは、WebAssembly環境では利用できません。
- **正規表現**: 正規表現（`Regexp`）は現時点では部分的なサポートにとどまります。
- **`require` / `load`**: 前述の通り、外部ファイルの読み込みはサポートされていません。
- **スレッド / Fiber**: 並行処理の機能はWebAssembly環境の制約により利用できません。

また、今後、プラットフォームごとに利用できるRubyの機能も変わる可能性が高いです。ドキュメントに可能な限り記述する予定です。

### プラットフォーム間の機能差

:::message
この項目は随時更新予定です。最終更新日は 2026/03/14 です。
:::

Uzumibiは複数のプラットフォームに対応していますが、外部サービス連携の機能はプラットフォームごとに異なります。

| 機能 | Cloudflare | Fastly | Spin | Cloud Run |
|------|-----------|--------|------|-----------|
| 基本的なHTTP処理 | o | o | o | o |
| 外部HTTPリクエスト（Fetch） | o | - | - | - |
| Key-Valueストア | o (Durable Object) | - | - | - |
| メッセージキュー | o (Queues) | - | - | - |
| 静的アセット配信 | o | - | - | - |

現時点では、Cloudflare Workersが最も多くの機能をサポートしています。他のプラットフォームでも、今後順次対応が進む予定です。
