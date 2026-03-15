# はじめに

Uzumibiの話をする前に、Uzumibiのベースとなっているmruby実装であるmruby/edgeについて整理します。その後、Uzumibiの概要を説明します。

## mruby/edgeとは

mruby/edgeは、WebAssembly（Wasm）環境に特化した軽量なmruby実装です。軽量版Ruby VMであるmrubyのバイトコード仕様を利用して、エッジコンピューティングやサーバーレス環境で効率的に動作するような狙いを持って開発しました。

### mrubyとは

[mruby](https://mruby.org/)は、組み込み環境やリソースの限られた環境で動作するよう設計された、軽量なRuby言語の実装です。Rubyの生みの親であるまつもとゆきひろ氏が主導して開発されています。CRuby（標準のRuby実装）と比べてバイナリサイズが小さく、メモリ使用量も少ないため、IoTデバイスやゲームエンジンなど、さまざまな環境に組み込まれて利用されています。

https://mruby.org/

昨今では、さらに小さい [mruby/c](https://github.com/mrubyc/mrubyc) やより実践的な組み込みのエコシステムを持った [PicoRuby](https://picoruby.org/) といった、mrubyのバイトコードと互換性を持った実装も開発され、注目されています。

### mruby/edgeの特徴

mruby/edgeは、mrubyのバイトコード（`.mrb`ファイル）を実行するVMをRustで再実装したものです。以下の特徴があります。

- **WebAssemblyの第一級サポート**: ポータブルなWasmコードを生成できるよう意図しています。現在、`wasm32-unknown-unknown` や `wasm32-wasip1` ターゲットへのコンパイルに対応しており、ブラウザやWasmtime、Cloudflare Workers、Fastly Compute@Edgeなどの各種WASMランタイムで動作します。
- **Rustによる実装**: VMの実装がRustで書かれているため、メモリ安全性が高く、Wasmへのコンパイルがスムーズです。
- **mruby 3.2.0互換**: mruby 3.2.0〜3.4.0のオペコードをサポートしており、基本的なRubyの構文やデータ型が利用できます。
- **SharedMemory**: Wasmのリニアメモリを活用して、ホスト環境（JavaScript等）とのデータ受け渡しを効率的に行う仕組みを備えています。

### mrubyコードからWASMまでの流れ

mruby/edgeでRubyコードをWasmに変換する流れは以下のようになります。

```
Rubyソースコード (.rb)
    ↓ mrubyコンパイラ (mrbc、mruby-compiler2等) でコンパイル
mrubyバイトコード (.mrb)
    ↓ Rustのバイナリに埋め込み
    ↓ cargoでのWasmへのコンパイル
Wasmモジュール (.wasm)
```

このWasmモジュールはブラウザで動作可能なものです。また、Cloudflare WorkersやFastlyなどのエッジプラットフォーム上で実行することで、Rubyで書かれたアプリケーションをエッジで動かすことができます。

他にも、mruby-compiler2 をRustに組み込むことでRubyのスクリプトを直接mruby/edgeで実行可能にすることもできます。実行は [Playground](https://mrubyedge.github.io/playground/) で試すことができます。

https://mrubyedge.github.io/playground/

### サポートしているRubyの機能

mruby/edgeでは以下のRuby言語機能がサポートされています。

- 基本的な演算（四則演算、比較演算）
- 条件分岐（`if/elsif/else`、`case/when`）
- メソッド定義と再帰呼び出し
- クラス定義とインスタンス変数
- `attr_reader` 宣言
- グローバル変数（`$var`）
- 配列（`Array`）、ハッシュ（`Hash`）、文字列（`String`）
- 範囲オブジェクト（`Range`）
- ブロックとイテレータ
- 例外処理（`raise`/`rescue`）
- 文字列の `unpack`/`pack`（バイナリデータ処理）

具体的にサポートするメソッド等は [`COVERAGE.md`](https://github.com/mrubyedge/mrubyedge/blob/v1.1.10/mrubyedge/COVERAGE.md) というファイルで確認が可能です。AIにも参照させると良いでしょう。

また、以下のようなライブラリもmrubyedgeのgemという形で利用可能です。

- `mruby-random`
- `mruby-regexp`
- `mruby-math`
- `mruby-serde-json`
- `mruby-time`

### mruby/edgeのまとめ

mruby/edgeは、Uzumibiフレームワークの基盤として、Rubyで書かれたWebアプリケーションロジックをエッジ環境で実行可能にする重要な役割を担っています。

## Uzumibiとは

Uzumibi（うずみび）は、mruby/edgeを基盤とした、エッジコンピューティングプラットフォーム上でRubyによるWebアプリケーションを構築するための軽量フレームワークです。

Uzumibiという名前は大流行中のエッジフレームワークの先輩 [Hono](https://hono.dev/) を意識しています。Hono + Embedded（埋まっている） = Uzumibi ということです😅

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

### 動作の仕組み

Uzumibiアプリケーションは、以下のような2層構造で動作します。

1. **Wasm層（Rust + mruby/edge）**: Rubyコードをmrubyバイトコードにコンパイルし、Wasmモジュールに埋め込みます。リクエスト処理はmruby/edge VM上でRubyコードが実行されます。
2. **プラットフォーム層（JavaScript / Rust）**: 各エッジプラットフォーム固有のAPIとWasmモジュールの橋渡しをするグルーコードです。HTTPリクエスト/レスポンスのバイナリシリアライゼーションを担当します。

リクエストの流れは以下のようになります。

```
クライアント
    → エッジプラットフォーム
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

URLパスには動的パラメータ（`:name`）やワイルドカード（`*`）を含めることができます。パラメータは `req.params` というHash経由でアクセスできます。

```ruby
# 動的パラメータの例
get "/users/:id" do |req, res|
  user_id = req.params[:id]
  # ...
end
```

本書では、get, post... などのメソッドで定義するRubyのブロック処理を「ルートハンドラ」と呼びます。

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

Uzumibiは、エッジプラットフォームが提供する外部サービスへのアクセスもサポートしています。Cloudflare Workersの場合以下が利用できます。

- **`Uzumibi::Fetch`** - 外部のHTTP APIを呼び出す
- **`Uzumibi::KV`** - Durable Objectを利用したKey-Valueストア
- **`Uzumibi::Queue`** - Cloudflare Queuesによる非同期メッセージング

これらの機能については後の章で詳しく解説します。

現状（2026/03/14段階）では、Cloudflare Workers以外のプラットフォオームでは外部サービスを利用できません。随時対応していきます。無論、Pull Requestは歓迎します！