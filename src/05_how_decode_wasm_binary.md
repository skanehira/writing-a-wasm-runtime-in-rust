# Basic implementation method of binary decoding

In this chapter, we will use a parser combinator called [nom](https://crates.io/crates/nom) to decode a Wasm binary that only contains a preamble, explaining the basic implementation of binary decoding.

The Rust version used in this book is as follows.

```sh
$ rustc --version 
rustc 1.77.2 (25ef9e3d8 2024-04-09)
```

Also, the implementation of the `Wasm Runtime` created for this book is located in the following repository, so if there are any confusing parts, please refer directly to the code.

https://github.com/skanehira/tiny-wasm-runtime

## Preparation
Let's create a Rust project right away and introduce the necessary crates.

```sh
$ cargo new tiny-wasm-runtime --name tinywasm
```

After creating the project, add the following to `Cargo.toml`.

```toml:Cargo.toml
[dependencies]
anyhow = "1.0.71"     # Crate for easy error handling
nom = "7.1.3"         # Parser combinator
nom-leb128 = "0.2.0"  # For decoding LEB128 variable length code compressed numbers Crate
num-derive = "0.4.0"  # Crate that makes converting numeric types convenient
num-traits = "0.2.15" # Crate that makes converting numeric types convenient

[dev-dependencies]
wat = "=1.0.67"             # Crate for compiling Wasm binaries from WAT
pretty_assertions = "1.4.0" # Crate that makes it easier to see differences during testing
```

## Decoding the Preamble
As explained in the chapter on the structure of Wasm binary, the preamble is structured as follows:
It consists of a total of 8 bytes, where the first 4 bytes are `\0asm` and the remaining 4 bytes are version information.

```
           \0asm
         ┌───┴───┐
0000000: 0061 736d      ; WASM_BINARY_MAGIC
~~~~~~~  ~~             ~~~~~~~~~~~~~~~~~~~~ 
 │        │                   │
 │        │                   └ Comment
 │        └ Hexadecimal notation, 2 digits = 1 byte
 └ Offset of address

0000004: 0100 0000      ; WASM_BINARY_VERSION
```

Representing this in a Rust struct would look like the following.

```rust
pub struct Module {
    pub magic: String,
    pub version: u32,
}
```

It's good to start with a small implementation, so let's start by implementing the decoding process of the preamble.

First, create the following files under the `src` directory.

- `src/binary.rs`
- `src/lib.rs`
- `src/binary/module.rs`

For each file, write the following.

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

Next, proceed with implementing the tests.
In the tests, compile WAT code into Wasm binary and verify that the decoded result matches the expected data structure.

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

As shown in the code above, to pass the tests, you need to implement `Moduele::new()` and `Module::default()`.
Since the magic number and version are constant, implementing the `Default` trait can reduce the amount of code written during testing.

First, implement `Default()`.

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

Next, proceed with the implementation of the decoding process.

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

With this, the decoding process of the preamble has been implemented, so if the tests pass, it's okay.

```sh
$ cargo test decode_simplest_module
    Finished test [unoptimized + debuginfo] target(s) in 0.05s
     Running unittests src/lib.rs (target/debug/deps/tinywasm-010073c10c93afeb)

running 1 test
test binary::module::tests::decode_simplest_module ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

     Running unittests src/main.rs (target/debug/deps/tinywasm-9670d80381f93079)
```

### Explanation of the Decoding Process
For those who are not familiar with using `nom`, the code above may not be easy to understand, so let's explain it.
If you already understand, feel free to skip this explanation.

First, `nom` is designed to take a byte sequence as input and return a tuple of the read byte sequence and the remaining byte sequence.
For example, when you pass input to `le_u32()`, you get the result `(remaining byte sequence, read byte sequence)`.

```rust
let (input, version) = le_u32(input)?;
```

`le_u32()` is one of the parsers provided by `nom`, which reads a 4-byte value in little-endian[^1] and returns the result converted to `u32`.
So, if you want to obtain a `u32` numeric value from a byte sequence, you can use this function.

Additionally, `nom` also provides a parser called `tag()`.
This parser returns an error if the input does not match the byte sequence passed to `tag()`.
Think of it as being able to handle input validation and reading at the same time.

When looking at the above code, it can be understood that `b"\0asm"` is passed to `tag()` to read the input and retrieve only the remaining input.

```rust
let (input, _) = tag(b"\0asm")(input)?;
```

By the way, if the passed value does not match the input, the following error will occur.

```sh
Error: failed to parse wasm: Parsing Error: Error { input: [0, 97, 115, 109, 1, 0, 0, 0], code: Tag }
```

In summary, the processing of the `decode()` function is as follows:

- Read 4 bytes from the beginning of the binary, if it matches `\0asm`, receive the remaining input
- Read another 4 bytes from the remaining input, convert it to a `u32` value, and receive the input

## Summary
In this chapter, we actually implemented the decoding of the preamble, explained the basic usage of `nom`, and the flow of decoding process.

Binary decoding basically involves parsing a byte sequence according to a format and converting it to the specified data type repeatedly, so the process itself is very simple.

It may be difficult at first, but with practice, you will get used to it, so take your time and don't rush.
By the way, the author also had a hard time at first, but gradually got used to it by writing more, so please rest assured.

[^1]: In the `Wasm spec`, binaries are encoded in little-endian.
