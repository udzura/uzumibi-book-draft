# Cloudflare Queueでやり取りをする

Cloudflare Queuesは、メッセージキューサービスです。プロデューサー（送信側）がキューにメッセージを送信し、コンシューマー（受信側）がメッセージを非同期に処理するという、メッセージキューイングの基本的なパターンを提供します。

## Cloudflare Queueの具体的な利用シーン

### Publisher

Publisher（プロデューサー）は、キューにメッセージを送信する役割です。Uzumibiでは `Uzumibi::Queue.send` メソッドを使ってメッセージを送信します。

典型的な利用シーンとしては以下のようなものがあります。

- **非同期のバックグラウンド処理**: ユーザーのリクエストに対して即座にレスポンスを返しつつ、時間のかかる処理（メール送信、画像変換、データ集計など）をキューに委譲する
- **イベント通知**: あるWorkerで発生したイベント（ユーザー登録、注文完了など）を、別のWorkerに通知して後続処理を実行する
- **レート制限の緩和**: 外部APIへのリクエストを一度キューに入れ、コンシューマー側で制御しながら順次処理する

```ruby
# 例: ユーザー登録後に非同期で歓迎メールを送信する
post "/api/users" do |req, res|
  # ユーザー登録処理（即座に完了）
  # ...

  # メール送信をキューに委譲（非同期）
  Uzumibi::Queue.send("UZUMIBI_QUEUE", "welcome_email:#{user_email}")

  res.status_code = 201
  res.body = JSON.generate({status: "registered"})
  res
end
```

### Consumer

Consumer（コンシューマー）は、キューからメッセージを受信して処理する役割です。

:::message
本来、Cloudflareのプロジェクトでは、Cloudflare Queueの送信をするWebアプリと、キューからメッセージを受信する処理は、同じWorkerのファイルで定義できます。
ただし、Uzumibi 0.6 系ではコンシューマー専用アプリを**別途生成する必要**がありますので留意してください。
:::

コンシューマー専用アプリの中のRubyコードでは、 `Uzumibi::Consumer` クラスを継承したクラスを用意し、`on_receive` メソッドをオーバーライドして処理を記述します。

コンシューマーには以下の特徴があります。

- **バッチ処理**: メッセージはバッチ単位（最大10件、最大5秒待機）で受信される
- **リトライ機能**: 処理に失敗したメッセージを指定した遅延時間後にリトライできる
- **確認応答**: 処理が完了したメッセージは `ack!` で確認応答を送り、キューから削除する

```ruby
class Consumer < Uzumibi::Consumer
  def on_receive(message)
    debug_console("Processing: #{message.body}")

    # 処理が成功したら確認応答
    message.ack!
  end
end
```

Messageオブジェクトは以下の属性を持ちます。

| 属性 | 型 | 説明 |
|-----|-----|------|
| `message.id` | String | メッセージのユニークID |
| `message.timestamp` | String | メッセージの送信タイムスタンプ（ISO 8601形式） |
| `message.body` | String | メッセージ本文 |
| `message.attempts` | Integer | これまでの配信試行回数 |

また、Messageオブジェクトには以下のメソッドがあります。

| メソッド | 説明 |
|---------|------|
| `message.ack!` | メッセージの処理完了を通知し、キューから削除する |
| `message.retry(delay_seconds: N)` | N秒後にメッセージを再配信する |

## Cloudflare Queueを使ったメッセージのやり取りの概要

Cloudflare Queuesを使ったUzumibiアプリケーションでは、送信側（Publisher）と受信側（Consumer）の2つのWorkerを構成します。ここでは全体のアーキテクチャとセットアップの流れを解説します。

### アーキテクチャ

```
クライアント
    ↓ HTTPリクエスト
[Publisher Worker] (enable-external featureで生成)
    ↓ Uzumibi::Queue.send
[Cloudflare Queue]
    ↓ メッセージ配信
[Consumer Worker] (queue featureで生成)
    ↓ on_receive(message)
    メッセージ処理 → ack! or retry
```

送信側と受信側は同一のキューに対してバインディングを設定した別々のWorkerとして動作します。送信側は `queues.producers` バインディングを通じてメッセージを投入し、受信側は `queues.consumers` バインディングを通じてメッセージを受信します。

### Cloudflare Queueの作成

まず、Cloudflare Queuesにキューを作成する必要があります。Cloudflareの管理画面化、Wrangler CLIを使って作成します。

```bash
$ npx wrangler queues create my-app-queue
...
 ⛅️ wrangler 4.73.0
───────────────────
🌀 Creating queue 'my-app-queue'
✅ Created queue 'my-app-queue'
```

キューが作成されたことを確認します。

```bash
$ npx wrangler queues list
┌──────────────────────────────────┬───────────────┬─────────────────────────────┬─────────────────────────────┬───────────┬───────────┐
│ id                               │ name          │ created_on                  │ modified_on                 │ producers │ consumers │
├──────────────────────────────────┼───────────────┼─────────────────────────────┼─────────────────────────────┼───────────┼───────────┤
│ 7941fbb9762d4a02b1a1c644XXXXXXXX │ my-app-queue  │ 2026-03-14T12:48:00.918834Z │ 2026-03-14T12:48:00.918834Z │ 0         │ 0         │
└──────────────────────────────────┴───────────────┴─────────────────────────────┴─────────────────────────────┴───────────┴───────────┘
```

### プロジェクト構成

Queueのやり取りには2つのプロジェクトが必要です。

1. **送信側プロジェクト**: `enable-external` featureで作成し、`Uzumibi::Queue.send` でメッセージを送信する
2. **受信側プロジェクト**: `queue` featureで作成し、`Uzumibi::Consumer` でメッセージを受信・処理する

```bash
# 送信側
uzumibi new --template cloudflare --features enable-external queue-publisher
# 受信側
uzumibi new --template cloudflare --features queue queue-consumer
```

:::message
`queue` feature が有効の時は自動的に `enable-external` featureも有効の状態でプロジェクトが作られます。
:::

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
},
// ...snip
```

`binding` はコード内でキューを参照する際の名前で、`Uzumibi::Queue.send` の第1引数に対応します。`queue` は先ほどCloudflare Queuesに作成した実際のキュー名を指定してください。

#### 受信側の設定

受信側の `wrangler.jsonc` では、`queues.consumers` にキューバインディングがプロジェクト名から自動設定されています。今回は、正しい名前に手動で変更してください。

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

必要に応じて以下の値も設定してください。

| 設定項目 | 説明 |
|---------|------|
| `queue` | 受信するキュー名 |
| `max_batch_size` | 一度に受信するメッセージの最大数（デフォルト: 10） |
| `max_batch_timeout` | バッチがいっぱいになるまでの最大待機秒数（デフォルト: 5） |

## 送信側の実装

送信側のWorker（queue-publisher）は、HTTPリクエストを受けてCloudflare Queueにメッセージを送信します。

### アプリケーションの実装

`lib/app.rb` を以下のように編集します。

```ruby
class App < Uzumibi::Router
  # メッセージ送信API
  post "/api/send" do |req, res|
    message = req.body

    if message == "" || message == nil
      res.status_code = 400
      res.headers = {
        "content-type" => "application/json",
      }
      res.body = JSON.generate({ error: "message body is required" })
    else
      debug_console("Sending message to queue: #{message}")
      Uzumibi::Queue.send("UZUMIBI_QUEUE", message)

      res.status_code = 202
      res.headers = {
        "content-type" => "application/json",
      }
      res.body = JSON.generate({ status: "accepted", message: message })
    end
    res
  end

  # タスクをキューに投入する例
  post "/api/tasks/:task_type" do |req, res|
    task_type = req.params[:task_type]
    payload = req.body

    # タスクタイプとペイロードをJSONフォーマットで送信
    queue_message = JSON.generate({ type: task_type, payload: payload })
    debug_console("Enqueuing task: #{queue_message}")
    Uzumibi::Queue.send("UZUMIBI_QUEUE", queue_message)

    res.status_code = 202
    res.headers = {
      "content-type" => "application/json",
    }
    res.body = JSON.generate({ status: "queued", task_type: task_type })
    res
  end
end

$APP = App.new
```

#### コードの解説

**メッセージの送信**

```ruby
Uzumibi::Queue.send("UZUMIBI_QUEUE", message)
```

`Uzumibi::Queue.send` の第1引数には `wrangler.jsonc` の `queues.producers` で設定した `binding` 名を指定します。第2引数にはメッセージ本文（文字列）を渡します。

**HTTPステータスコード 202**

キューへのメッセージ投入は非同期処理の開始を意味するため、HTTPステータスコードとして `202 Accepted`（受理済み）を返しています。メッセージの実際の処理はコンシューマーWorkerが行います。

### 動作確認

開発サーバーを起動して、メッセージ送信をテストします。

```bash
pnpm run dev
```

```bash
# メッセージの送信
$ curl -X POST -d "Hello, Queue\!" http://localhost:8787/api/send
{"status": "accepted", "message": "Hello, Queue!"}

# タスクの投入
$ curl -X POST -d "user@example.com" \
    http://localhost:8787/api/tasks/send_email
{"status": "queued", "task_type": "send_email"}
```

ローカル開発時には、キューのメッセージは実際にはlocalhostのダミーサバーに送られます。Wranglerのコンソールにデバッグメッセージが表示されます。

```
[debug]: Sending message to queue: Hello, Queue!
[debug]: Enqueuing task: {"type": "send_email", "payload": "user@example.com"}
```

### 連携確認のため一度デプロイをする

簡単な動作確認の後で、以下のコマンドで一度リモートの Cloudflare Workerにデプロイしておきましょう。

```bash
pnpm run deploy
```

デプロイ後、簡単に動作確認しておきましょう。

```bash
curl -X POST -d "Hello, Production Queue\!" \
    http://queue-publisher.<ID>.workers.dev/api/send
{"message":"Hello, Production Queue!","status":"accepted"}
```

## 受信側の実装

受信側のWorkerは、Cloudflare Queueからメッセージを受信し、処理します。Uzumibiでは `Uzumibi::Consumer` クラスを継承して、`on_receive` メソッドに処理を記述します。

### プロジェクトの構成の確認

`queue` featureを有効にすると、通常のプロジェクトの `lib/app.rb` の代わりに以下のファイルが生成されます。

- `lib/consumer.rb` - コンシューマーのRubyコード

また、`wasm-app/src/lib.rs` にはコンシューマー用のWasmエクスポート関数（`uzumibi_initialize_message`、`uzumibi_start_message`）が追加されます。

```
queue-consumer/
├── lib/
│   └── consumer.rb     # Queueコンシューマー処理
├── src/
│   └── index.js        # JS グルーコード（HTTPとQueue両方を処理）
├── wasm-app/
│   ├── build.rs        # app.rb と consumer.rb の両方をコンパイル
│   └── src/
│       └── lib.rs      # Wasm（HTTP処理 + Queue処理の両方をエクスポート）
├── wrangler.jsonc
└── package.json
```

受信側のWorkerQueueメッセージの処理を専任で処理する形になります。

### wrangler.jsoncの確認

受信側の `wrangler.jsonc` には、`queues.consumers` の設定が含まれています。キュー名を送信側と合っているか再確認してください。

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

コンシューマーのインスタンスはグローバル変数 `$CONSUMER` に代入します。Wasm側のRustコードから参照されるため、必ず設定する必要があります。

### 確認のためのデプロイ

:::message
現在のWranglerでは、複数のプロジェクトを跨いだQueueのやりとりについては、localhostの開発環境では検証できません。
:::

検証可能なように、queue-consumerを一度デプロイします。

まずビルドを行います。

```bash
pnpm install
pnpm run dev
# HTTP部分は空っぽなので、起動はするが動作は確認できない。
# Wasmのビルドをするためだけなので停止する。
```

### 動作確認

#### テスト

受信のログを確認するために、以下のコマンドでターミナルにログを表示しましょう。

ターミナル（queue-consumer）:
```bash
$ cd queue-consumer
$ npx wrangler tail

 ⛅️ wrangler 4.73.0
───────────────────
Successfully created tail, expires at 2026-03-14T19:19:39Z
```

メッセージ送信のために別のターミナルで以下の curl コマンドを発行します。

```bash
curl -X POST -d "Test message from publisher" \
    http://queue-publisher.<ID>.workers.dev/api/send
{"message":"Test message from publisher","status":"accepted"}
```

queue-consumerのコンソールに、メッセージの処理ログが表示されます。

```
Queue my-app-queue (1 message) - Ok @ 2026/3/14 22:22:05
  (log) [debug]: [Consumer] Received message: id=25f4ef4ca8b2fdb7055cc5b9XXXXXXXX, body=Test message from publisher, attempts=1
  (log) [debug]: [Consumer] Processing: Test message from publisher
  (log) [debug]: [Consumer] Message content: Test message from publisher
  (log) [debug]: [Consumer] Successfully processed message 25f4ef4ca8b2fdb7055cc5b9XXXXXXXX
```

処理状況はCloudflareダッシュボードの「Workers & Pages」 > 対象Worker > 「Logs」からも確認できます。

## この章のまとめ

Queue を用いることで、Cloudflare Workers + Uzumibiで手軽に非同期処理が実現できます。

Queue を使う場合、アーキテクチャが少し特殊になります。Uzumibiの場合2つのWorkerが必要になりますので留意してください。