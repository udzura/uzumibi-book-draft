## Uzumibiとは

Uzumibi（うずみび）は、mruby/edgeを基盤とした、エッジコンピューティングプラットフォーム上でRubyによるWebアプリケーションを構築するための軽量フレームワークです。

「うずみび（埋み火）」とは、灰の中に炭火を埋めて火を絶やさないようにする日本の伝統的な技法のことです。小さくても確実に動くRubyの力を、エッジ環境に灯し続けるというコンセプトが名前に込められています。

### Uzumibiの概要

Uzumibiは、SinatraライクなDSLを使ってルーティングを定義できるWebアプリケーションフレームワークです。以下のようなRubyコードでHTTPリクエストを処理するアプリケーションを記述できます。

```ruby
class App < Uzumibi::Router
  get "/" do |req, res|
    res.status_code = 200
    res.headers = {
      "content-type" => "text/plain",
      "x-powered-by" => "#{RUBY_ENGINE} #{RUBY_VERSION}"
    }
    res.body = "It works!\n"
    res
  end

  get "/greet/to/:name" do |req, res|
    res.status_code = 200
    res.headers = {
      "content-type" => "text/plain",
    }
    res.body = "Hello, #{req.params[:name]}!!\n"
    res
  end
end

$APP = App.new
```

### フレームワークの構成

Uzumibiは以下の3つのRustクレートから構成されています。

| クレート | 概要 |
|---------|------|
| **uzumibi-cli** (v0.6.0-rc2) | プロジェクトのスキャフォールド（雛形生成）を行うCLIツール |
| **uzumibi-gem** (v0.5.0) | mruby/edge上で動作するフレームワークのコア機能を提供するgemライブラリ |
| **uzumibi-art-router** (v0.3.1+) | Adaptive Radix Tree（ART）ベースの高速なURLルーティングライブラリ |

### 動作の仕組み

Uzumibiアプリケーションは、以下のような2層構造で動作します。

1. **WASM層（Rust + mruby/edge）**: Rubyコードをmrubyバイトコードにコンパイルし、WASMモジュールに埋め込みます。リクエスト処理はmruby/edge VM上でRubyコードが実行されます。
2. **プラットフォーム層（JavaScript / Rust）**: 各エッジプラットフォーム固有のAPIとWASMモジュールの橋渡しをするグルーコードです。HTTPリクエスト/レスポンスのバイナリシリアライゼーションを担当します。

リクエストの流れは以下のようになります。

```
クライアント → エッジプラットフォーム
    → グルーコード（JS/Rust）がリクエストをバイナリにシリアライズ
    → WASMモジュール内のmruby/edge VMでRubyコードを実行
    → レスポンスをバイナリからデシリアライズ
    → クライアントにレスポンスを返却
```

### ルーティング

Uzumibiのルーターは以下のHTTPメソッドに対応しています。

- `get`
- `post`
- `put`
- `delete`
- `head`
- `options`

URLパスには動的パラメータ（`:name`）やワイルドカード（`*`）を含めることができます。パラメータは `req.params` ハッシュでアクセスできます。

```ruby
# 動的パラメータの例
get "/users/:id" do |req, res|
  user_id = req.params[:id]
  # ...
end
```

### リクエストオブジェクト

ルートハンドラのブロックに渡される `req` オブジェクト（`Uzumibi::Request`）は、以下の情報を保持しています。

- `req.method` - HTTPメソッド（GET, POSTなど）
- `req.path` - リクエストパス
- `req.query` - クエリ文字列
- `req.headers` - リクエストヘッダー（Hash）
- `req.body` - リクエストボディ
- `req.params` - URLパラメータ、クエリパラメータ、フォームデータを統合したHash

### レスポンスオブジェクト

`res` オブジェクト（`Uzumibi::Response`）には以下のプロパティを設定してレスポンスを構築します。

- `res.status_code` - HTTPステータスコード（整数）
- `res.headers` - レスポンスヘッダー（Hash）
- `res.body` - レスポンスボディ（文字列）

ルートハンドラのブロックは必ず `res` オブジェクトを返す必要があります。

### 外部サービス連携

Uzumibiは、エッジプラットフォームが提供する外部サービスへのアクセスもサポートしています（Cloudflare Workersの場合）。

- **Uzumibi::Fetch** - 外部のHTTP APIを呼び出す
- **Uzumibi::KV** - Durable Objectを利用したKey-Valueストア
- **Uzumibi::Queue** - Cloudflare Queuesによる非同期メッセージング

これらの機能については後の章で詳しく解説します。
