# プロジェクトの作成とhello world

## uzumibi-cliのインストール

Uzumibiでの開発を始めるには、まず `uzumibi-cli` とその前提となるツール群をインストールする必要があります。

### 前提条件

以下のツールが事前にインストールされている必要があります。

#### Rustツールチェイン

Uzumibiの内部ビルドにはRustコンパイラが必要です。`rustup` を使ってインストールします。

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

インストール後、WASMターゲットを追加します。

```bash
rustup target add wasm32-unknown-unknown
```

バージョンを確認します。

```bash
$ rustc --version
rustc 1.85.0 (4d215e2c2 2025-02-17)
```

#### Node.js と pnpm

Cloudflare Workersでは、Wranglerを使った開発・デプロイにNode.jsが必要です。

```bash
# Node.jsのインストール（バージョンマネージャーを使う場合の例）
# fnmの場合
fnm install --lts
# nvm の場合
nvm install --lts

# pnpmのインストール
npm install -g pnpm
```

バージョンを確認します。

```bash
$ node --version
v22.14.0

$ pnpm --version
10.6.2
```

#### Wrangler CLI

Cloudflare Workersの開発・デプロイに使用するCLIツールです。プロジェクト作成後に `pnpm install` で自動的にインストールされますが、グローバルにインストールしておくと便利です。

```bash
pnpm install -g wrangler
```

#### clang / ビルドツール

mrubyのコンパイラ（`mrbc`）のビルドに `clang` が必要です。

macOSの場合：

```bash
xcode-select --install
```

Linuxの場合：

```bash
# Ubuntu/Debian
sudo apt-get install clang build-essential

# Fedora
sudo dnf install clang
```

### uzumibi-cliのインストール

`uzumibi-cli` はRustのパッケージマネージャー `cargo` を使ってインストールします。

```bash
cargo install uzumibi-cli
```

インストールが完了したら、バージョンを確認します。

```bash
$ uzumibi --version
uzumibi-cli 0.6.0-rc2
```

ヘルプを表示して、利用可能なコマンドを確認できます。

```bash
$ uzumibi --help
```

これで開発環境の準備は完了です。次の節でプロジェクトを作成していきましょう。
