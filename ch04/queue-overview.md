## Cloudflare Queueを使ったメッセージのやり取りの概要

Cloudflare Queuesを使ったUzumibiアプリケーションでは、送信側（Publisher）と受信側（Consumer）のWorkerを構成します。ここでは全体のアーキテクチャとセットアップの流れを解説します。

### アーキテクチャ

```
クライアント
    ↓ HTTPリクエスト
[Publisher Worker] (enable-external feature)
    ↓ Uzumibi::Queue.send
[Cloudflare Queue]
    ↓ メッセージ配信
[Consumer Worker] (queue feature)
    ↓ on_receive(message)
    メッセージ処理 → ack! or retry
```

送信側と受信側は別々のWorkerとして動作します。同一のキューに対して、送信側は `queues.producers` バインディングを通じてメッセージを投入し、受信側は `queues.consumers` バインディングを通じてメッセージを受信します。

### Cloudflare Queueの作成

まず、Cloudflare Queuesにキューを作成する必要があります。Wrangler CLIを使って作成します。

```bash
npx wrangler queues create my-app-queue
```

キューが作成されたことを確認します。

```bash
npx wrangler queues list
```

### プロジェクト構成

Queueのやり取りには2つのプロジェクトが必要です。

1. **送信側プロジェクト**: `enable-external` featureで作成し、`Uzumibi::Queue.send` でメッセージを送信する
2. **受信側プロジェクト**: `enable-external,queue` featureで作成し、`Uzumibi::Consumer` でメッセージを受信・処理する

```bash
# 送信側
uzumibi new --template cloudflare --features enable-external queue-publisher
# 受信側
uzumibi new --template cloudflare --features enable-external,queue queue-consumer
```

### wrangler.jsoncの設定

#### 送信側の設定

送信側の `wrangler.jsonc` では、`queues.producers` にキューバインディングを設定します。デフォルトのテンプレートではコメントアウトされているため、コメントを解除して有効にします。

```jsonc
{
    "queues": {
        "producers": [
            {
                "binding": "UZUMIBI_QUEUE",
                "queue": "my-app-queue"
            }
        ]
    }
}
```

`binding` はコード内でキューを参照する際の名前で、`Uzumibi::Queue.send` の第1引数に対応します。`queue` はCloudflare Queuesに作成した実際のキュー名です。

#### 受信側の設定

受信側の `wrangler.jsonc` では、`queues.consumers` にキューバインディングが自動設定されています。

```jsonc
{
    "queues": {
        "producers": [
            {
                "binding": "UZUMIBI_QUEUE",
                "queue": "my-app-queue"
            }
        ],
        "consumers": [
            {
                "queue": "my-app-queue",
                "max_batch_size": 10,
                "max_batch_timeout": 5
            }
        ]
    }
}
```

| 設定項目 | 説明 |
|---------|------|
| `queue` | 受信するキュー名 |
| `max_batch_size` | 一度に受信するメッセージの最大数（デフォルト: 10） |
| `max_batch_timeout` | バッチがいっぱいになるまでの最大待機秒数（デフォルト: 5） |

次の節では、送信側と受信側の具体的な実装を見ていきます。
