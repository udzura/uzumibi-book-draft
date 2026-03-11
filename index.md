# 入門 Uzumibi on Cloudflare Workers

# はじめに

## [mruby/edgeとは](ch01/mrubyedge-toha.md)

## [Uzumibiとは](ch01/uzumibi-toha.md)

## [Uzumibiのサポートするプラットフォーム](ch01/supported-platforms.md)

# プロジェクトの作成とhello world

## [uzumibi-cliのインストール](ch02/install-cli.md)

## [プロジェクトの作成](ch02/create-project.md)

## [hello worldアプリケーションの実装](ch02/hello-world.md)

### フロントの実装

### APIの実装

## [devサーバーの立ち上げ](ch02/dev-server.md)

## [デプロイ](ch02/deploy.md)

## [この章のまとめ](ch02/chapter-summary.md)

# 外部サービスを使う

## [Uzumibi on Cloudflare Workersで対応する外部サービス](ch03/external-services-overview.md)

### Fetch

### Durable Object

### Queue

## [Fetchを使って外部の公開APIを呼び出す](ch03/fetch-api.md)

### プロジェクトの作成

### 利用するAPIの選定

### APIを呼び出す

### 動作確認

## [Durable Objectを使ってメモを保存する](ch03/durable-object.md)

### アーキテクチャの確認

### プロジェクトの作成

### 実装

### 動作確認

# Queueのやり取りをする

## [Cloudflare Queueの具体的な利用シーン](ch04/queue-use-cases.md)

### Publisher

### Consumer

## [Cloudflare Queueを使ったメッセージのやり取りの概要](ch04/queue-overview.md)

## [送信側の実装](ch04/queue-publisher.md)

### プロジェクトの作成

## [受信側の実装](ch04/queue-consumer.md)

# 終わりに: まだUzumibiではできないこと

## [複数ファイルのアプリケーション記述](ch05/multi-file.md)

## [まだサポートしていない機能](ch05/unsupported-features.md)

# [参考資料](references.md)
