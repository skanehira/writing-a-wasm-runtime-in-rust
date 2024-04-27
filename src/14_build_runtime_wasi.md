---
Runtimeの実装 ~ "Hello, World!"出力まで ~
---

本章では`WASI`の`fd_write`関数を実装して、`Hello, World!`を出力できるようにしていく。
最終的には次のWATを実行できるようになる。

```WAT:src/fixtures/hello_world.wat
(module
  (import "wasi_snapshot_preview1" "fd_write"
    (func $fd_write (param i32 i32 i32 i32) (result i32))
  )
  (memory 1)
  (data (i32.const 0) "Hello, World!\n")

  (func $hello_world (result i32)
    (local $iovs i32)

    (i32.store (i32.const 16) (i32.const 0))
    (i32.store (i32.const 20) (i32.const 14))

    (local.set $iovs (i32.const 16))

    (call $fd_write
      (i32.const 1)
      (local.get $iovs)
      (i32.const 1)
      (i32.const 24)
    )
  )
  (export "_start" (func $hello_world))
)
```

## `WASI`の`fd_write()`の実装

`WASI`は現在まだバージョン1に到達しておらず、`0.1`と`0.2`がある。
`0.1`は一般的に`wasi_snapshot_preview1`と呼ばれていて、主にシステムコール関数を定義したものとなっている。
詳細は[こちら](https://wasi.dev)を参照。

`wasi_snapshot_preview1`に定義されている`fd_write()`は`Wasm Runtime`のメモリからデータを読み取り、標準出力・標準エラー出力にデータを書き込む関数である。
なので、この関数を実装すれば`Hello, World!`を出力できる`Wasm Runtime`の完成だ。

改めて冒頭のWATのインポートを見てみよう。

```wat
(import "wasi_snapshot_preview1" "fd_write"
  (func $fd_write (param i32 i32 i32 i32) (result i32))
)
```

引数と戻り値はそれぞれ、次のようになっている。

- 引数1つ目: 書き込み先の`fd`、`1`は標準出力、`2`は標準エラー出力
- 引数2つ目: メモリの読み取り開始の位置
- 引数3つ目: メモリの読み取り回数、2つ目の値が4バイトずつ加算される
- 引数4つ目: 出力に書き込んだバイト数の保存先、メモリのインデックス値

引数の意味がわかったところ、`$hello_wrold`でやっていることについて解説する。

```wat
(func $hello_world (result i32)
  (local $iovs i32)

  (i32.store (i32.const 16) (i32.const 0)) ;; 1
  (i32.store (i32.const 20) (i32.const 14)) ;; 2

  (local.set $iovs (i32.const 16)) ;; 3

  (call $fd_write ;; 4
    (i32.const 1)
    (local.get $iovs)
    (i32.const 1)
    (i32.const 24)
  )
)
```

1. メモリの16バイト目に`0`を書き込む
   `0`は書き出すメモリデータの先頭
2. メモリの20バイト目に`14`を書き込む
   `14`は書き出すメモリデータバイト数
   つまり`0`から`14`バイト読み取って書き出すということになる
3. 宣言したローカル変数に`16`の値をセットする
   `16`はメモリの読み取り開始の位置の値、`fd_write()`はこの値で指定された位置からメモリを読み取る
4. `fd_write`を呼び出し、`fd 1`に書き出したバイト数をメモリの`24`バイト目に書き込むように指定

ちょっと説明が分かりづらいが、要は`fd`に書き出すメモリデータの範囲の値を配置して、
`fd_write()`が範囲の値を読み取って、その範囲のデータを書き出すということをやっている。

これでやっていることがわかったと思うので、それを実装にしていく。

まず`src/execution/wasi.rs`を作成して次のように`wasi_snapshot_preview1`を表現した構造体を用意する。

src/execution.rs
```diff
diff --git a/src/execution.rs b/src/execution.rs
index 5d6aec6..02686e0 100644
--- a/src/execution.rs
+++ b/src/execution.rs
@@ -2,3 +2,4 @@ pub mod import;
 pub mod runtime;
 pub mod store;
 pub mod value;
+pub mod wasi;
```

src/execution/wasi.rs
```rust
use std::{fs::File, os::fd::FromRawFd};

#[derive(Default)]
pub struct WasiSnapshotPreview1 {
    pub file_table: Vec<Box<File>>,
}

impl WasiSnapshotPreview1 {
    pub fn new() -> Self {
        unsafe {
            Self {
                file_table: vec![
                    Box::new(File::from_raw_fd(0)),
                    Box::new(File::from_raw_fd(1)),
                    Box::new(File::from_raw_fd(2)),
                ],
            }
        }
    }
}
```

`WasiSnapshotPreview1`はファイルのテーブルを持っていて、デフォルトでは`stdin/stdout/stderr`の3つを持つようにしている。
`WASI`にはファイルを開く関数`path_open()`があり、その際にファイルテーブルの配列に追加していくことになるので、本書のスコープ外なので実際はしないがそれを見越したデータ構造にしておく。

次に`WasiSnapshotPreview1::invoke(...)`を実装して、`Wasm Runtime`から`WASI`関数を実行できるようにする。

src/execution/wasi.rs
```diff
diff --git a/src/execution/wasi.rs b/src/execution/wasi.rs
index a75dc9c..b0da928 100644
--- a/src/execution/wasi.rs
+++ b/src/execution/wasi.rs
@@ -1,5 +1,8 @@
+use anyhow::Result;
 use std::{fs::File, os::fd::FromRawFd};
 
+use super::{store::Store, value::Value};
+
 #[derive(Default)]
 pub struct WasiSnapshotPreview1 {
     pub file_table: Vec<Box<File>>,
@@ -17,4 +20,20 @@ impl WasiSnapshotPreview1 {
             }
         }
     }
+
+    pub fn invoke(
+        &mut self,
+        store: &mut Store,
+        func: &str,
+        args: Vec<Value>,
+    ) -> Result<Option<Value>> {
+        match func {
+            "fd_write" => self.fd_write(store, args),
+            _ => unimplemented!("{}", func),
+        }
+    }
+
+    pub fn fd_write(&mut self, store: &mut Store, args: Vec<Value>) -> Result<Option<Value>> {
+        // TODO
+        Ok(Some(0.into()))
+    }
 }
```

`WasiSnapshotPreview1::invoke(...)`は指定した関数名に応じて`WASI`関数を呼べるようにしていて、今後も`WASI`関数を追加する際は`match`の分岐を増やしていくことになる。

続けて、メモリからデータを読み取って`fd`に書き出す処理を実装する。

src/execution/wasi.rs
```diff
diff --git a/src/execution/wasi.rs b/src/execution/wasi.rs
index b0da928..6283250 100644
--- a/src/execution/wasi.rs
+++ b/src/execution/wasi.rs
@@ -1,5 +1,5 @@
 use anyhow::Result;
-use std::{fs::File, os::fd::FromRawFd};
+use std::{fs::File, io::prelude::*, os::fd::FromRawFd};
 
 use super::{store::Store, value::Value};
 
@@ -34,6 +34,49 @@ impl WasiSnapshotPreview1 {
     }
 
     pub fn fd_write(&mut self, store: &mut Store, args: Vec<Value>) -> Result<Option<Value>> {
+        let args: Vec<i32> = args.into_iter().map(Into::into).collect();
+
+        let fd = args[0];
+        let mut iovs = args[1] as usize;
+        let iovs_len = args[2];
+        let rp = args[3] as usize;
+
+        let file = self
+            .file_table
+            .get_mut(fd as usize)
+            .ok_or(anyhow::anyhow!("not found fd"))?;
+
+        let memory = store
+            .memories
+            .get_mut(0)
+            .ok_or(anyhow::anyhow!("not found memory"))?;
+
+        let mut nwritten = 0;
+
+        for _ in 0..iovs_len { // 5
+            let start = memory_read(&memory.data, iovs)? as usize; // 1
+            iovs += 4;
+
+            let len: i32 = memory_read(&memory.data, iovs)?; // 2
+            iovs += 4;
+
+            let end = start + len as usize; // 3
+            nwritten += file.write(&memory.data[start..end])?; // 4
+        }
+
+        memory_write(&mut memory.data, rp, &nwritten.to_le_bytes())?; // 5
+
         Ok(Some(0.into()))
     }
 }
+
+fn memory_read(buf: &[u8], start: usize) -> Result<i32> {
+    let end = start + 4;
+    Ok(<i32>::from_le_bytes(buf[start..end].try_into()?))
+}
+
+fn memory_write(buf: &mut [u8], start: usize, data: &[u8]) -> Result<()> {
+    let end = start + data.len();
+    buf[start..end].copy_from_slice(data);
+    Ok(())
+}
```

`WasiSnapshotPreview1::fd_write(...)`でやっていることはちょっと分かりづらいので、バイト列を示しながら解説する。

まず`WasiSnapshotPreview1::fd_write(...)`が呼ばれる時点のメモリの状態は次のようになる。

| 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 | 11 | 12 | 13 | 14 | 15 | 16 | 17 | 18 | 19 | 20 | ... |
|---|---|---|---|---|---|---|---|---|---|----|----|----|----|----|----|----|----|----|----|----|-----|
| H | e | l | l | o | , |   | W | o | r | l  | d  | !  | \n | 0  | 0  | 0  | 0  | 0  | 0  | 14 | ... |

1で`iovs`で指定された位置にある書き出すデータの開始位置を取得する。
`iovs`は`16`なので値は`0`になっていて、`(i32.store (i32.const 16) (i32.const 0))`で配置した値である。

メモリは4バイトでアライメントされているので`iovs`を+4して、2で書き出すデータの長さを取得する。
`20`にある`14`は`(i32.store (i32.const 20) (i32.const 14))`で配置した値である。

これで、書き出すデータ長さがわかったので、3で切り出すメモリデータの範囲（0~14バイト）を計算して、4でその範囲のバイト列を`fd`に書き出す。
これを`iovs_len`の回数分繰り返し、5で書き出した合計バイト数を`rp`で指定したメモリの番地に配置する。

5が終わった時点で、メモリは次の状態になる。

| ... | 14 | 15 | 16 | 17 | 18 | 19 | 20 | ... | 24 | ... |
|-----|----|----|----|----|----|----|----|-----|----|-----|
| ... | 0  | 0  | 0  | 0  | 0  | 0  | 14 | ... | 13 | ... |

これで`WasiSnapshotPreview1::fd_write(...)`の実装ができたので、それを呼べるようにしていく。

src/execution/runtime.rs
```diff
diff --git a/src/execution/runtime.rs b/src/execution/runtime.rs
index 4fb8807..6fba7e7 100644
--- a/src/execution/runtime.rs
+++ b/src/execution/runtime.rs
@@ -4,6 +4,7 @@ use super::{
     import::Import,
     store::{ExternalFuncInst, FuncInst, InternalFuncInst, Store},
     value::Value,
+    wasi::WasiSnapshotPreview1,
 };
 use crate::binary::{
     instruction::Instruction,
@@ -27,6 +28,7 @@ pub struct Runtime {
     pub stack: Vec<Value>,
     pub call_stack: Vec<Frame>,
     pub import: Import,
+    pub wasi: Option<WasiSnapshotPreview1>,
 }
 
 impl Runtime {
@@ -120,6 +122,13 @@ impl Runtime {
         let args = self
             .stack
             .split_off(self.stack.len() - func.func_type.params.len());
+
+        if func.module == "wasi_snapshot_preview1" {
+            if let Some(wasi) = &mut self.wasi {
+                return wasi.invoke(&mut self.store, &func.func, args);
+            }
+        }
+
         let module = self
             .import
             .get_mut(&func.module)
```

続けて、`Runtime`を生成するときに`WasiSnapshotPreview1`のインスタンスを渡せるようにする

src/execution/runtime.rs
```diff
diff --git a/src/execution/runtime.rs b/src/execution/runtime.rs
index 6fba7e7..573539f 100644
--- a/src/execution/runtime.rs
+++ b/src/execution/runtime.rs
@@ -41,6 +41,19 @@ impl Runtime {
         })
     }
 
+    pub fn instantiate_with_wasi(
+        wasm: impl AsRef<[u8]>,
+        wasi: WasiSnapshotPreview1,
+    ) -> Result<Self> {
+        let module = Module::new(wasm.as_ref())?;
+        let store = Store::new(module)?;
+        Ok(Self {
+            store,
+            wasi: Some(wasi),
+            ..Default::default()
+        })
+    }
+
     pub fn add_import(
         &mut self,
         module_name: impl Into<String>,
```

最後に`wat2wasm`で`hello_world.wat`をコンパイルした`hello_world.wasm`を読み取って実行する処理を`main.rs`に追加する。

src/main.rs
```diff
diff --git a/src/main.rs b/src/main.rs
index e7a11a9..fd8f527 100644
--- a/src/main.rs
+++ b/src/main.rs
@@ -1,3 +1,10 @@
-fn main() {
-    println!("Hello, world!");
+use anyhow::Result;
+use tinywasm::execution::{runtime::Runtime, wasi::WasiSnapshotPreview1};
+
+fn main() -> Result<()> {
+    let wasi = WasiSnapshotPreview1::new();
+    let wasm = include_bytes!("./fixtures/hello_world.wasm");
+    let mut runtime = Runtime::instantiate_with_wasi(wasm, wasi)?;
+    runtime.call("_start", vec![]).unwrap();
+    Ok(())
 }
```

実装が問題なければ、次のように`Hello, World!`が出力されるはずだ。

```sh
$ cargo run -q
Hello, World!
```

## まとめ
これで`Hello, World!`を出力できる小さな`Wasm Runtime`が完成した。
色々と覚えることは多かったと思うが、やっていることは意外と難しくないのではなかろうか。

本書で実装した命令はほんの僅かなのでできることはほとんどないが、`Wasm Runtime`が動く仕組みを実装レベルで理解するには充分である。
もし、完全な`Wasm Runtime`の実装にチャレンジしたい方はぜひ仕様書を読みつつ実装してみてほしい。
大変だが、動かせたときはとても楽しいと思う。

参考までに、version 1の命令とある程度の`WASI`を実装すると次の様なことができる。

https://zenn.dev/skanehira/articles/2023-09-18-rust-wasm-runtime-containerd
https://zenn.dev/skanehira/articles/2023-12-02-wasm-risp

最後にこの本を読んでくれたことに感謝を述べたい。
この本を読んで良かったと思えたら、ぜひSNSなどで拡散してもらえると嬉しい。
