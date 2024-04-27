---
関数のデコード実装 ~ セクションのデコードまで ~
---

前章ではデコードの基本的な実装方法について解説をしたので、本章では関数をデコードできるように実装していく。

まず手始めに次のように何もしないもっとも小さな関数をデコードできるように実装をしていく。

```wat
(module
  (func)
)
```

上記のWATをWasmバイナリにコンパイルすると次のようなバイナリ構造になる。

```
0000000: 0061 736d        ; WASM_BINARY_MAGIC
0000004: 0100 0000        ; WASM_BINARY_VERSION
; section "Type" (1)
0000008: 01               ; section code
0000009: 04               ; section size
000000a: 01               ; num types
; func type 0
000000b: 60               ; func
000000c: 00               ; num params
000000d: 00               ; num results
; section "Function" (3)
000000e: 03               ; section code
000000f: 02               ; section size
0000010: 01               ; num functions
0000011: 00               ; function 0 signature index
; section "Code" (10)
0000012: 0a               ; section code
0000013: 04               ; section size
0000014: 01               ; num functions
; function body 0
0000015: 02               ; func body size
0000016: 00               ; local decl count
0000017: 0b               ; end
```

## セクションのデコード
関数のデコードを実装するには、次のセクションのデコード処理を実装する必要がある。

| セクション         | 概要                                 |
|--------------------|--------------------------------------|
| `Type Section`     | 関数シグネチャの情報                 |
| `Code Section`     | 関数ごとの命令などの情報             |
| `Function Section` | 関数シグネチャへの参照情報           |

各種セクションのフォーマットについてはWasmバイナリの構造の章で説明したとおりなので、それを参照してもらいながら実装について解説していく。

### セクションヘッダーのデコード
各種セクションには必ず`section code`と`section size`を持つセクションヘッダーがあるので、
まずは`src/binary/section.rs`ファイルを作成して、`section code`のEnumを定義する。

src/binary/section.rs
```rust
use num_derive::FromPrimitive;

#[derive(Debug, PartialEq, Eq, FromPrimitive)]
pub enum SectionCode {
    Type = 0x01,
    Import = 0x02,
    Function = 0x03,
    Memory = 0x05,
    Export = 0x07,
    Code = 0x0a,
    Data = 0x0b,
}
```

src/binary.rs
```diff
 pub mod module;
+pub mod section;
```

この`SectionCode`をみて、各セクションのデコード処理をしていくことになる。

```rust
match code {
    SectionCode::Type => {
        ...
    }
    SectionCode::Function => {
        ...
    }
    ...
}
```

次にセクションヘッダーをデコードする関数`decode_section_header()`を実装していく。
この関数は入力から`section code`と`section size`を取得するだけだが、いくつか新しい関数があるのでそれについて解説していく。

src/binary/module.rs
```diff
-use nom::{IResult, number::complete::le_u32, bytes::complete::tag};
+use super::section::SectionCode;
+use nom::{
+    bytes::complete::tag,
+    number::complete::{le_u32, le_u8},
+    sequence::pair,
+    IResult,
+};
+use nom_leb128::leb128_u32;
+use num_traits::FromPrimitive as _;
 
 #[derive(Debug, PartialEq, Eq)]
 pub struct Module {
@@ -34,6 +42,17 @@ impl Module {
     }
 }
 
+fn decode_section_header(input: &[u8]) -> IResult<&[u8], (SectionCode, u32)> {
+    let (input, (code, size)) = pair(le_u8, leb128_u32)(input)?; // ①
+    Ok((
+        input,
+        (
+            SectionCode::from_u8(code).expect("unexpected section code"), // ②
+            size,
+        ),
+    ))
+}
+
 #[cfg(test)]
 mod tests {
     use crate::binary::module::Module;
```

①の`pair()`は2つのパーサーをもとに、新しいパーサーを返してくれる。
`pair()`を使っている理由だが、`section code`と`section size`はフォーマットが決まっているので、それぞれをパースする関数をまとめて1回の関数呼び出しで処理できるようするためである。

`pair()`を使わない場合は次のような実装になる。

```rust
let (input, code) = le_u8(input);
let (input, size) = leb128_u32(input);
```

`section code`は1バイト固定なので`le_u8()`を使っている。
`section size`はLEB128[^1]でエンコードされた`u32`なので、値を読み取る際は`leb128_u32()`を使う必要がある。

<div class="warning">

`Wasm spec`では数値はすべてLEB128でエンコードすると定められていることに注意  
筆者はこれを知らず、テストが通らなくて数時間を溶かしたことがある

</div>

②の`SectionCode::from_u8()`は`num_derive::FromPrimitive`マクロで実装された関数である。
読み取った1バイトの数値から`SectionCode`に変換するために使っている。
これを使わない場合、次のように筋肉で解決する必要が出てくる。

```rust
impl From<u8> for SectionCode {
    fn from(code: u8) -> Self {
        match code {
            0x00 => Self::Custom,
            0x01 => Self::Type,
            ...
        }
    }
}
```

セクションヘッダーのデコードの実装はできたので、続けてデコード処理の骨組みを実装していく。

src/binary/module.rs
```diff
 use super::section::SectionCode;
 use nom::{
-    bytes::complete::tag,
+    bytes::complete::{tag, take},
     number::complete::{le_u32, le_u8},
     sequence::pair,
     IResult,
@@ -38,6 +38,29 @@ impl Module {
             magic: "\0asm".into(),
             version,
         };
+
+        let mut remaining = input;
+
+        while !remaining.is_empty() { // 1
+            match decode_section_header(remaining) { // 2
+                Ok((input, (code, size))) => {
+                    let (rest, section_contents) = take(size)(input)?; // 3
+
+                    match code {
+                        _ => todo!(), // 4
+                    };
+
+                    remaining = rest; // 5
+                }
+                Err(err) => return Err(err),
+            }
+        }
         Ok((input, module))
     }
 }
```

上記の実装では、次のことをやっている。

1. 入力である`remaining`が空になるまで、②~⑤の処理を繰り返す
2. セクションヘッダーをデコードし、セクションコードとサイズ、残りの入力を取得
3. セクションサイズ分のバイト列を残りの入力から更に取得する
    - `take()`は指定したサイズ分だけ、入力を読み取る関数
    - 読み取ったバイト列は`section_contents`、残りは`rest`
4. 各種セクションのデコード処理を記述する
5. 残りの入力`rest`を次のループで使うため、`remaining`に再代入

やっていることはシンプルだが、最初は分かりづらいと思うので、バイナリ構造をもとに上記の②~⑤の処理を考えてみよう。

まず`Type Section`と`Function Section`のバイナリ構造体の部分を抜き出すと次のとおりである。

```
; section "Type" (1)
0000008: 01               ; section code
0000009: 04               ; section size
000000a: 01               ; num types
; func type 0
000000b: 60               ; func
000000c: 00               ; num params
000000d: 00               ; num results
; section "Function" (3)
000000e: 03               ; section code
000000f: 02               ; section size
0000010: 01               ; num functions
0000011: 00               ; function 0 signature index
...
```

1の時点では`remaining`は次のようになっている。

| remaining                                                       |
|-----------------------------------------------------------------|
| [0x01, 0x04, 0x01, 0x60, 0x0, 0x0, 0x03, 0x02, 0x01, 0x00, ...] |


2が終わった時点で、`input`などが次のようになっている。

| section code | section size | input                                               |
|--------------|--------------|-----------------------------------------------------|
| 0x01         | 0x04         | [0x01, 0x60, 0x0, 0x0, 0x03, 0x02, 0x01, 0x00, ...] |

3が終わった時点で`rest`と`section_contents`は次のようになっている。

| section_contents       | rest                          |
|------------------------|-------------------------------|
| [0x01, 0x60, 0x0, 0x0] | [0x03, 0x02, 0x01, 0x00, ...] |

4では`section_contents`を更にデコードしていく。

5では`remaining`に`rest`の値が入る、この時点で`remaining`が次のセクションの入力になる。

| remaining                     |
|-------------------------------|
| [0x03, 0x02, 0x01, 0x00, ...] |

このように、繰り返し入力を消費して各セクションをデコードしていく。
理解してしまえばシンプルだが、理解するまでは繰り返し本節の説明を読み直したり、読者自身で書いてみたりすると良いと思う。

### `Type Section`のデコード
骨組みができたので、続けて`Type Section`のデコード処理を実装していく。
`Type Section`は関数シグネチャ情報を持つセクションで、シグネチャは引数と戻り値の組み合わせである。

バイナリ構造は次のとおり。

```
; section "Type" (1)
0000008: 01               ; section code
0000009: 04               ; section size
000000a: 01               ; num types
; func type 0
000000b: 60               ; func
000000c: 00               ; num params
000000d: 00               ; num results
```

まずは`src/binary/types.rs`ファイルを作って、シグネチャの構造体を定義していく。

src/binary/types.rs
```rust
#[derive(Debug, Default, Clone, PartialEq, Eq)]
pub struct FuncType {
    pub params: Vec<ValueType>,
    pub results: Vec<ValueType>,
}

#[derive(Debug, Clone, PartialEq, Eq)]
pub enum ValueType {
    I32, // 0x7F
    I64, // 0x7E
}

impl From<u8> for ValueType {
    fn from(value: u8) -> Self {
        match value {
            0x7F => Self::I32,
            0x7E => Self::I64,
            _ => panic!("invalid value type: {:X}", value),
        }
    }
}
```

```diff:src/binary.rs
 pub mod module;
 pub mod section;
+pub mod types;
```

`ValueType`は引数の型を表現している。
今回は定義した関数に引数がないためバイナリ構造には型情報が出てこないが、`0x7F`なら`i32`、`0x7E`なら`i64`という仕様になっている。

<div class="warning">

`Wasm Spec`では`i32`、`i64`、`f32`、`f64`の4つの値が定義されているが、
今回は`i32`と`i64`があれば良いので`ValueType`はその2つのみ実装する

</div>

続けて、`Module`構造体に`type_section`フィールドを追加するのと`todo!()`を実装していく。

src/binary/module.rs
```diff
-use super::section::SectionCode;
+use super::{section::SectionCode, types::FuncType};
 use nom::{
     bytes::complete::{tag, take},
     number::complete::{le_u32, le_u8},
@@ -12,6 +12,7 @@ use num_traits::FromPrimitive as _;
 pub struct Module {
     pub magic: String,
     pub version: u32,
+    pub type_section: Option<Vec<FuncType>>,
 }
 
 impl Default for Module {
@@ -19,6 +20,7 @@ impl Default for Module {
         Self {
             magic: "\0asm".to_string(),
             version: 1,
+            type_section: None,
         }
     }
 }
@@ -34,9 +36,10 @@ impl Module {
         let (input, _) = tag(b"\0asm")(input)?;
         let (input, version) = le_u32(input)?;
 
-        let module = Module {
+        let mut module = Module {
             magic: "\0asm".into(),
             version,
+            ..Default::default()
         };
 
         let mut remaining = input;
@@ -47,6 +50,10 @@ impl Module {
                     let (rest, section_contents) = take(size)(input)?;
 
                     match code {
+                        SectionCode::Type => {
+                            let (_, types) = decode_type_section(section_contents)?;
+                            module.type_section = Some(types);
+                        }
                         _ => todo!(),
                     };
```

`decode_type_section()`は実際に`Type Section`のデコード処理をしている関数だが、少し複雑になってくるので、ひとまず固定のデータを返すようにする。
引数と戻り値のデコードと合わせて次章で実装する。

src/binary/module.rs
```diff
@@ -77,6 +77,14 @@ fn decode_section_header(input: &[u8]) -> IResult<&[u8], (SectionCode, u32)> {
     ))
 }
 
+fn decode_type_section(_input: &[u8]) -> IResult<&[u8], Vec<FuncType>> {
+    let func_types = vec![FuncType::default()];
+
+    // TODO: 引数と戻り値のデコード
+
+    Ok((&[], func_types))
+}
+
 #[cfg(test)]
 mod tests {
     use crate::binary::module::Module;
```

### `Function Section`のデコード
`Function Section`はWasmバイナリの構造の章で説明をしたとおり、関数のシグネチャ情報(`Type Section`)を紐付けるための領域である。

バイナリ構造は次のとおり。

```
; section "Function" (3)
000000e: 03               ; section code
000000f: 02               ; section size
0000010: 01               ; num functions
0000011: 00               ; function 0 signature index
```

これをデコードしていくので、まずは`Module`に`function_section`を追加する。

src/binary/module.rs
```diff
@@ -13,6 +13,7 @@ pub struct Module {
     pub magic: String,
     pub version: u32,
     pub type_section: Option<Vec<FuncType>>,
+    pub function_section: Option<Vec<u32>>,
 }
 
 impl Default for Module {
@@ -21,6 +22,7 @@ impl Default for Module {
             magic: "\0asm".to_string(),
             version: 1,
             type_section: None,
+            function_section: None,
         }
     }
 }
@@ -54,6 +56,10 @@ impl Module {
                             let (_, types) = decode_type_section(section_contents)?;
                             module.type_section = Some(types);
                         }
+                        SectionCode::Function => {
+                            let (_, func_idx_list) = decode_function_section(section_contents)?;
+                            module.function_section = Some(func_idx_list);
+                        }
                         _ => todo!(),
                     };
```

`Function Section`は単に`Type Section`と`Code Section`を紐付けるためのインデックス情報を持っているだけなので、Rustでは`Vec<u32>`で表現する。

続けて、`decode_function_section()`を次のように実装する。
`decode_type_section()`が呼ばれる時点の`input`は次のようになっている。
`num functions`は関数の個数を表していて、この数だけインデックスの値を読み取っていく。

```
0000010: 01               ; num functions
0000011: 00               ; function 0 signature index
```

実装は次のようになる。

src/binary/module.rs
```diff
@@ -91,6 +91,19 @@ fn decode_type_section(_input: &[u8]) -> IResult<&[u8], Vec<FuncType>> {
     Ok((&[], func_types))
 }
 
+fn decode_function_section(input: &[u8]) -> IResult<&[u8], Vec<u32>> {
+    let mut func_idx_list = vec![];
+    let (mut input, count) = leb128_u32(input)?; // 1
+
+    for _ in 0..count { // 2
+        let (rest, idx) = leb128_u32(input)?;
+        func_idx_list.push(idx);
+        input = rest;
+    }
+
+    Ok((&[], func_idx_list))
+}
+
 #[cfg(test)]
 mod tests {
     use crate::binary::module::Module;
```

1は`num functions`を読み取り、2はその値の回数だけインデックスの値を繰り返し読み取っている。

### `Code Section`のデコード
`Code Section`は関数の情報が保存されている領域。
関数の情報は次の2つから構成されている。

- ローカル変数の個数と型情報
- 命令列

これをRustで表現すると、次のようになる。

```rust
// 関数の定義
pub struct Function {
    pub locals: Vec<FunctionLocal>,
    pub code: Vec<Instruction>,
}

// ローカル変数の個数と型情報の定義
pub struct FunctionLocal {
    pub type_count: u32,       // ローカル変数の個数
    pub value_type: ValueType, // 変数の型情報
}

// 命令の定義
pub enum Instruction {
    LocalGet(u32),
    End,
    ...
}
```

バイナリ構造は次のとおり。
何もしない関数の場合、ローカル変数はなし、命令も`end`命令の1つだけとなっている。
`end`命令は関数の終わりを意味していて、何もしない関数でも`end`は必ず1つあるという仕様となっている。

```
; section "Code" (10)
0000012: 0a               ; section code
0000013: 04               ; section size
0000014: 01               ; num functions
; function body 0
0000015: 02               ; func body size
0000016: 00               ; local decl count
0000017: 0b               ; end
```

まず`src/binary/instruction.rs`ファイルを作成してそこに命令を定義していく。

src/binary/instruction.rs
```rust
#[derive(Debug, Clone, PartialEq, Eq)]
pub enum Instruction {
    End,
}
```

src/binary.rs
```diff
+pub mod instruction;
 pub mod module;
 pub mod section;
 pub mod types;
```

続けて、ローカル変数の情報を表す`FunctionLocal`を定義していく。

src/binary/types.rs
```diff
@@ -19,3 +19,9 @@ impl From<u8> for ValueType {
         }
     }
 }
+
+#[derive(Debug, Clone, PartialEq, Eq)]
+pub struct FunctionLocal {
+    pub type_count: u32,
+    pub value_type: ValueType,
+}
```

続けて、関数を表す`Function`を定義していく。

src/binary/section.rs
```diff
+use super::{instruction::Instruction, types::FunctionLocal};
 use num_derive::FromPrimitive;
 
 #[derive(Debug, PartialEq, Eq, FromPrimitive)]
@@ -15,3 +16,9 @@ pub enum SectionCode {
     Code = 0x0a,
     Data = 0x0b,
 }
+
+#[derive(Default, Debug, Clone, PartialEq, Eq)]
+pub struct Function {
+    pub locals: Vec<FunctionLocal>,
+    pub code: Vec<Instruction>,
+}
```

続けてデコード処理を実装していきたいが、少し複雑になってくるので現時点ではまずテストを通せる固定のデータ構造を返すようにする。
デコード実装は次章で実装していく。

src/binary/module.rs
```diff
-use super::{section::SectionCode, types::FuncType};
+use super::{
+    instruction::Instruction,
+    section::{Function, SectionCode},
+    types::FuncType,
+};
 use nom::{
     bytes::complete::{tag, take},
     number::complete::{le_u32, le_u8},
@@ -14,6 +18,7 @@ pub struct Module {
     pub version: u32,
     pub type_section: Option<Vec<FuncType>>,
     pub function_section: Option<Vec<u32>>,
+    pub code_section: Option<Vec<Function>>,
 }
 
 impl Default for Module {
@@ -23,6 +28,7 @@ impl Default for Module {
             version: 1,
             type_section: None,
             function_section: None,
+            code_section: None,
         }
     }
 }
@@ -60,6 +66,10 @@ impl Module {
                             let (_, func_idx_list) = decode_function_section(section_contents)?;
                             module.function_section = Some(func_idx_list);
                         }
+                        SectionCode::Code => {
+                            let (_, funcs) = decode_code_section(section_contents)?;
+                            module.code_section = Some(funcs);
+                        }
                         _ => todo!(),
                     };
 
@@ -104,6 +114,16 @@ fn decode_function_section(input: &[u8]) -> IResult<&[u8], Vec<u32>> {
     Ok((&[], func_idx_list))
 }
 
+fn decode_code_section(_input: &[u8]) -> IResult<&[u8], Vec<Function>> {
+    // TODO: ローカル変数と命令のデコード
+    let functions = vec![Function {
+        locals: vec![],
+        code: vec![Instruction::End],
+    }];
+
+    Ok((&[], functions))
+}
+
 #[cfg(test)]
 mod tests {
     use crate::binary::module::Module;
```

最後にテストを実装して通ることを確認する。
といってもテストを通すように実装を合わせているだけなので意味は薄いが、次章でちゃんとデコード処理を実装するので今はこれでよい。

src/binary/module.rs
```diff
@@ -126,7 +126,9 @@ fn decode_code_section(_input: &[u8]) -> IResult<&[u8], Vec<Function>> {
 
 #[cfg(test)]
 mod tests {
-    use crate::binary::module::Module;
+    use crate::binary::{
+        instruction::Instruction, module::Module, section::Function, types::FuncType,
+    };
     use anyhow::Result;
 
     #[test]
@@ -136,4 +138,23 @@ mod tests {
         assert_eq!(module, Module::default());
         Ok(())
     }
+
+    #[test]
+    fn decode_simplest_func() -> Result<()> {
+        let wasm = wat::parse_str("(module (func))")?;
+        let module = Module::new(&wasm)?;
+        assert_eq!(
+            module,
+            Module {
+                type_section: Some(vec![FuncType::default()]),
+                function_section: Some(vec![0]),
+                code_section: Some(vec![Function {
+                    locals: vec![],
+                    code: vec![Instruction::End],
+                }]),
+                ..Default::default()
+            }
+        );
+        Ok(())
+    }
 }
```

テストを実行すると通っていることが分かる。

```sh
running 2 tests
test binary::module::tests::decode_simplest_module ... ok
test binary::module::tests::decode_simplest_func ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

## まとめ
一部TODOが残っているが、本章では関数デコードの実装を解説した。
大枠はこれで把握できたのではないかと思うので、次章は関数の引数、戻り値のデコードと合わせてTODOの部分の実装について解説していく。

[^1]: 任意の大きさの数値を少ないバイト数で格納するための可変長符号圧縮の方式
