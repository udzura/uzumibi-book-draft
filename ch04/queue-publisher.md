## 送信側の実装

送信側のWorkerは、HTTPリクエストを受けてCloudflare Queueにメッセージを送信します。

### プロジェクトの作成

```bash
uzumibi new --template cloudflare --features enable-external queue-publisher
cd queue-publisher
pnpm install
```

### wrangler.jsoncの編集

生成された `wrangler.jsonc` を開き、Queueのプロデューサー設定のコメントを解除して有効にします。

```jsonc
{
    "$schema": "node_modules/wrangler/config-schema.json",
    "name": "queue-publisher",
    "main": "src/index.js",
    "compatibility_date": "2025-12-30",
    "observability": {
        "enabled": true
    },
    "assets": {
        "directory": "./public",
        "binding": "ASSETS"
    },
    "queues": {
        "producers": [
            {
                "binding": "UZUMIBI_QUEUE",
                "queue": "my-app-queue"
            }
        ]
    },
    "durable_objects": {
        "bindings": [
            {
                "name": "UZUMIBI_KV_DATA",
                "class_name": "UzumibiKVObject"
            }
        ]
    },
    "migrations": [
        {
            "tag": "v1",
            "new_sqlite_classes": [
                "UzumibiKVObject"
            ]
        }
    ]
}
```

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
      res.body = "{\"error\": \"message body is required\"}"
    else
      debug_console("Sending message to queue: #{message}")
      Uzumibi::Queue.send("UZUMIBI_QUEUE", message)

      res.status_code = 202
      res.headers = {
        "content-type" => "application/json",
      }
      res.body = "{\"status\": \"accepted\", \"message\": \"#{message}\"}"
    end
    res
  end

  # タスクをキューに投入する例
  post "/api/tasks/:task_type" do |req, res|
    task_type = req.params[:task_type]
    payload = req.body

    # タスクタイプとペイロードをJSON風のフォーマットで送信
    queue_message = "{\"type\": \"#{task_type}\", \"payload\": \"#{payload}\"}"
    debug_console("Enqueuing task: #{queue_message}")
    Uzumibi::Queue.send("UZUMIBI_QUEUE", queue_message)

    res.status_code = 202
    res.headers = {
      "content-type" => "application/json",
    }
    res.body = "{\"status\": \"queued\", \"task_type\": \"#{task_type}\"}"
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
$ curl -X POST -d "Hello, Queue!" http://localhost:8787/api/send
{"status": "accepted", "message": "Hello, Queue!"}

# タスクの投入
$ curl -X POST -d "user@example.com" http://localhost:8787/api/tasks/send_email
{"status": "queued", "task_type": "send_email"}
```

ローカル開発時には、キューのメッセージは実際にはCloudflareのインフラに送信されます（ローカル環境でもキューの動作テストは可能です）。Wranglerのコンソールにデバッグメッセージが表示されます。

```
[debug]: Sending message to queue: Hello, Queue!
[debug]: Enqueuing task: {"type": "send_email", "payload": "user@example.com"}
```
