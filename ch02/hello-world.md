## hello worldアプリケーションの実装

プロジェクトが作成できたので、実際にアプリケーションを実装していきましょう。UzumibiではRubyのコード（`lib/app.rb`）と、静的なフロントエンド（`public/` ディレクトリ）の両方を扱えます。

### フロントの実装

まず、`public/` ディレクトリに静的なHTMLファイルを配置してみましょう。`public/index.html` を作成します。

```html
<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <title>Hello Uzumibi</title>
    <style>
        body {
            font-family: sans-serif;
            max-width: 600px;
            margin: 50px auto;
            padding: 0 20px;
        }
        #result {
            margin-top: 20px;
            padding: 15px;
            background: #f0f0f0;
            border-radius: 5px;
        }
    </style>
</head>
<body>
    <h1>Hello Uzumibi!</h1>
    <p>RubyでCloudflare Workersをやっていく気持ち</p>

    <button id="greetBtn">APIを呼び出す</button>
    <div id="result"></div>

    <script>
        document.getElementById('greetBtn').addEventListener('click', async () => {
            const res = await fetch('/api/hello');
            const text = await res.text();
            document.getElementById('result').textContent = text;
        });
    </script>
</body>
</html>
```

`public/` ディレクトリに配置したファイルは、Cloudflare Workersの静的アセット機能によって自動的に配信されます。

### APIの実装

次に、`lib/app.rb` を編集して、APIエンドポイントを実装します。デフォルトで生成されたコードを以下のように書き換えます。

```ruby
class App < Uzumibi::Router
  # ルートパスは静的アセット（public/index.html）に委譲
  get "/" do |req, res|
    fetch_assets
  end

  # シンプルなAPIエンドポイント
  get "/api/hello" do |req, res|
    res.status_code = 200
    res.headers = {
      "content-type" => "text/plain",
      "x-powered-by" => "#{RUBY_ENGINE} #{RUBY_VERSION}"
    }
    res.body = "Hello from Uzumibi! Running on #{RUBY_ENGINE} #{RUBY_VERSION}\n"
    res
  end

  # 動的パラメータを使った挨拶API
  get "/api/greet/:name" do |req, res|
    name = req.params[:name]
    res.status_code = 200
    res.headers = {
      "content-type" => "application/json",
    }
    res.body = JSON.generate({ message: "Hello, #{name}!" })
    res
  end

  # POSTリクエストの処理
  post "/api/echo" do |req, res|
    res.status_code = 200
    res.headers = {
      "content-type" => "text/plain",
    }
    res.body = "Received: #{req.body.inspect}\n"
    res
  end
end

$APP = App.new
```

### コードの解説

#### ルーティング

`Uzumibi::Router` を継承したクラスで、`get`、`post` などのクラスメソッドを使ってルートを定義します。

```ruby
class App < Uzumibi::Router
  get "/パス" do |req, res|
    # ハンドラの処理
  end
end
```

各ルートハンドラのブロックには `req`（リクエスト）と `res`（レスポンス）の2つの引数が渡されます。

#### 静的アセットの配信

```ruby
get "/" do |req, res|
  fetch_assets
end
```

`fetch_assets` メソッドを呼ぶと、リクエストが `public/` ディレクトリの静的アセットに委譲されます。これにより、 `/` のパスに対応する `public/index.html` が返されます。

この時、たとえば `"/file"` にハンドラを設定した上で `/file` というリクエストが来た場合は、`public/file.html` が存在すればそれが返されます。拡張子が補完されることに留意してください。

#### レスポンスの構築

レスポンスオブジェクトに `status_code`、`headers`、`body` を設定し、最後に `res` を返します。

```ruby
get "/api/hello" do |req, res|
  res.status_code = 200
  res.headers = {
    "content-type" => "text/plain",
  }
  res.body = "Hello!\n"
  res  # 必ずresを返す
end
```

#### 動的パラメータ

URLパスに `:パラメータ名` を含めると、そのパスセグメントが動的パラメータとしてキャプチャされます。`req.params` は Hash オブジェクトなので、そこを経由してからアクセスできます。

```ruby
get "/api/greet/:name" do |req, res|
  name = req.params[:name]
  # ...
end
```

#### リクエストボディ

POSTリクエストなどのボディは `req.body` で、同期的に取得できます。リクエストの `content-type` がフォームデータ（`application/x-www-form-urlencoded`）やJSONデータ（`application/json`）の場合は自動的にパースされるので、 `req.params` から統合的にアクセスすることも可能です。

#### グローバル変数 `$APP`

```ruby
$APP = App.new
```

Uzumibi `0.6.x` 系の仕様では、最後にアプリケーションのインスタンスをグローバル変数 `$APP` に代入するルールとしています。この名前の変数がWasm側のRustコードから参照されるため、必ず設定する必要があります。
