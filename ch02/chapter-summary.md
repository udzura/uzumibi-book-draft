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

### 開発のワークフロー

Uzumibiでの基本的な開発ワークフローをまとめると以下のようになります。

```
1. uzumibi new でプロジェクト作成
2. pnpm install で依存関係インストール
3. lib/app.rb を編集してルートとロジックを実装
4. public/ にHTML/CSS/JSなどの静的ファイルを配置
5. pnpm run dev で開発サーバーを起動して動作確認
6. pnpm run deploy でCloudflare Workersにデプロイ
```

### 次の章に向けて

ここまでは、自己完結するシンプルなAPIを実装しました。しかし、実際のアプリケーションでは外部のAPIを呼び出したり、データを永続的に保存したりする必要があります。

次の章では、Uzumibi on Cloudflare Workersで利用できる外部サービス連携の機能（Fetch、Durable Object、Queue）について解説します。
