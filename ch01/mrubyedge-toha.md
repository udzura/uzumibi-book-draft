# はじめに

## mruby/edgeとは

mruby/edgeは、WebAssembly環境に特化した軽量なmruby実装です。通常のmruby（軽量版Ruby VM）をベースに、エッジコンピューティングやサーバーレス環境で効率的に動作するよう最適化されています。

### mrubyとは

mrubyは、組み込み環境やリソースの限られた環境で動作するよう設計された、軽量なRuby言語の実装です。Rubyの生みの親であるまつもとゆきひろ氏が主導して開発されています。CRuby（標準のRuby実装）と比べてバイナリサイズが小さく、メモリ使用量も少ないため、IoTデバイスやゲームエンジンなど、さまざまな環境に組み込まれて利用されています。

### mruby/edgeの特徴

mruby/edgeは、mrubyのバイトコード（`.mrb`ファイル）を実行するVMをRustで再実装したものです。以下の特徴があります。

- **WebAssemblyネイティブ**: `wasm32-unknown-unknown` や `wasm32-wasip1` ターゲットへのコンパイルに対応しており、ブラウザやCloudflare Workers、Fastly Compute@Edgeなどの各種WASMランタイムで動作します。
- **Rustによる実装**: VMの実装がRustで書かれているため、メモリ安全性が高く、WASMへのコンパイルがスムーズです。
- **mruby 3.2.0互換**: mruby 3.2.0のオペコードをサポートしており、基本的なRubyの構文やデータ型が利用できます。
- **SharedMemory**: WASMのリニアメモリを活用して、ホスト環境（JavaScript等）とのデータ受け渡しを効率的に行う仕組みを備えています。

### mruby/edgeのアーキテクチャ

mruby/edgeは以下のコンポーネントで構成されています。

1. **RITEバイナリローダー**: mrubyコンパイラ（`mrbc`）が出力するRITEバイナリフォーマット（`.mrb`）を読み込みます。
2. **YAMRB VM**: バイトコードを実行する仮想マシン本体です。256個のレジスタを持ち、100以上のオペコードをサポートしています。
3. **プレリュードライブラリ**: `Object`、`Array`、`String`、`Integer`、`Hash`、`Range`などの基本的なRubyクラスがRustで実装されています。
4. **ヘルパー関数**: Rustコードからmrubyのメソッドを呼び出す `mrb_funcall` や、mrubyにCメソッドを定義する `mrb_define_cmethod` などのFFI関数群です。

### mrubyコードからWASMまでの流れ

mruby/edgeでRubyコードをWASMに変換する流れは以下のようになります。

```
Rubyソースコード (.rb)
    ↓ mrubyコンパイラ (mrbc) でコンパイル
mrubyバイトコード (.mrb)
    ↓ Rustのバイナリに埋め込み (include_bytes!)
    ↓ cargo build --target wasm32-unknown-unknown
WebAssemblyモジュール (.wasm)
```

このWASMモジュールをCloudflare WorkersやFastlyなどのエッジプラットフォーム上で実行することで、Rubyで書かれたアプリケーションをエッジで動かすことができます。

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

### 実装済みのgem

mruby/edgeでは、通常のmrubyにおけるmrbgem（拡張ライブラリ）に相当する機能が、Rustのcrateとして実装されています。利用可能なgemは以下の通りです。

#### コアに組み込まれたgem（フィーチャーフラグで有効化）

| gem名 | フィーチャーフラグ | 説明 |
|---|---|---|
| mruby-random | `mruby-random` | `Random` クラスと `rand()` メソッドを提供します。XorShift PRNGによる実装です。 |
| mruby-regexp | `mruby-regexp` | `Regexp` クラスと `MatchData` クラスを提供します。Rustの `regex` クレートを利用しています。`String#=~` なども利用可能になります。 |

これらはCargo.tomlのfeaturesセクションで有効化します。

```toml
[dependencies]
mrubyedge = { version = "...", features = ["mruby-random", "mruby-regexp"] }
```

#### 独立したcrateとして提供されるgem

| gem名 | crateバージョン | 説明 |
|---|---|---|
| mruby-math | v0.1.1 | `Math` モジュールを提供します。`Math::PI`、`Math::E` などの定数、`sin`、`cos`、`sqrt`、`log` など多数の数学関数が利用できます。 |
| mruby-serde-json | v0.1.1 | `JSON` クラスを提供します。`JSON.parse`（デシリアライズ）と `JSON.dump` / `JSON.generate`（シリアライズ）が利用できます。内部では `serde_json` クレートを利用しています。 |
| mruby-time | v0.1.2 | `Time` クラスを提供します。`Time.now`、`Time.at` によるインスタンス生成、`year`、`month`、`day` などの日時アクセサ、算術演算や比較が利用できます。 |

独立したgemはCargo.tomlにdependenciesとして追加し、Rustコード側で `init_*` 関数を呼び出して初期化します。

```toml
[dependencies]
mruby-math = "0.1"
mruby-serde-json = "0.1"
mruby-time = "0.1"
```

```rust
// VMの初期化後に各gemを有効化
mruby_math::init_math(&mut vm);
mruby_serde_json::init_json(&mut vm);
mruby_time::init_time(&mut vm);
```

#### コアVMに含まれる組み込みクラス

gem以外にも、mruby/edgeのコアVMには以下の基本クラスが組み込まれています。

- `Object`、`Module`、`Class` — オブジェクトシステムの基盤
- `Integer`、`Float` — 数値型
- `String` — 文字列操作（`unpack`/`pack` 含む）
- `Symbol` — シンボル（`to_proc` 対応）
- `Array`、`Hash`、`Range` — コレクション型（`Enumerable` モジュールを include）
- `Proc` — ブロック・ラムダ
- `TrueClass`、`FalseClass`、`NilClass` — 真偽値・nil
- `Enumerable` — `map`、`select`、`find`、`sort`、`reduce`、`sum` など多数のイテレータメソッド
- 例外クラス群 — `RuntimeError`、`TypeError`、`ArgumentError`、`NoMethodError` など
- `SharedMemory` — WASMリニアメモリを介したホスト環境とのデータ受け渡し（mruby/edge独自）

mruby/edgeは、Uzumibiフレームワークの基盤として、Rubyで書かれたWebアプリケーションロジックをエッジ環境で実行可能にする重要な役割を担っています。

