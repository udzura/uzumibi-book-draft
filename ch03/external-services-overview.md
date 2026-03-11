# 外部サービスを使う

## Uzumibi on Cloudflare Workersで対応する外部サービス

Cloudflare Workersでは、Cloudflareが提供するさまざまなサービスにアクセスできます。Uzumibiでは、これらのサービスの一部をRubyから利用するためのAPIを提供しています。

外部サービス連携を利用するには、プロジェクト作成時に `--features enable-external` オプションを指定します。

```bash
uzumibi new --template cloudflare --features enable-external my-app
```

現在、Uzumibi on Cloudflare Workersでは以下の3つの外部サービスに対応しています。

### Fetch

`Uzumibi::Fetch` は、Worker内から外部のHTTPリクエストを発行するためのAPIです。外部のREST APIやWebサービスを呼び出す際に使用します。

```ruby
# 基本的な使い方
response = Uzumibi::Fetch.fetch("https://api.example.com/data")
# response.status_code => 200
# response.body => レスポンスボディ
# response.headers => レスポンスヘッダー（Hash）
```

`Uzumibi::Fetch.fetch` メソッドは以下の引数を取ります。

| 引数 | 型 | 説明 |
|-----|-----|------|
| `url` | String | リクエスト先のURL（必須） |
| `method` | String | HTTPメソッド（省略時は `"GET"`） |
| `body` | String | リクエストボディ（省略時は空文字列） |

戻り値は `Uzumibi::Response` オブジェクトで、`status_code`、`headers`、`body` を持ちます。

内部的には、JavaScriptの `fetch()` APIを通じて非同期HTTPリクエストを実行しています。WASMの非同期処理には `asyncify-wasm` ライブラリを使用しています。

### Durable Object

`Uzumibi::KV` は、Cloudflare Durable Objectを利用したKey-Valueストアです。データをリクエストをまたいで永続的に保存できます。

```ruby
# 値を保存
Uzumibi::KV.set("key", "value")

# 値を取得
value = Uzumibi::KV.get("key")
# => "value"

# 存在しないキー
value = Uzumibi::KV.get("unknown")
# => nil
```

Durable Objectは、Cloudflare Workersのステートフルなストレージ機能です。Uzumibiでは `UzumibiKVObject` という名前のDurable Objectクラスが自動的に設定され、SQLiteベースのストレージでデータを永続化します。

| メソッド | 引数 | 戻り値 | 説明 |
|---------|------|--------|------|
| `Uzumibi::KV.get(key)` | key: String | String or nil | キーに対応する値を取得 |
| `Uzumibi::KV.set(key, value)` | key: String, value: String | true | キーに値を保存 |

### Queue

`Uzumibi::Queue` は、Cloudflare Queuesにメッセージを送信するためのAPIです。非同期のバックグラウンド処理に利用できます。

```ruby
# メッセージの送信
Uzumibi::Queue.send("UZUMIBI_QUEUE", "Hello from queue!")
```

| メソッド | 引数 | 戻り値 | 説明 |
|---------|------|--------|------|
| `Uzumibi::Queue.send(queue_name, message)` | queue_name: String, message: String | true | 指定したキューにメッセージを送信 |

`queue_name` は `wrangler.jsonc` で定義したキューバインディングの名前です。Queueの受信側（Consumer）の実装については第4章で詳しく解説します。

### 外部サービス機能の内部動作

外部サービス連携が有効なプロジェクトでは、通常のプロジェクトと以下の点が異なります。

1. **WASMモジュール**: `wasm-app/Cargo.toml` に `enable-external` featureが追加され、Rust側にFetch/KV/Queueのラッパー関数が組み込まれます。
2. **JavaScriptグルーコード**: `src/index.js` が `asyncify-wasm` ライブラリを使用するバージョンに差し替わり、非同期処理（`await`）がWASM内で利用可能になります。
3. **wrangler.jsonc**: Durable Objectバインディングやマイグレーション設定が追加されます。
