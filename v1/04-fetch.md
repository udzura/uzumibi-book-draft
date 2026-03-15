# Fetchを使って外部の公開APIを呼び出す

この章では、`Uzumibi::Fetch` を使って外部のWeb APIを呼び出すアプリケーションを作成します。

### プロジェクトの作成

外部サービス連携を有効にしたプロジェクトを新規に作成します。

```bash
uzumibi new \
  --template cloudflare \
  --features enable-external \fetch-example
cd fetch-example
pnpm install
```

### 利用するAPIの選定

この例では、無料で公開されている [JSONPlaceholder](https://jsonplaceholder.typicode.com/) APIを使います。このAPIはダミーのTODOデータやユーザーデータなどを返すテスト用APIです。

主に以下のエンドポイントを利用します。

- `GET https://jsonplaceholder.typicode.com/todos/1` - ID指定でTODOを取得
- `GET https://jsonplaceholder.typicode.com/todos?_limit=5` - TODOの一覧を取得（件数制限付き）

### APIを呼び出す

`lib/app.rb` を以下のように編集します。

```ruby
class App < Uzumibi::Router
  get "/" do |req, res|
    fetch_assets
  end

  # 外部APIからTODOを1件取得
  get "/api/todo/:id" do |req, res|
    id = req.params[:id]
    api_url = "https://jsonplaceholder.typicode.com/todos/#{id}"

    debug_console("Fetching: #{api_url}")
    api_response = Uzumibi::Fetch.fetch(api_url)

    res.status_code = api_response.status_code
    res.headers = {
      "content-type" => "application/json",
    }
    res.body = api_response.body
    res
  end

  # 外部APIからTODO一覧を取得
  get "/api/todos" do |req, res|
    api_url = "https://jsonplaceholder.typicode.com/todos?_limit=5"

    debug_console("Fetching: #{api_url}")
    api_response = Uzumibi::Fetch.fetch(api_url)

    res.status_code = api_response.status_code
    res.headers = {
      "content-type" => "application/json",
    }
    res.body = api_response.body
    res
  end

  # POSTリクエストの転送例
  post "/api/todo" do |req, res|
    api_url = "https://jsonplaceholder.typicode.com/todos"

    debug_console("Posting to: #{api_url}")
    api_response = Uzumibi::Fetch.fetch(
      api_url, "POST",
      JSON.generate(req.body),
      {"content-type" => "application/json"}
    )

    res.status_code = api_response.status_code
    res.headers = {
      "content-type" => "application/json",
    }
    res.body = api_response.body
    res
  end
end

$APP = App.new
```

#### コードの解説

**GETリクエストの転送**

```ruby
api_response = Uzumibi::Fetch.fetch(api_url)
```

`Uzumibi::Fetch.fetch` にURLを渡すだけで、GETリクエストが実行されます。戻り値の `api_response` は `Uzumibi::Response` オブジェクトで、外部APIからのレスポンスを保持しています。

**POSTリクエストの転送**

```ruby
api_response = Uzumibi::Fetch.fetch(api_url, "POST", req.body)
```

第2引数にHTTPメソッド、第3引数にリクエストボディを指定することで、POST等のリクエストも送信できます。

認証など、ヘッダの情報は第4引数にHashとして指定してください。

**デバッグ出力**

```ruby
debug_console("Fetching: #{api_url}")
```

`debug_console` メソッドを使うと、Wranglerのコンソール（ターミナル）にデバッグメッセージを出力できます。ブラウザには表示されません。

### 動作確認

開発サーバーを起動します。

```bash
pnpm run dev
```

curlで動作を確認します。

```bash
# TODO 1件を取得
$ curl http://localhost:8787/api/todo/1
{
  "userId": 1,
  "id": 1,
  "title": "delectus aut autem",
  "completed": false
}

# TODO一覧を取得
$ curl http://localhost:8787/api/todos
[
  {
    "userId": 1,
    "id": 1,
    "title": "delectus aut autem",
    "completed": false
  },
  {
    "userId": 1,
    "id": 2,
    "title": "quis ut nam facilis et officia qui",
    "completed": false
  },...
]
# POSTリクエスト
$ curl -X POST -H "Content-Type: application/json" \
    -d '{"title":"New Todo","completed":false,"userId":1}' \
    http://localhost:8787/api/todo
{
  "completed": false,
  "title": "New Todo",
  "userId": 1,
  "id": 201
}
```

Wranglerのコンソールには、以下のようなデバッグメッセージが表示されます。

```
[debug]: Fetching: https://jsonplaceholder.typicode.com/todos/1
```

Uzumibiから外部APIを呼び出すアプリケーションのサンプルを実装しました。

### この章のまとめ

`Uzumibi::Fetch` を使えば、外部のマイクロサービスやサードパーティAPIとの連携が簡単に実現できます。
