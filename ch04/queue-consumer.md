## 受信側の実装

受信側のWorkerは、Cloudflare Queueからメッセージを受信し、処理します。Uzumibiでは `Uzumibi::Consumer` クラスを継承して、`on_receive` メソッドに処理を記述します。

### プロジェクトの作成

```bash
uzumibi new --template cloudflare --features enable-external,queue queue-consumer
cd queue-consumer
pnpm install
```

`queue` featureを有効にすると、通常のプロジェクトに加えて以下のファイルが生成されます。

- `lib/consumer.rb` - コンシューマーのRubyコード

また、`wasm-app/src/lib.rs` にはコンシューマー用のWASMエクスポート関数（`uzumibi_initialize_message`、`uzumibi_start_message`）が追加されます。

### プロジェクト構成

```
queue-consumer/
├── lib/
│   ├── app.rb          # HTTPリクエスト処理（通常のWorker）
│   └── consumer.rb     # Queueコンシューマー処理
├── src/
│   └── index.js        # JS グルーコード（HTTPとQueue両方を処理）
├── wasm-app/
│   ├── build.rs        # app.rb と consumer.rb の両方をコンパイル
│   └── src/
│       └── lib.rs      # WASM（HTTP処理 + Queue処理の両方をエクスポート）
├── wrangler.jsonc
└── package.json
```

受信側のWorkerは、HTTPリクエストの処理とQueueメッセージの処理を1つのWorkerで兼ねることができます。

### wrangler.jsoncの確認

受信側の `wrangler.jsonc` には、`queues.consumers` の設定が含まれています。キュー名を送信側と合わせます。

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

### コンシューマーの実装

`lib/consumer.rb` を編集して、メッセージの処理ロジックを記述します。

```ruby
class Consumer < Uzumibi::Consumer
  # @rbs message: Uzumibi::Message
  def on_receive(message)
    debug_console("[Consumer] Received message: id=#{message.id}, body=#{message.body}, attempts=#{message.attempts}")

    # メッセージの処理
    body = message.body
    debug_console("[Consumer] Processing: #{body}")

    # 処理に成功したら確認応答
    if message.attempts > 3
      # 3回以上リトライしても処理できなかった場合は諦めてackする
      debug_console("[Consumer] Giving up after #{message.attempts} attempts, acknowledging message #{message.id}")
      message.ack!
    else
      # 通常の処理
      begin
        process_message(body)
        debug_console("[Consumer] Successfully processed message #{message.id}")
        message.ack!
      rescue => e
        debug_console("[Consumer] Error processing message: retrying in 5 seconds")
        message.retry(delay_seconds: 5)
      end
    end
  end

  def process_message(body)
    # 実際のメッセージ処理ロジック
    debug_console("[Consumer] Message content: #{body}")
    # ここに具体的な処理を記述
    # 例: 外部APIの呼び出し、データの保存など
  end
end

$CONSUMER = Consumer.new
```

#### コードの解説

**`Uzumibi::Consumer` の継承**

```ruby
class Consumer < Uzumibi::Consumer
  def on_receive(message)
    # ...
  end
end
```

`Uzumibi::Consumer` を継承し、`on_receive` メソッドをオーバーライドします。このメソッドはキューからメッセージを受信するたびに呼ばれます。

**メッセージの属性**

`on_receive` に渡される `message` オブジェクト（`Uzumibi::Message`）からは、以下の情報を取得できます。

```ruby
message.id        # メッセージID（文字列）
message.body      # メッセージ本文（文字列）
message.attempts  # 配信試行回数（整数）
message.timestamp # 送信タイムスタンプ（ISO 8601形式の文字列）
```

**確認応答（ack!）**

```ruby
message.ack!
```

処理が完了したメッセージには `ack!` を呼んで確認応答を送ります。`ack!` が呼ばれたメッセージはキューから削除されます。

**リトライ**

```ruby
message.retry(delay_seconds: 5)
```

処理に失敗した場合は `retry` を呼んで、メッセージを再配信できます。`delay_seconds` で再配信までの遅延時間を秒単位で指定します。次に `on_receive` が呼ばれるときには `message.attempts` がインクリメントされています。

**グローバル変数 `$CONSUMER`**

```ruby
$CONSUMER = Consumer.new
```

コンシューマーのインスタンスはグローバル変数 `$CONSUMER` に代入します。WASM側のRustコードから参照されるため、必ず設定する必要があります。

### 動作確認

#### ローカルでのテスト

送信側と受信側の両方の開発サーバーを起動してテストします。

ターミナル1（受信側）:
```bash
cd queue-consumer
pnpm run dev
```

ターミナル2（送信側）:
```bash
cd queue-publisher
pnpm run dev
```

ターミナル3（メッセージ送信）:
```bash
curl -X POST -d "Test message from publisher" http://localhost:8787/api/send
```

受信側のコンソールに、メッセージの処理ログが表示されます。

```
[debug]: [Consumer] Received message: id=xxx, body=Test message from publisher, attempts=1
[debug]: [Consumer] Processing: Test message from publisher
[debug]: [Consumer] Message content: Test message from publisher
[debug]: [Consumer] Successfully processed message xxx
```

#### 本番環境へのデプロイ

まず、Cloudflare Queueを作成します（まだ作成していない場合）。

```bash
npx wrangler queues create my-app-queue
```

送信側と受信側の両方をデプロイします。

```bash
# 受信側をデプロイ
cd queue-consumer
pnpm run deploy

# 送信側をデプロイ
cd queue-publisher
pnpm run deploy
```

デプロイ後、送信側のURLにメッセージをPOSTすると、受信側のWorkerが非同期でメッセージを処理します。処理状況はCloudflareダッシュボードの「Workers & Pages」 > 対象Worker > 「Logs」から確認できます。
