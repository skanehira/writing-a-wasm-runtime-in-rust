# Wasmの概要

本章ではWasmの概要について、主に次の内容を解説していく。

- Wasm Runtimeの概要
- Wasmのメリット・デメリット
- Wasmの利用シーン

## Wasm Runtimeの概要
前章で書いたとおり、Wasmは仮想命令セットである。
その仮想命令を読み取って実行するWasm Runtimeはいわば仮想マシンそのものである。

仮想マシンといえばJava VMやRubyVMなどがあるが、Wasm Runtimeも同類である。
JavaやRubyはソースコードをコンパイルしバイトコードを生成して、そのバイトコードを仮想マシンが実行するといった流れだが、Wasmもほぼ同様である。

ただ、図1で示しているように、WasmはC、Go、Rustといった多数の言語からコンパイルできるのが特徴となっている。

![](/images/about_wasm_runtime.png)
*図1*

本書はいわばJava VMやRubyVMのような仮想マシンを実装していくことになるが、
Wasm Runtime自体はそれほど複雑ではなく、できることは数値の計算とWasm Runtime自身がもつメモリ操作のみである。

では標準出力などの処理はできないのか？と疑問に感じる人も居るだろう。
実はリソース（ファイルやネットワークなど）の操作はWasm Specに含まれておらず、[WASI（WebAssembly System Interface）](https://wasi.dev)という仕様に含まれている。
WASIはPOSIXライクなシステムコール関数の集まりとなっていて、それらの関数を呼ぶことでリソースを操作できるようになる。

ちなみに今回は`Hello World`を出力するために、WASIの`fd_write`という関数を実装していく。

## Wasmのメリット・デメリット

筆者が考えるWasmのメリット・デメリットは次のとおり。

### メリット
- **セキュアな実行**  
  Wasm RuntimeはWASIを使わない場合は基本的にRuntime外に影響を及ぼすことはない[^1]ので、ある種のサンドボックス環境となっている  
  たとえば環境変数から機密情報を抜き取る、といったようなことはできないためセキュアである
- **ポータビリティ**  
  WasmはOS・CPUに依存せず、Runtimeがあればどこでも実行できる  
  Google ChromeやFirefox、Microsoft Edgeといった主要なブラウザで実行できる
  主要のブラウザ以外にも、[Wasmtime](https://wasmtime.dev)や[wazero](https://wazero.io)といったサーバーサイドで実行できるRuntimeもある
- **言語の多様性**   
  Wasmは複数の言語からコンパイルできるので各言語の財産を利用できる  
  また、他のWasmバイナリをimportできるため、言語の壁を超えて各言語の財産を共有できる

### デメリット
- **古いブラウザのサポートをしていない**  
  基本的にないと思うが、古いブラウザではWasmをサポートしていないため、Wasmバイナリを実行できないことがある
  その場合は[polywasm](https://github.com/evanw/polywasm)というライブラリを使えばWasmを動かせる
- **発展途上の技術である**  
  Wasmは比較的に新しい技術で、WASIとともに現在も仕様の拡張が行われている
  そのためエコシステムもまだ成熟しておらず、Wasmだけで本格的なアプリケーションを構築するのはまだ難しい
- **パフォーマンスはRuntimeに依存する**  
  Runtimeの実装によってはパフォーマンスの差異が発生する
  たとえば、ChromeとWasmtimeでの実行はそもそもRuntimeが異なるため、ベンチマークの比較はその部分を考慮する必要がある
  Wasmバイナリの実行速度を比較するのであれば同じRuntimeで計測する必要があり、Runtimeの速度を計測する場合は同じWasmバイナリで計測する必要がある

## Wasmの利用シーン
Wasmが利用されているシーンについて、いくつか例を紹介していく。

### プラグインシステム
Wasmは複数の言語からコンパイルできることから、プラグイン機構を構築する際にWasmを採用されることがよくある。
たとえば、[zellij](https://github.com/zellij-org/zellij)というターミナルマルチプレクサではWasmを採用している。詳細は[こちら](https://zellij.dev/news/new-plugin-system/)を参照してほしい。

他にも[Evnoy Proxy](https://www.envoyproxy.io)というプロキシサーバーではWasmを使って機能拡張できる仕組みも用意されている。

### サーバーレスアプリケーション
[spin](https://developer.fermyon.com/spin)というフレームワークを使うと、Wasmでサーバレスアプリケーションを構築できる。
spin以外には[wasmCloud](https://wasmcloud.com)や[Wasmer Edge](https://wasmer.io/products/edge)などがある。

### コンテナ
`Docker`や`Kubernetes`でもLinuxコンテナの代わりにWasmを使うことができる。
`Docker`や`Kubernetes`がどのようにLinuxコンテナとWasmを動かしているのかについて、図2をもとに概要を説明する。

![](/images/containerd_shim.png)
*図2*

- `containerd`
  - コンテナイメージの管理（取得や削除など）やコンテナの操作（作成や開始など）をキックするなど
  - 高レベルコンテナランタイムとも呼ばれる
- `runc`
  - 実際にLinuxコンテナの作成と起動をする
  - 低レベルコンテナランタイムとも呼ばれる
- `containerd-shim`
  - `containerd`と`runc`を繋げてくれる
  - 実体はただの実行バイナリ
- `containerd-shim-*`
  - `containerd`と`Wasm Runtime`を繋げてくれる
  - 実体はただの実行バイナリ
  - `containerd-shim-wasmtime`や`containerd-shim-wasmedge`といった実行バイナリがある
  - Rustで`containerd-shim-*`を実装するときは[runwasi](https://github.com/containerd/runwasi)を使う

## まとめ
本章はWasm RuntimeやWasmを使ったエコシステムについてざっくりと紹介したので、
次章では実際にWasmtimeを使ってWasmを実際に使ってみる。

[^1]: Runtimeの実装に脆弱性がないことを前提とする
