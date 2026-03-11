# Queueのやり取りをする

## Cloudflare Queueの具体的な利用シーン

Cloudflare Queuesは、メッセージキューサービスです。プロデューサー（送信側）がキューにメッセージを送信し、コンシューマー（受信側）がメッセージを非同期に処理するという、メッセージキューイングの基本的なパターンを提供します。

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
  res.body = "{\"status\": \"registered\"}"
  res
end
```

### Consumer

Consumer（コンシューマー）は、キューからメッセージを受信して処理する役割です。Uzumibiでは `Uzumibi::Consumer` クラスを継承し、`on_receive` メソッドをオーバーライドして処理を記述します。

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

Messageオブジェクトには以下のメソッドがあります。

| メソッド | 説明 |
|---------|------|
| `message.ack!` | メッセージの処理完了を通知し、キューから削除する |
| `message.retry(delay_seconds: N)` | N秒後にメッセージを再配信する |
