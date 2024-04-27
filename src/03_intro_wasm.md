---
Wasm入門
---

本章はWAT（WebAssembly Text Format）というWasmバイナリへコンパイルできる言語を使って、実際にWasmを動かすことを体験していく。

なお、WATの解説はMDNの[WebAssembly テキスト形式の理解](https://developer.mozilla.org/en-US/docs/WebAssembly/Understanding_the_text_format)にとてもわかり易く書かれているので、詳細な説明はそちらを参照してほしい。
「はじめての関数本体」までをひととおり理解できれば、基本的に本章以降の説明で困ることはないと思う。

## 環境
本書は次の環境を使って解説していく。

- OS: macOS Ventura
- CPU: Apple M1 Pro（ARM64）

## 事前準備

### wabtのインストール
まず最初に、[wabt](https://github.com/WebAssembly/wabt)というツール群をインストールする。
次にmacOSでHomebrewを使ったインストール手順を示すが、macOS以外のインストール方法はリポジトリを参照してほしい。

```sh
$ brew install wabt
```

本章ではWATをWasmバイナリに変換する`wat2wasm`を使う。
執筆時点のバージョンは次のとおり。

```sh
$ wat2wasm --version
1.0.33
```

### Wasmtimeのインストール
コンパイルされたWasmバイナリを実行するため、Wasmtimeをインストールする。
次にmacOSとLinuxのインストール手順を示すが、Windowsでのインストール方法は[公式ドキュメント](https://docs.wasmtime.dev/cli-install.html#installing-wasmtime)を参照してほしい。

```sh
$ curl https://wasmtime.dev/install.sh -sSf | bash
```

執筆時点のバージョンは次のとおり。

```sh
$ wasmtime --version
wasmtime-cli 12.0.1
```

## Wasmバイナリを実行してみる
まず`add.wat`ファイルを作って、次のコードを貼り付ける。
このコードは2つの引数を受け取って加算した結果を返す処理を行っている関数となっている。

```wabt
(module
  (func (export "add") (param $a i32) (param $b i32) (result i32)
    (local.get $a)
    (local.get $b)
    i32.add
  )
)
```

次に`wat2wasm`を使ってWasmバイナリを出力して、`wasmtime`を使って実行する。
`wat2wasm`は`WAT`をWasmバイナリにコンパイルしてくれるCLIである。

```sh
# コンパイル
$ wat2wasm add.wat      
# Wasmバイナリが出力されていることを確認する
$ ls
 add.wasm
 add.wat
# wasmtime を使って関数を実行する
$ wasmtime add.wasm --invoke add 1 2
warning: using `--invoke` with a function that takes arguments is experimental and may break in the future
warning: using `--invoke` with a function that returns values is experimental and may break in the future
3
```

## スタックマシンの補足
MDNでもスタックマシンについて説明があったが、少し足りないと感じたので補足する。
まず、さきほど使ったコードの命令リストを見ると次のようになっている。

```wat
(local.get $a)
(local.get $b)
i32.add
```

これは`local.get`は引数の値をスタックにpush、`i32.add`はスタックから値を2つpopして加算した結果をまたスタックにpushする、という処理を行っている。
そして関数が呼び出し元に戻るとき、戻り値がある場合はスタックから値をpopする。

これをRustの擬似コードで示すと、次のような感じになる。

```rust
// 処理する値を保存するスタック
let mut stack: Vec<i32> = vec![];
// 関数のローカル変数を保持する領域
let mut locals: Vec<i32> = vec![];

// 命令を処理するループ
loop {
    let instruction = fetch_inst();

    match instruction {
        inst::LocalGet => {
          let value = locals.pop();
          stack.push(value);
        }
        inst::I32Add => {
          let right = stack.pop();
          let left = stack.pop();
          stack.push(left + right);
        }
        ...
    }
}

return stack.pop();
```

このように、Wasm Runtimeはスタックマシンを使って値を計算しているというすごくシンプルなことをやっている。

<div class="warning">

実際の実装はもっと複雑だが、本質的には上記のようなことを繰り返し処理している。

</div>

## まとめ
本章では軽くWasmを動かしてみつつ、擬似コードで少し実装についても触れた。
WATについて、ほとんどの説明をMDNに丸投げしているが、筆者が書くより遥かにわかりやすいので分からない場合はぜひそちらを繰り返し読み直してみてほしい。

次章はWasm Runtimeを実装する前準備として、Wasmバイナリの構造について解説していく。
