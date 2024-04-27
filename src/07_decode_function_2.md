---
関数のデコード実装 ~ 命令のデコードまで ~
---

前章では何もしないもっとも小さな関数のデコードを実装したが、以下のTODOを残していた。

- `Type Section`の引数と戻り値のデコード
- `Code Section`のローカル変数と命令のデコード

本章ではそれらの処理を実装をしていく。

## 関数の引数のデコード
引数のデコードを実装していくので、引数を持つ関数を次のように定義する。

```wat
(module
  (func (param i32 i64)
  )
)
```

`Type Section`は次のようになっている。

```
; section "Type" (1)
0000008: 01      ; section code
0000009: 06      ; section size
000000a: 01      ; num types
; func type 0
000000b: 60      ; func
000000c: 02      ; num params
000000d: 7f      ; i32
000000e: 7e      ; i64
000000f: 00      ; num results
```

これを次の手順でデコードしていく。

1. 関数シグネチャの個数である`num types`を読み取る
    - この値の回数だけ2~6を繰り返す
2. `func`の値は読み捨てる
    - 関数シグネチャの種類を表す値だが、`Wasm spec`では`0x60`固定
    - 本書では特に使わない
3. 引数の個数である`num params`を読み取る
4. `num params`の回数だけ、引数の型情報をデコードする
    - `0x7F`の場合は`i32`、`0x7E`の場合は`i64`なので、それぞれ`ValueType`に変換
5. 戻り値の個数である`num results`を読み取る
6. `num results`の回数だけ、戻り値の型情報をデコードする
    - `0x7F`の場合は`i32`、`0x7E`の場合は`i64`なので、それぞれ`ValueType`に変換

上記の手順を実装すると次のとおり。
コメントの番号がそれぞれ上記の手順である。

/src/binary/module.rs
```diff
 use super::{
     instruction::Instruction,
     section::{Function, SectionCode},
-    types::FuncType,
+    types::{FuncType, ValueType},
 };
 use nom::{
     bytes::complete::{tag, take},
+    multi::many0,
     number::complete::{le_u32, le_u8},
     sequence::pair,
     IResult,
@@ -93,10 +94,33 @@ fn decode_section_header(input: &[u8]) -> IResult<&[u8], (SectionCode, u32)> {
     ))
 }
 
-fn decode_type_section(_input: &[u8]) -> IResult<&[u8], Vec<FuncType>> {
-    let func_types = vec![FuncType::default()];
+fn decode_value_type(input: &[u8]) -> IResult<&[u8], ValueType> {
+    let (input, value_type) = le_u8(input)?;
+    Ok((input, value_type.into()))
+}
+
+fn decode_type_section(input: &[u8]) -> IResult<&[u8], Vec<FuncType>> {
+    let mut func_types: Vec<FuncType> = vec![];
+
+    let (mut input, count) = leb128_u32(input)?; // 1
+
+    for _ in 0..count {
+        let (rest, _) = le_u8(input)?; // 2
+        let mut func = FuncType::default();
+
+        let (rest, size) = leb128_u32(rest)?; // 3
+        let (rest, types) = take(size)(rest)?;
+        let (_, types) = many0(decode_value_type)(types)?; // 4
+        func.params = types;
+
+        let (rest, size) = leb128_u32(rest)?; // 5
+        let (rest, types) = take(size)(rest)?;
+        let (_, types) = many0(decode_value_type)(types)?; // 6
+        func.results = types;
 
-    // TODO: 引数と戻り値のデコード
+        func_types.push(func);
+        input = rest;
+    }
 
     Ok((&[], func_types))
 }
```

`many0()`は受け取った関数を使って、入力が終わるまでパースし続けて、入力の残りとパース結果を`Vec`で返す関数である。
これを使って、「`u8`を読み取って`ValueType`に変換」する関数である`decode_value_type()`を繰り返している。
このように、`many0()`を使うことで`for`を使ってデコードする必要がなくなり、実装がシンプルになる。

実装はできたので、続けてテストを実装していくが、現時点では引数のテストのみとする。
戻り値をテストするには命令のデコード処理を実装する必要があるので、別途実装する。

/src/binary/module.rs
```diff
 #[cfg(test)]
 mod tests {
     use crate::binary::{
-        instruction::Instruction, module::Module, section::Function, types::FuncType,
+        instruction::Instruction,
+        module::Module,
+        section::Function,
+        types::{FuncType, ValueType},
     };
     use anyhow::Result;
 
@@ -181,4 +184,26 @@ mod tests {
         );
         Ok(())
     }
+
+    #[test]
+    fn decode_func_param() -> Result<()> {
+        let wasm = wat::parse_str("(module (func (param i32 i64)))")?;
+        let module = Module::new(&wasm)?;
+        assert_eq!(
+            module,
+            Module {
+                type_section: Some(vec![FuncType {
+                    params: vec![ValueType::I32, ValueType::I64],
+                    results: vec![],
+                }]),
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

問題なければ次のとおり、テストが通る。

```sh
running 3 tests
test binary::module::tests::decode_simplest_module ... ok
test binary::module::tests::decode_func_param ... ok
test binary::module::tests::decode_simplest_func ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

## ローカル変数のデコード
次に、ローカル変数のデコードを実装していく。
使用するWATコードは次のとおり。

```wat
(module
  (func
    (local i32)
    (local i64 i64)
  )
)
```

`Code Section`のバイナリ構造は次のようになっている。

```
; section "Code" (10)
0000012: 0a         ; section code
0000013: 08         ; section size 
0000014: 01         ; num functions
; function body 0
0000015: 06         ; func body size 
0000016: 02         ; local decl count
0000017: 01         ; local type count
0000018: 7f         ; i32
0000019: 02         ; local type count
000001a: 7e         ; i64
000001b: 0b         ; end
```

これを次の手順でデコードしていく。

1. 関数の個数である`num functions`を読み取る
    - この値の回数だけ2~5を繰り返す
2. `func body size`を読み取る
3. 2で取得した値のバイト列を切り出す
    - ローカル変数と命令のデコード処理の入力として使用する
4. 3で取得したバイト列を使ってローカル変数の情報をデコードする
    1. ローカル変数の個数である`local decl count`を読み取る
    2. `local decl count`の回数だけ4-3 ~ 4-4の処理を繰り返す
    3. 型の個数である`local type count`を読み取る
    4. `local type count`の回数だけ値を`ValueType`に変換する
5. 残りのバイト列を命令にデコードする

5の命令デコードはひとまず次節で実装するが、流れは上記のとおり。
この手順で実装すると次のとおりになる。

src/binary/module.rs
```diff
 use super::{
     instruction::Instruction,
     section::{Function, SectionCode},
-    types::{FuncType, ValueType},
+    types::{FuncType, FunctionLocal, ValueType},
 };
 use nom::{
     bytes::complete::{tag, take},
@@ -138,16 +138,42 @@ fn decode_function_section(input: &[u8]) -> IResult<&[u8], Vec<u32>> {
     Ok((&[], func_idx_list))
 }
 
-fn decode_code_section(_input: &[u8]) -> IResult<&[u8], Vec<Function>> {
-    // TODO: ローカル変数と命令のデコード
-    let functions = vec![Function {
-        locals: vec![],
-        code: vec![Instruction::End],
-    }];
+fn decode_code_section(input: &[u8]) -> IResult<&[u8], Vec<Function>> {
+    let mut functions = vec![];
+    let (mut input, count) = leb128_u32(input)?; // 1
+
+    for _ in 0..count {
+        let (rest, size) = leb128_u32(input)?; // 2
+        let (rest, body) = take(size)(rest)?; // 3
+        let (_, body) = decode_function_body(body)?; // 4
+        functions.push(body);
+        input = rest;
+    }
 
     Ok((&[], functions))
 }
 
+fn decode_function_body(input: &[u8]) -> IResult<&[u8], Function> {
+    let mut body = Function::default();
+
+    let (mut input, count) = leb128_u32(input)?; // 4-1
+
+    for _ in 0..count { // 4-2
+        let (rest, type_count) = leb128_u32(input)?; // 4-3
+        let (rest, value_type) = le_u8(rest)?; // 4-4
+        body.locals.push(FunctionLocal {
+            type_count,
+            value_type: value_type.into(),
+        });
+        input = rest;
+    }
+
+    // TODO: 命令のデコード
+    body.code = vec![Instruction::End];
+
+    Ok((&[], body))
+}
+
 #[cfg(test)]
 mod tests {
     use crate::binary::{
```

実装はできたので、続けてテストを実装していく。

src/binary/module.rs
```diff
@@ -180,7 +180,7 @@ mod tests {
         instruction::Instruction,
         module::Module,
         section::Function,
-        types::{FuncType, ValueType},
+        types::{FuncType, ValueType, FunctionLocal},
     };
     use anyhow::Result;
 
@@ -232,4 +232,35 @@ mod tests {
         );
         Ok(())
     }
+
+    #[test]
+    fn decode_func_local() -> Result<()> {
+        let wasm = wat::parse_file("src/fixtures/func_local.wat")?;
+        let module = Module::new(&wasm)?;
+        assert_eq!(
+            module,
+            Module {
+                type_section: Some(vec![FuncType::default()]),
+                function_section: Some(vec![0]),
+                code_section: Some(vec![Function {
+                    locals: vec![
+                        FunctionLocal {
+                            type_count: 1,
+                            value_type: ValueType::I32,
+                        },
+                        FunctionLocal {
+                            type_count: 2,
+                            value_type: ValueType::I64,
+                        },
+                    ],
+                    code: vec![Instruction::End],
+                }]),
+                ..Default::default()
+            }
+        );
+        Ok(())
+    }
 }
```

テストデータも用意する。

src/fixtures/func_local.wat
```wat
(module
  (func
    (local i32)
    (local i64 i64)
  )
)
```

問題なければ次のとおり、テストは通る。

```sh
running 4 tests
test binary::module::tests::decode_simplest_module ... ok
test binary::module::tests::decode_simplest_func ... ok
test binary::module::tests::decode_func_param ... ok
test binary::module::tests::decode_func_local ... ok

test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

## 命令のデコード
Wasmの命令は基本的に、オペコードとオペランドの2つから構成されている。
オペコードは命令の識別番号で、その命令がどんなことを行うかを示す部分である。
オペランドはその命令の対象となるものを示す部分である。

たとえば`i32.const`という命令はオペランドの値をスタックにpushするという操作で、
`(i32.const 1)`の場合は`1`、`(i32.const 2)`は`2`をスタックにpushするという意味になる。

なお、オペランドが存在しない命令もある。
たとえば、`i32.add`はスタックから値を2つ`pop`して加算した結果をスタックに`push`するが、この命令にはオペランドはない。

このように、オペコードごとにどんな操作を行うかはWasm Specの[Index of Instructions](https://www.w3.org/TR/wasm-core-1/#a7-index-of-instructions)で見ることができる。
Indexを見るとかなりの命令数があるが、本書では数個の命令だけ実装する。

デコードの話に戻るが、 本節では次のWATをデコードできるように実装していく。

```wat
(module
  (func (param i32 i32) (result i32)
    (local.get 0)
    (local.get 1)
    i32.add
  )
)
```

これは引数を2つ受け取って加算した結果を返す関数である。

`local.get`は引数を取得してスタックに`push`する命令で、オペランドは引数のインデックスとなる。
例えば1つ目の引数を取得したい場合は`0`、2つ目は`1`と言った感じ。

`i32.add`は先程解説したとおり、スタックから値を2つ`pop`して加算する。
結果、2つの引数を加算した値がスタックに積まれ、関数の呼び出し元に戻った際にスタックから値を`pop`して戻り値を取得できる。

`Code Section`のバイナリ構造は次のとおり。

```
; section "Code" (10)
0000015: 0a                                        ; section code
0000016: 09                                        ; section size
0000017: 01                                        ; num functions
; function body 0
0000018: 07                                        ; func body size
0000019: 00                                        ; local decl count
000001a: 20                                        ; local.get
000001b: 00                                        ; local index
000001c: 20                                        ; local.get
000001d: 01                                        ; local index
000001e: 6a                                        ; i32.add
000001f: 0b                                        ; end
```

処理の流れは前の節で解説したので、命令のデコードの部分のみの手順を示す。

5. 残りのバイト列を命令にデコードする
    1. 1バイト読み取り、オペコードに変換
    2. オペコードの種類に応じて、オペランドを読み取る
        1. `local.get`の場合は更に4バイト読み取って、`u32`に変換してからオペコードと合わせて命令に変換
        2. `i32.add`と`end`の場合は、そのまま命令に変換

この手順を実装していく。

まずオペコードを定義するファイル`src/binary/opcode.rs`を作成して、オペコードを定義する。

src/binary/opcode.rs
```rust
use num_derive::FromPrimitive;

#[derive(Debug, FromPrimitive, PartialEq)]
pub enum Opcode {
    End = 0x0B,
    LocalGet = 0x20,
    I32Add = 0x6A,
}
```

src/binary.rs
```diff
 pub mod instruction;
 pub mod module;
+pub mod opcode;
 pub mod section;
 pub mod types;
```

続けて、命令の定義を追加する。

src/binary/instruction.rs
```diff
 #[derive(Debug, Clone, PartialEq, Eq)]
 pub enum Instruction {
     End,
+    LocalGet(u32),
+    I32Add,
 }
```

src/binary/module.rs
```diff
 use super::{
     instruction::Instruction,
     section::{Function, SectionCode},
-    types::{FuncType, FunctionLocal, ValueType},
+    types::{FuncType, FunctionLocal, ValueType}, opcode::Opcode,
 };
 use nom::{
     bytes::complete::{tag, take},
@@ -168,12 +168,31 @@ fn decode_function_body(input: &[u8]) -> IResult<&[u8], Function> {
         input = rest;
     }
 
-    // TODO: 命令のデコード
-    body.code = vec![Instruction::End];
+    let mut remaining = input;
+
+    while !remaining.is_empty() { // 5
+        let (rest, inst) = decode_instructions(remaining)?;
+        body.code.push(inst);
+        remaining = rest;
+    }
 
     Ok((&[], body))
 }
 
+fn decode_instructions(input: &[u8]) -> IResult<&[u8], Instruction> {
+    let (input, byte) = le_u8(input)?;
+    let op = Opcode::from_u8(byte).unwrap_or_else(|| panic!("invalid opcode: {:X}", byte)); // 5-1
+    let (rest, inst) = match op { // 5-2
+        Opcode::LocalGet => { // 5-2-1
+            let (rest, idx) = leb128_u32(input)?;
+            (rest, Instruction::LocalGet(idx))
+        }
+        Opcode::I32Add => (input, Instruction::I32Add), // 5-2-2
+        Opcode::End => (input, Instruction::End), // 5-2-2
+    };
+    Ok((rest, inst))
+}
+
 #[cfg(test)]
 mod tests {
     use crate::binary::{
```

続けて、テストを実装していく。

src/binary/module.rs
```diff
@@ -280,4 +280,31 @@ mod tests {
         );
         Ok(())
     }
+
+    #[test]
+    fn decode_func_add() -> Result<()> {
+        let wasm = wat::parse_file("src/fixtures/func_add.wat")?;
+        let module = Module::new(&wasm)?;
+        assert_eq!(
+            module,
+            Module {
+                type_section: Some(vec![FuncType {
+                    params: vec![ValueType::I32, ValueType::I32],
+                    results: vec![ValueType::I32],
+                }]),
+                function_section: Some(vec![0]),
+                code_section: Some(vec![Function {
+                    locals: vec![],
+                    code: vec![
+                        Instruction::LocalGet(0),
+                        Instruction::LocalGet(1),
+                        Instruction::I32Add,
+                        Instruction::End
+                    ],
+                }]),
+                ..Default::default()
+            }
+        );
+        Ok(())
+    }
 }
```

テストデータも用意する。

/src/fixtures/func_add.wat
```wat
(module
  (func (param i32 i32) (result i32)
    (local.get 0)
    (local.get 1)
    i32.add
  )
)
```

問題なければ次のとおり、テストは通る。

```sh
running 5 tests
test binary::module::tests::decode_simplest_module ... ok
test binary::module::tests::decode_simplest_func ... ok
test binary::module::tests::decode_func_param ... ok
test binary::module::tests::decode_func_add ... ok
test binary::module::tests::decode_func_local ... ok

test result: ok. 5 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

## まとめ
本章では、関数の引数と戻り値、そしてローカル変数と命令のデコード実装について解説した。
最低限の動く関数をデコードできるようになったので、次は関数の実行の仕組みについて解説していく。
