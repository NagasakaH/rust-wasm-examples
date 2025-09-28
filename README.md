# WASM学習用リポジトリ

最近Wasmが使われる環境が増えて来たのである程度使えるように基礎を習得しておく 
現状でそこそこ実用性がありそうなRustでとりあえず試してみる

学習した限りまだ過渡期で次の3.0辺りで他の言語サポートが強化されたり　Wasmコンテナの実用性が上がれば一気に普及する可能性もあるかなといった印象

WASM関連の情報

- ここ数年でWASMが使われてるプロジェクトの参考例
    - https://madewithwebassembly.com/all-projects/
- figmaの高速化事例
    - https://www.figma.com/blog/webassembly-cut-figmas-load-time-by-3x/
- WebAssembly 3.0正式仕様決定(2025/9)
  - https://webassembly.org/news/2025-09-17-wasm-3.0/
  - ガベージコレクションにも対応するようになるのでC/C++/Rust以外の言語の移植性が大きく上がるかもしれない
  - ブラウザには実装されてきてる => 年末〜来年以降に期待！

学習コンテンツ

- ここは今後使われなくなるので見る必要なし
    - ~~[Rust and WebAssembly](https://rustwasm.github.io/book/)のチュートリアル~~
- 今後もメンテナンスされていくはずのドキュメント
    - [The `wasm-bindgen` Guide](https://wasm-bindgen.github.io/wasm-bindgen/) 
    - [WebAssembly Guides(Mozilla)](https://developer.mozilla.org/en-US/docs/WebAssembly/Guides)
        - [Compiling from Rust to Webassembly](https://developer.mozilla.org/en-US/docs/WebAssembly/Guides/Rust_to_Wasm)

## 補足

rustwasmグループからwasm-bindgenグループに移行するらしいので情報の参照先が今後大きく変わる(古いドキュメントのリンク先に注意)

https://blog.rust-lang.org/inside-rust/2025/07/21/sunsetting-the-rustwasm-github-org/

古い情報だとwasm-packがよく使われているが、wasm-bindgenのサンプルを見ているとnpm側で@wasm-tool/wasm-pack-pluginを使ってビルドを行うものがいくつかあるように見える(webpackでnpm経由でビルドをするっぽい)

wasm-packはここに引っ越しされている、今後このあたりのツール類で何が主流になっていくのか情報を追っていく必要がありそう、今回はとりあえずwasm-packも使って進める

https://github.com/drager/wasm-pack

## 前提条件

- rustup,rustc,cargoはRust公式サイトので導入済みであること
- 最新のNPMが導入されていること

## 環境構築

### wasm-packのインストール

```
cargo install wasm-pack
```

###  wasm-bindgen-cliのインストール

```
cargo install wasm-bindgen-cli
```

### お試し用のソースコードを作成する

下記の２ファイルを作成する

Cargo.toml
```toml
[package]
authors = ["The wasm-bindgen Developers"]
edition = "2021"
name = "hello_world"
publish = false
version = "0.0.0"

[lib]
crate-type = ["cdylib"]

[dependencies]
wasm-bindgen = "0.2"
```

src/lib.rs

```rust
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
extern "C" {
    fn alert(s: &str);
}

#[wasm_bindgen]
pub fn greet(name: &str) {
    alert(&format!("Hello, {name}!"));
}
```

## wasm_bidgenについて

RustのコードをJavaScriptから呼び出せるようにコンパイルするのに必要

JavaScriptのコードをRustから呼べるようにする

```rust
#[wasm_bindgen]
extern "C" {
    fn alert(s: &str)
}
```

RustのコードをJavaScriptから呼べるようにする

```rust
#[wasm_bindgen]
pub fn greet() {
    alert("Hello, wasm-game-of-life!");
}
```

## プロジェクトをwasm-packでビルドする

ビルドコマンド

```bash
#ビルドコマンド
$ wasm-pack build
```

下記の成果物(wasmそのものとそれを呼び出すためのインターフェイスファイル一式)が生成される

```text
.
├── hello_world_bg.js
├── hello_world_bg.wasm
├── hello_world_bg.wasm.d.ts
├── hello_world.d.ts
├── hello_world.js
└── package.json
```

javascriptやtypescriptのプロジェクト(Reactなど）から使いたい場合はこんな形でimportしてメソッドを呼ぶのみでOK

```js
import { greet } from "./hello_world";

greet("World!");
```

## プロジェクトをwebpackでビルドできるようにする

コマンド一括でnodeのプロジェクトとrustのプロジェクトをビルドできるようにする ※webpackがwasm-bindgenのドキュメントを見ると使われているがwebpackが開発中止になっており今後置き換えが必要になってくる可能性あり

### 設定ファイルの作成


package.json

```js
{
  "scripts": {
    "build": "webpack",
    "serve": "webpack serve"
  },
  "devDependencies": {
    "@wasm-tool/wasm-pack-plugin": "1.5.0",
    "html-webpack-plugin": "^5.6.0",
    "webpack": "^5.97.0",
    "webpack-cli": "^5.1.4",
    "webpack-dev-server": "^5.0.4"
  }
}
```

webpack.config.js

```js
const path = require('path');
const HtmlWebpackPlugin = require('html-webpack-plugin');
const webpack = require('webpack');
const WasmPackPlugin = require("@wasm-tool/wasm-pack-plugin");

module.exports = {
    entry: './index.js',
    output: {
        path: path.resolve(__dirname, '..', 'dist', 'hello_world'),
        filename: 'index.js',
    },
    plugins: [
        new HtmlWebpackPlugin(),
        new WasmPackPlugin({
            crateDirectory: __dirname
        }),
    ],
    mode: 'development',
    experiments: {
        asyncWebAssembly: true
   }
};
```

### npmでホスティングしてwasmのコードを実行する

下記のコマンドでwasmのビルド~webpackのビルド、サーバーの起動までをやってくれる  
※　最近はwebpackからvite等への移行がweb界隈では進んでいるのでドキュメントの更新やプラグイン等の対応が追いついていないなどあるかもしれない

```bash
npm install
npm run serve
```

## Canvasでお絵描きをする


[wasmbindegen(start)]をつけるとwasmのロードが終わると自動でコードが実行される [参考](https://wasm-bindgen.github.io/wasm-bindgen/reference/attributes/on-rust-exports/start.html)

rustでhtml上のキャンバスをdomから取得してそこに対して描画処理を行うことができる(dom操作で動的にcanvas作ってもOK)

Cargo.toml
```toml
[package]
authors = ["The wasm-bindgen Developers"]
edition = "2021"
name = "hello_world"
publish = false
version = "0.0.0"

[lib]
crate-type = ["cdylib"]

[dependencies]
wasm-bindgen = "0.2"
js-sys = "0.3.81"

[dependencies.web-sys]
version = "0.3.81"
features = ['CanvasRenderingContext2d', 'Document', 'Element', 'HtmlCanvasElement', 'Window']
```

index.html

```html
<html>
  <head>
    <meta content="text/html;charset=utf-8" http-equiv="Content-Type"/>
  </head>
  <body>
    <canvas id="canvas" height="150" width="150"></canvas>
  </body>
</html>
```

rust側のコード

```rust
use std::f64;
use wasm_bindgen::prelude::*;

#[wasm_bindgen(start)]
fn start() {
    let document = web_sys::window().unwrap().document().unwrap();
    let canvas = document.get_element_by_id("canvas").unwrap();
    let canvas: web_sys::HtmlCanvasElement = canvas
        .dyn_into::<web_sys::HtmlCanvasElement>()
        .map_err(|_| ())
        .unwrap();

    let context = canvas
        .get_context("2d")
        .unwrap()
        .unwrap()
        .dyn_into::<web_sys::CanvasRenderingContext2d>()
        .unwrap();

    context.begin_path();

    // Draw the outer circle.
    context
        .arc(75.0, 75.0, 50.0, 0.0, f64::consts::PI * 2.0)
        .unwrap();

    // Draw the mouth.
    context.move_to(110.0, 75.0);
    context.arc(75.0, 75.0, 35.0, 0.0, f64::consts::PI).unwrap();

    // Draw the left eye.
    context.move_to(65.0, 65.0);
    context
        .arc(60.0, 65.0, 5.0, 0.0, f64::consts::PI * 2.0)
        .unwrap();

    // Draw the right eye.
    context.move_to(95.0, 65.0);
    context
        .arc(90.0, 65.0, 5.0, 0.0, f64::consts::PI * 2.0)
        .unwrap();

    context.stroke();
}
```