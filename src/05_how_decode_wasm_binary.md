---
バイナリデコードの基本的な実装方法
---

本章では[nom](https://crates.io/crates/nom)という[パーサーコンビネーター](https://en.wikipedia.org/wiki/Parser_combinator)を使って、
プリアンブルしか存在しないWasmバイナリをデコードしながら、バイナリデコードの基本実装について解説していく。

本書で使用するRustのバージョンは次のとおり。

```sh
$ rustc --version 
rustc 1.77.2 (25ef9e3d8 2024-04-09)
```

また本書用に作成した`Wasm Runtime`の実装は次のリポジトリにおいてあるので、もし分かりづらい部分があれば直接コードを参照してほしい。

https://github.com/skanehira/tiny-wasm-runtime

## 準備
早速Rustのプロジェクトを作成して、必要なクレートを導入しよう。

```sh
$ cargo new tiny-wasm-runtime --name tinywasm
```

プロジェクトを作成したら、`Cargo.toml`に以下を追記する。

```toml:Cargo.toml
[dependencies]
anyhow = "1.0.71"     # エラーハンドリングを簡易にできるクレート
nom = "7.1.3"         # パーサーコンビネーター
nom-leb128 = "0.2.0"  # LEB128という可変長符号圧縮された数値をデコードするためのクレート
num-derive = "0.4.0"  # 数値型の変換を便利にするクレート
num-traits = "0.2.15" # 数値型の変換を便利にするクレート

[dev-dependencies]
wat = "=1.0.67"             # WATからWasmバイナリをコンパイルするためのクレート
pretty_assertions = "1.4.0" # テスト時の差分を見やすくしてくれるクレート
```

## プリアンブルのデコード
プリアンブルはWasmバイナリの構造の章で説明したとおり、次のようなバイナリ構造になっている。
全部で8バイトあり、先頭の4バイトは`\0asm`、残りの4バイトはバージョン情報となっている。

```
           \0asm
         ┌───┴───┐
0000000: 0061 736d      ; WASM_BINARY_MAGIC
~~~~~~~  ~~             ~~~~~~~~~~~~~~~~~~~~ 
 │        │                   │
 │        │                   └ コメント
 │        └ 16進数表記、2桁で1バイト
 └ アドレスのオフセット

0000004: 0100 0000      ; WASM_BINARY_VERSION
```

これをRustの構造体で表現すると次のとおり。

```rust
pub struct Module {
    pub magic: String,
    pub version: u32,
}
```

実装は小さく始めるのがよいので、最初はプリアンブルのデコード処理を実装していく。

まずは`src`配下に次のファイルを作成する。

- `src/binary.rs`
- `src/lib.rs`
- `src/binary/module.rs`

それぞれ、次のように記述する。

src/binary/module.rs
```rust
#[derive(Debug, PartialEq, Eq)]
pub struct Module {
    pub magic: String,
    pub version: u32,
}
```

src/binary.rs
```rust
pub mod module;
```

src/lib.rs
```rust
pub mod binary;
```

次にテストを実装していく。
テストではWATコードをWasmバイナリにコンパイルして、それをデコードした結果が想定したデータ構造になっていることを確認していく。

src/binary/module.rs
```diff
@@ -3,3 +3,17 @@ pub struct Module {
     pub magic: String,
     pub version: u32,
 }
+
+#[cfg(test)]
+mod tests {
+    use crate::binary::module::Module;
+    use anyhow::Result;
+
+    #[test]
+    fn decode_simplest_module() -> Result<()> {
+        // プリアンブルしか存在しないwasmバイナリを生成
+        let wasm = wat::parse_str("(module)")?;
+        // バイナリをデコードしてModule構造体を生成
+        let module = Module::new(&wasm)?;
+        // 生成したModule構造体が想定通りになっているかを比較
+        assert_eq!(module, Module::default());
+        Ok(())
+    }
+}
```

上記のコードで分かるように、テストをパスするためには`Moduele::new()`と`Module::default()`を実装する必要がある。
`Magic number`とバージョンは不変なので、`Default`トレイトを実装してテスト時の記述量を減らす。

まずは`Default()`を実装していく。

src/binary/module.rs
```diff
     pub version: u32,
 }
 
+impl Default for Module {
+    fn default() -> Self {
+        Self {
+            magic: "\0asm".to_string(),
+            version: 1,
+        }
+    }
+}
+
 #[cfg(test)]
 mod tests {
     use crate::binary::module::Module;
```

続けて、デコード処理の実装をしていく。

src/binary/module.rs
```diff
+use nom::{IResult, number::complete::le_u32, bytes::complete::tag};
+
 #[derive(Debug, PartialEq, Eq)]
 pub struct Module {
     pub magic: String,
@@ -13,6 +15,25 @@ impl Default for Module {
     }
 }
 
+impl Module {
+    pub fn new(input: &[u8]) -> anyhow::Result<Module> {
+        let (_, module) =
+            Module::decode(input).map_err(|e| anyhow::anyhow!("failed to parse wasm: {}", e))?;
+        Ok(module)
+    }
+
+    fn decode(input: &[u8]) -> IResult<&[u8], Module> {
+        let (input, _) = tag(b"\0asm")(input)?;
+        let (input, version) = le_u32(input)?;
+
+        let module = Module {
+            magic: "\0asm".into(),
+            version,
+        };
+        Ok((input, module))
+    }
+}
+
 #[cfg(test)]
 mod tests {
     use crate::binary::module::Module;
```

これでプリアンブルのデコード処理が実装できたので、テストを実行してパスすればOK。

```sh
$ cargo test decode_simplest_module
    Finished test [unoptimized + debuginfo] target(s) in 0.05s
     Running unittests src/lib.rs (target/debug/deps/tinywasm-010073c10c93afeb)

running 1 test
test binary::module::tests::decode_simplest_module ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

     Running unittests src/main.rs (target/debug/deps/tinywasm-9670d80381f93079)
```

### デコード処理の解説
`nom`を使ったことがない方は上記のコードを読んでもよくわからないと思うので解説していく。
理解できている方は読み飛ばして問題ない。

まず、`nom`は入力のバイト列を受け取って、読み取ったバイト列と残りのバイト列をタプルで返すという設計になっている。
なので、たとえば次の`le_u32()`に入力を渡すと、`(残りのバイト列, 読み取ったバイト列)`という結果を得られる。

```rust
let (input, version) = le_u32(input)?;
```

`le_u32()`は`nom`が提供しているパーサーの1つで、リトルエンディアン[^1]で4バイト読み取った値を`u32`に変換した結果を返してくれる。
なので、バイト列から`u32`な数値を取得したい場合はこの関数を使えばよい。

また、`nom`は`tag()`というパーサーも提供していている。
こちらは`tag()`に渡したバイト列と入力が一致しない場合はエラーを返すという挙動をする。
入力のバリデーションと読み取りを同時に処理できると考えればよい。

上記のコードを見ると`b"\0asm"`を`tag()`に渡して、入力を読み取って残りの入力だけを取得する処理ということが分かる。

```rust
let (input, _) = tag(b"\0asm")(input)?;
```

ちなみに、渡した値と入力が一致しない場合は次のようなエラーが発生する。

```sh
Error: failed to parse wasm: Parsing Error: Error { input: [0, 97, 115, 109, 1, 0, 0, 0], code: Tag }
```

まとめると、`decode()`関数の処理は

- バイナリの先頭から4バイトを読み取り、`\0asm`であれば残りの入力を受取る
- 残りの入力からさらに4バイト読み取って残りと`u32`に変換した値の入力を受取る

ということをやっている。

## まとめ
本章ではプリアンブルのデコードを実際に実装して、`nom`の基本的な使い方やデコード処理の流れについて解説した。

バイナリのデコードは基本的にバイト列をフォーマットに従って解析し、所定のデータ型に変換することを繰り返していくだけなので、やること自体はとてもシンプルである。

最初は中々慣れないと思うが、繰り返し書いてみると慣れてくると思うので、焦らずにゆっくりやっていこう。
ちなみに筆者も最初は慣れなかったが、書いていくうちに慣れていったので安心してほしい。

[^1]: `Wasm spec`では、バイナリはリトルエンディアンでエンコードされている
