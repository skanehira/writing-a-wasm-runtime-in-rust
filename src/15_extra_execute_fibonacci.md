---
番外編 ~ フィボナッチ関数が動くまで ~
---

本章は番外編としてフィボナッチ関数を実行できるように、追加実装をしていく。
`Hello, World!`の出力だとちょっと物足りない人にとってちょうどよいデザートになると思う。

```wat
(module
  (func $fib (export "fib") (param $n i32) (result i32)
    (if
      (i32.lt_s (local.get $n) (i32.const 2))
      (return (i32.const 1))
    )
    (return
      (i32.add
        (call $fib (i32.sub (local.get $n) (i32.const 2)))
        (call $fib (i32.sub (local.get $n) (i32.const 1)))
      )
    )
  )
)
```

## バイナリ構造について

`if`のようなフロー制御命令がある場合はどのようなバイナリ構造になるかについて解説する。
次が関数の部分のバイナリ構造となっている

```
; function body 0
0000020: 1d          ; func body size
0000021: 00          ; local decl count
0000022: 20          ; local.get
0000023: 00          ; local index
0000024: 41          ; i32.const
0000025: 02          ; i32 literal
0000026: 48          ; i32.lt_s
0000027: 04          ; if
0000028: 40          ; void
0000029: 41          ; i32.const
000002a: 01          ; i32 literal
000002b: 0f          ; return
000002c: 0b          ; end
000002d: 20          ; local.get
000002e: 00          ; local index
000002f: 41          ; i32.const
0000030: 02          ; i32 literal
0000031: 6b          ; i32.sub
0000032: 10          ; call
0000033: 00          ; function index
0000034: 20          ; local.get
0000035: 00          ; local index
0000036: 41          ; i32.const
0000037: 01          ; i32 literal
0000038: 6b          ; i32.sub
0000039: 10          ; call
000003a: 00          ; function index
000003b: 6a          ; i32.add
000003c: 0f          ; return
000003d: 0b          ; end
```

上記では次の未実装の命令を使っている。

| 命令       | 概要                                               |
|------------|----------------------------------------------------|
| `i32.sub`  | スタックにある2つの値を`pop`して引き算する         |
| `i32.lt_s` | スタックから2つ値を`pop`して`<`で比較する          |
| `if`       | スタックから1つ値を`pop`して評価して分岐を処理する |
| `return`   | 呼び出し元に戻る                                   |

まずは`if`の部分の命令列を見ていく。

```wat
(if
  (i32.lt_s (local.get $n) (i32.const 2))
  (return (i32.const 1))
)
```

命令列は次のとおり。

```wat
0000022: 20          ; local.get
0000023: 00          ; local index
0000024: 41          ; i32.const
0000025: 02          ; i32 literal
0000026: 48          ; i32.lt_s
0000027: 04          ; if
0000028: 40          ; void
0000029: 41          ; i32.const
000002a: 01          ; i32 literal
000002b: 0f          ; return
000002c: 0b          ; end
```

引数と`2`をスタックに`push`して、`i32.lt_s`を使って`<`で比較して結果をスタックに`push`している。
ちなみに`Wasm`では比較結果が`0`の場合は`false`、それ以外は`true`になる。

次に`if`はスタックから比較結果を`pop`して`true`なら次に進み、`false`なら`end`までジャンプする。
`if`の次の`void`だが、`Wasm`では`if`は値を返すことができて、`void`は返り値なしという意味となっている。
この部分は`if`命令のオペランドになる。

今回では`true`の場合は`1`をスタックに`push`して`return`で呼び出し元に戻る。

次に`return`以降の部分を見ていく

```wat
(return
  (i32.add
    (call $fib (i32.sub (local.get $n) (i32.const 2)))
    (call $fib (i32.sub (local.get $n) (i32.const 1)))
  )
)
```

命令列は次のとおり。

```
000002d: 20          ; local.get     
000002e: 00          ; local index   
000002f: 41          ; i32.const     
0000030: 02          ; i32 literal   
0000031: 6b          ; i32.sub       
0000032: 10          ; call          
0000033: 00          ; function index
0000034: 20          ; local.get     
0000035: 00          ; local index   
0000036: 41          ; i32.const     
0000037: 01          ; i32 literal   
0000038: 6b          ; i32.sub       
0000039: 10          ; call          
000003a: 00          ; function index
000003b: 6a          ; i32.add       
000003c: 0f          ; return        
000003d: 0b          ; end           
```

ちょっと分かりづらいかもしれないので、どの命令がどの処理なのか補足すると次のようになる。

```
local.get        ┐                  ┐                 ┐
local index      │ (i32.sub         │                 │
i32.const        ├   (local.get $n) ├ (call $fib ...) │
i32 literal      │   (i32.const 2)) │                 │
i32.sub          ┘                  │                 │
call             ┌──────────────────┘                 │
function index   ┘                                    ├ (return (i32.add ...))
local.get        ┐                  ┐                 │
local index      │ (i32.sub         │                 │
i32.const        ├   (local.get $n) ├ (call $fib ...) │
i32 literal      │   (i32.const 1)) │                 │
i32.sub          ┘                  │                 │
call             ┌──────────────────┘                 │
function index   ┘                                    │
i32.add                             ┌─────────────────┘
return           ┌──────────────────┘
end              ┘                                                    
```

## 命令デコードの実装

バイナリ構造についてわかったと思うので、実装していく。

最初にオペコードを追加する。

src/binary/opcode.rs
```diff
diff --git a/src/binary/opcode.rs b/src/binary/opcode.rs
index 1e0931b..165a563 100644
--- a/src/binary/opcode.rs
+++ b/src/binary/opcode.rs
@@ -2,11 +2,15 @@ use num_derive::FromPrimitive;
 
 #[derive(Debug, FromPrimitive, PartialEq)]
 pub enum Opcode {
+    If = 0x04,
     End = 0x0B,
+    Return = 0x0F,
     LocalGet = 0x20,
     LocalSet = 0x21,
     I32Store = 0x36,
     I32Const = 0x41,
+    I32LtS = 0x48,
     I32Add = 0x6A,
+    I32Sub = 0x6B,
     Call = 0x10,
 }
```

次に`if`命令で使う`Block`を定義する。

src/binary/types.rs
```diff
diff --git a/src/binary/types.rs b/src/binary/types.rs
index bff2cd4..474c296 100644
--- a/src/binary/types.rs
+++ b/src/binary/types.rs
@@ -66,3 +66,23 @@ pub struct Data {
     pub offset: u32,
     pub init: Vec<u8>,
 }
+
+#[derive(Debug, Clone, PartialEq, Eq)]
+pub struct Block {
+    pub block_type: BlockType,
+}
+
+#[derive(Debug, Clone, PartialEq, Eq)]
+pub enum BlockType {
+    Void,
+    Value(Vec<ValueType>),
+}
+
+impl BlockType {
+    pub fn result_count(&self) -> usize {
+        match self {
+            Self::Void => 0,
+            Self::Value(value_types) => value_types.len(),
+        }
+    }
+}
```

`Wasm`の`if`命令は戻り値の数を`Block`で表現していて、`Block::Void`の場合は戻り値がない、ある場合は`Block::Value`になる。
今回は特に戻り値がないので、基本的に`Block::Void`になる。

ちなみに`If`命令以外にも`block`と`loop`命令があり、これらも同様に戻り値のオペランドを持っているので、それらでも`Block`を使う。

続けて命令を定義していく。

src/binary/instruction.rs
```diff
diff --git a/src/binary/instruction.rs b/src/binary/instruction.rs
index 326db0a..8595bdc 100644
--- a/src/binary/instruction.rs
+++ b/src/binary/instruction.rs
@@ -1,10 +1,16 @@
+use super::types::Block;
+
 #[derive(Debug, Clone, PartialEq, Eq)]
 pub enum Instruction {
+    If(Block),
     End,
+    Return,
     LocalGet(u32),
     LocalSet(u32),
     I32Store { align: u32, offset: u32 },
     I32Const(i32),
+    I32Lts,
     I32Add,
+    I32Sub,
     Call(u32),
 }
```

src/binary/module.rs
```diff
diff --git a/src/binary/module.rs b/src/binary/module.rs
index 3fdd323..82e250f 100644
--- a/src/binary/module.rs
+++ b/src/binary/module.rs
@@ -3,8 +3,8 @@ use super::{
     opcode::Opcode,
     section::{Function, SectionCode},
     types::{
-        Data, Export, ExportDesc, FuncType, FunctionLocal, Import, ImportDesc, Limits, Memory,
-        ValueType,
+        Block, BlockType, Data, Export, ExportDesc, FuncType, FunctionLocal, Import, ImportDesc,
+        Limits, Memory, ValueType,
     },
 };
 use nom::{
@@ -213,6 +213,11 @@ fn decode_instructions(input: &[u8]) -> IResult<&[u8], Instruction> {
     let (input, byte) = le_u8(input)?;
     let op = Opcode::from_u8(byte).unwrap_or_else(|| panic!("invalid opcode: {:X}", byte));
     let (rest, inst) = match op {
+        Opcode::If => {
+            let (rest, block) = decode_block(input)?;
+            (rest, Instruction::If(block))
+        }
+        Opcode::Return => (input, Instruction::Return),
         Opcode::LocalGet => {
             let (rest, idx) = leb128_u32(input)?;
             (rest, Instruction::LocalGet(idx))
@@ -230,7 +235,9 @@ fn decode_instructions(input: &[u8]) -> IResult<&[u8], Instruction> {
             let (rest, value) = leb128_i32(input)?;
             (rest, Instruction::I32Const(value))
         }
+        Opcode::I32LtS => (input, Instruction::I32Lts),
         Opcode::I32Add => (input, Instruction::I32Add),
+        Opcode::I32Sub => (input, Instruction::I32Sub),
         Opcode::End => (input, Instruction::End),
         Opcode::Call => {
             let (rest, idx) = leb128_u32(input)?;
@@ -330,6 +337,18 @@ fn deocde_data_section(input: &[u8]) -> IResult<&[u8], Vec<Data>> {
     Ok((input, data))
 }
 
+fn decode_block(input: &[u8]) -> IResult<&[u8], Block> {
+    let (input, byte) = le_u8(input)?;
+
+    let block_type = if byte == 0x40 {
+        BlockType::Void
+    } else {
+        BlockType::Value(vec![byte.into()])
+    };
+
+    Ok((input, Block { block_type }))
+}
+
 fn decode_name(input: &[u8]) -> IResult<&[u8], String> {
     let (input, size) = leb128_u32(input)?;
     let (input, name) = take(size)(input)?;
```

`decode_block(...)`は`Block`を生成している関数だが、`if`の次が`0x40`の場合は戻り値なしを意味し、それ以外はそのまま戻り値の数の値になる。
なお、version 1では戻り値は1つしか返せないので実質`1`固定になる。

続けて命令実装の部分は一旦TODOにしておく。

src/execution/runtime.rs
```diff
diff --git a/src/execution/runtime.rs b/src/execution/runtime.rs
index 573539f..0ebce7c 100644
--- a/src/execution/runtime.rs
+++ b/src/execution/runtime.rs
@@ -223,6 +223,7 @@ impl Runtime {
                         }
                     }
                 }
+                _ => todo!(),
             }
         }
         Ok(())
```

これでデコードのテストを書けるようになったので、実装が問題ないことを確認する。

src/binary/module.rs
```diff
diff --git a/src/binary/module.rs b/src/binary/module.rs
index 82e250f..357da5b 100644
--- a/src/binary/module.rs
+++ b/src/binary/module.rs
@@ -365,8 +365,8 @@ mod tests {
         module::Module,
         section::Function,
         types::{
-            Data, Export, ExportDesc, FuncType, FunctionLocal, Import, ImportDesc, Limits, Memory,
-            ValueType,
+            Block, BlockType, Data, Export, ExportDesc, FuncType, FunctionLocal, Import,
+            ImportDesc, Limits, Memory, ValueType,
         },
     };
     use anyhow::Result;
@@ -652,4 +652,51 @@ mod tests {
         );
         Ok(())
     }
+
+    #[test]
+    fn decode_fib() -> Result<()> {
+        let wasm = wat::parse_file("src/fixtures/fib.wat")?;
+        let module = Module::new(&wasm)?;
+        assert_eq!(
+            module,
+            Module {
+                type_section: Some(vec![FuncType {
+                    params: vec![ValueType::I32],
+                    results: vec![ValueType::I32],
+                }]),
+                function_section: Some(vec![0]),
+                code_section: Some(vec![Function {
+                    locals: vec![],
+                    code: vec![
+                        Instruction::LocalGet(0),
+                        Instruction::I32Const(2),
+                        Instruction::I32Lts,
+                        Instruction::If(Block {
+                            block_type: BlockType::Void
+                        }),
+                        Instruction::I32Const(1),
+                        Instruction::Return,
+                        Instruction::End,
+                        Instruction::LocalGet(0),
+                        Instruction::I32Const(2),
+                        Instruction::I32Sub,
+                        Instruction::Call(0),
+                        Instruction::LocalGet(0),
+                        Instruction::I32Const(1),
+                        Instruction::I32Sub,
+                        Instruction::Call(0),
+                        Instruction::I32Add,
+                        Instruction::Return,
+                        Instruction::End,
+                    ],
+                }]),
+                export_section: Some(vec![Export {
+                    name: "fib".into(),
+                    desc: ExportDesc::Func(0),
+                }]),
+                ..Default::default()
+            }
+        );
+        Ok(())
+    }
 }
```

```sh
running 21 tests
test binary::module::tests::decode_func_param ... ok
test binary::module::tests::decode_simplest_func ... ok
test binary::module::tests::decode_memory ... ok
test binary::module::tests::decode_i32_store ... ok
test binary::module::tests::decode_data ... ok
test binary::module::tests::decode_simplest_module ... ok
test binary::module::tests::decode_fib ... ok
test binary::module::tests::decode_func_local ... ok
test binary::module::tests::decode_func_add ... ok
test execution::runtime::tests::execute_i32_add ... ok
test execution::runtime::tests::not_found_export_function ... ok
test binary::module::tests::decode_func_call ... ok
test execution::runtime::tests::func_call ... ok
test binary::module::tests::decode_import ... ok
test execution::runtime::tests::call_imported_func ... ok
test execution::runtime::tests::not_found_imported_func ... ok
test execution::store::test::init_memory ... ok
test execution::runtime::tests::i32_const ... ok
test execution::runtime::tests::i32_store ... ok
test execution::runtime::tests::local_set ... ok
test execution::runtime::tests::fib ... ok
```

## 命令の実装

デコード処理できたので、次に命令の実装をしていく。
まず一番簡単な`i32.sub`から実装する。

### `i32.sub`の実装
まず`std::ops::Sub`を実装して`Value`同士で引き算ができるようにする。

src/execution/value.rs
```diff
diff --git a/src/execution/value.rs b/src/execution/value.rs
index 21d364d..6a7820f 100644
--- a/src/execution/value.rs
+++ b/src/execution/value.rs
@@ -35,3 +35,14 @@ impl std::ops::Add for Value {
         }
     }
 }
+
+impl std::ops::Sub for Value {
+    type Output = Self;
+    fn sub(self, rhs: Self) -> Self::Output {
+        match (self, rhs) {
+            (Value::I32(left), Value::I32(right)) => Value::I32(left - right),
+            (Value::I64(left), Value::I64(right)) => Value::I64(left - right),
+            _ => panic!("type mismatch"),
+        }
+    }
+}
```

src/execution/runtime.rs
```diff
diff --git a/src/execution/runtime.rs b/src/execution/runtime.rs
index 0ebce7c..a2aabe9 100644
--- a/src/execution/runtime.rs
+++ b/src/execution/runtime.rs
@@ -209,6 +209,13 @@ impl Runtime {
                     let result = left + right;
                     self.stack.push(result);
                 }
+                Instruction::I32Sub => {
+                    let (Some(right), Some(left)) = (self.stack.pop(), self.stack.pop()) else {
+                        bail!("not found any value in the stack");
+                    };
+                    let result = left - right;
+                    self.stack.push(result);
+                }
                 Instruction::Call(idx) => {
                     let Some(func) = self.store.funcs.get(*idx as usize) else {
                         bail!("not found func");
```

テストを追加して実装が問題ないことを確認する。

```wat:src/fixtures/func_sub.wat
(module
  (func (export "sub") (param i32 i32) (result i32) 
    (local.get 0)
    (local.get 1)
    i32.sub
  )
)
```

src/execution/runtime.rs
```diff
diff --git a/src/execution/runtime.rs b/src/execution/runtime.rs
index a2aabe9..4726a24 100644
--- a/src/execution/runtime.rs
+++ b/src/execution/runtime.rs
@@ -353,4 +353,13 @@ mod tests {
         assert_eq!(memory[0], 42);
         Ok(())
     }
+
+    #[test]
+    fn i32_sub() -> Result<()> {
+        let wasm = wat::parse_file("src/fixtures/func_sub.wat")?;
+        let mut runtime = Runtime::instantiate(wasm)?;
+        let result = runtime.call("sub", vec![Value::I32(10), Value::I32(5)])?;
+        assert_eq!(result, Some(Value::I32(5)));
+        Ok(())
+    }
 }
```

```sh
running 21 tests
test binary::module::tests::decode_func_local ... ok
test binary::module::tests::decode_simplest_func ... ok
test binary::module::tests::decode_func_param ... ok
test binary::module::tests::decode_memory ... ok
test binary::module::tests::decode_data ... ok
test binary::module::tests::decode_fib ... ok
test binary::module::tests::decode_i32_store ... ok
test binary::module::tests::decode_import ... ok
test binary::module::tests::decode_func_add ... ok
test binary::module::tests::decode_simplest_module ... ok
test binary::module::tests::decode_func_call ... ok
test execution::runtime::tests::execute_i32_add ... ok
test execution::runtime::tests::call_imported_func ... ok
test execution::runtime::tests::func_call ... ok
test execution::runtime::tests::i32_sub ... ok
test execution::runtime::tests::i32_store ... ok
test execution::runtime::tests::not_found_export_function ... ok
test execution::runtime::tests::not_found_imported_func ... ok
test execution::runtime::tests::i32_const ... ok
test execution::runtime::tests::local_set ... ok
test execution::store::test::init_memory ... ok
```

### `i32.lt_s`の実装

`lt_s`は`i32`同士を`<`で比較する命令なので、まずそれができるように`std::cmp::Ordering`を実装していく。
また、比較結果は`bool`になるので、それをスタックに`push`できるように`From<bool> for Value`も実装する。

/src/execution/value.rs
```diff
diff --git a/src/execution/value.rs b/src/execution/value.rs
index 6a7820f..eee47ac 100644
--- a/src/execution/value.rs
+++ b/src/execution/value.rs
@@ -1,3 +1,5 @@
+use std::cmp::Ordering;
+
 #[derive(Debug, Clone, Copy, PartialEq, Eq)]
 pub enum Value {
     I32(i32),
@@ -19,6 +21,12 @@ impl From<Value> for i32 {
     }
 }
 
+impl From<bool> for Value {
+    fn from(value: bool) -> Self {
+        Value::I32(if value { 1 } else { 0 })
+    }
+}
+
 impl From<i64> for Value {
     fn from(value: i64) -> Self {
         Value::I64(value)
@@ -46,3 +54,13 @@ impl std::ops::Sub for Value {
         }
     }
 }
+
+impl PartialOrd for Value {
+    fn partial_cmp(&self, other: &Self) -> Option<Ordering> {
+        match (self, other) {
+            (Value::I32(a), Value::I32(b)) => a.partial_cmp(b),
+            (Value::I64(a), Value::I64(b)) => a.partial_cmp(b),
+            _ => panic!("type mismatch"),
+        }
+    }
+}
```

続けて命令も実装し、テストを追加して実装が問題ないことを確認する。

/src/execution/runtime.rs
```diff
diff --git a/src/execution/runtime.rs b/src/execution/runtime.rs
index 4726a24..c163669 100644
--- a/src/execution/runtime.rs
+++ b/src/execution/runtime.rs
@@ -216,6 +216,13 @@ impl Runtime {
                     let result = left - right;
                     self.stack.push(result);
                 }
+                Instruction::I32Lts => {
+                    let (Some(right), Some(left)) = (self.stack.pop(), self.stack.pop()) else {
+                        bail!("not found any value in the stack");
+                    };
+                    let result = left < right;
+                    self.stack.push(result.into());
+                }
                 Instruction::Call(idx) => {
                     let Some(func) = self.store.funcs.get(*idx as usize) else {
                         bail!("not found func");
```

```wat:src/fixtures/func_lts.wat
(module
  (func (export "lts") (param i32 i32) (result i32) 
    (local.get 0)
    (local.get 1)
    i32.lt_s
  )
)
```

src/execution/runtime.rs
```diff
diff --git a/src/execution/runtime.rs b/src/execution/runtime.rs
index c163669..8d9ff62 100644
--- a/src/execution/runtime.rs
+++ b/src/execution/runtime.rs
@@ -369,4 +369,13 @@ mod tests {
         assert_eq!(result, Some(Value::I32(5)));
         Ok(())
     }
+
+    #[test]
+    fn i32_lts() -> Result<()> {
+        let wasm = wat::parse_file("src/fixtures/func_lts.wat")?;
+        let mut runtime = Runtime::instantiate(wasm)?;
+        let result = runtime.call("lts", vec![Value::I32(10), Value::I32(5)])?;
+        assert_eq!(result, Some(Value::I32(0)));
+        Ok(())
+    }
 }
```

```sh
running 10 tests
test execution::runtime::tests::i32_const ... ok
test execution::runtime::tests::not_found_imported_func ... ok
test execution::runtime::tests::func_call ... ok
test execution::runtime::tests::i32_sub ... ok
test execution::runtime::tests::local_set ... ok
test execution::runtime::tests::call_imported_func ... ok
test execution::runtime::tests::i32_store ... ok
test execution::runtime::tests::execute_i32_add ... ok
test execution::runtime::tests::not_found_export_function ... ok
test execution::runtime::tests::i32_lts ... ok
```

### `if`/`return`の実装

続けて`if`のフロー制御命令を実装していく。

まず最初に`Label`を追加する。
`Label`は`if`や`block`など戻り値を持つブロックの制御に使う。

たとえば`if`の場合は`else`の分岐もある（今回は実装しない）が、評価結果が`false`の場合は`else`命令にジャンプする必要があり、その挙動を`Label`を使って実現する。

また、`if`のブロックを抜ける際にスタックを巻き戻す必要があるが、それも同様に`Label`を使って実現する。

src/execution/value.rs
```diff
diff --git a/src/execution/value.rs b/src/execution/value.rs
index eee47ac..5c49e06 100644
--- a/src/execution/value.rs
+++ b/src/execution/value.rs
@@ -64,3 +64,16 @@ impl PartialOrd for Value {
         }
     }
 }
+
+#[derive(Debug, Clone, PartialEq)]
+pub enum LabelKind {
+    If,
+}
+
+#[derive(Debug, Clone)]
+pub struct Label {
+    pub kind: LabelKind,
+    pub pc: usize,
+    pub sp: usize,
+    pub arity: usize,
+}
```

次に、命令処理の実装をしていく。

src/execution/runtime.rs
```diff
diff --git a/src/execution/runtime.rs b/src/execution/runtime.rs
index 8d9ff62..f0d90f8 100644
--- a/src/execution/runtime.rs
+++ b/src/execution/runtime.rs
@@ -3,13 +3,16 @@ use std::mem::size_of;
 use super::{
     import::Import,
     store::{ExternalFuncInst, FuncInst, InternalFuncInst, Store},
-    value::Value,
+    value::{LabelKind, Value},
     wasi::WasiSnapshotPreview1,
 };
-use crate::binary::{
-    instruction::Instruction,
-    module::Module,
-    types::{ExportDesc, ValueType},
+use crate::{
+    binary::{
+        instruction::Instruction,
+        module::Module,
+        types::{ExportDesc, ValueType},
+    },
+    execution::value::Label,
 };
 use anyhow::{anyhow, bail, Result};
 
@@ -19,6 +22,7 @@ pub struct Frame {
     pub sp: usize,
     pub insts: Vec<Instruction>,
     pub arity: usize,
+    pub labels: Vec<Label>,
     pub locals: Vec<Value>,
 }
 
@@ -107,6 +111,7 @@ impl Runtime {
             insts: func.code.body.clone(),
             arity,
             locals,
+            labels: vec![],
         };
 
         self.call_stack.push(frame);
@@ -165,6 +170,24 @@ impl Runtime {
             };
 
             match inst {
+                Instruction::If(block) => {
+                    let cond = self
+                        .stack
+                        .pop()
+                        .ok_or(anyhow!("not found value in the stack"))?;
+
+                    if cond == Value::I32(0) { // 1
+                        frame.pc = get_end_address(&frame.insts, frame.pc as usize)? as isize; // 2
+                    }
+
+                    let label = Label {
+                        kind: LabelKind::If,
+                        pc: frame.pc as usize,
+                        sp: self.stack.len(),
+                        arity: block.block_type.result_count(),
+                    };
+                    frame.labels.push(label);
+                }
                 Instruction::End => {
                     let Some(frame) = self.call_stack.pop() else {
                         bail!("not found frame");
@@ -249,6 +272,30 @@ impl Runtime {
     }
 }
 
+pub fn get_end_address(insts: &[Instruction], pc: usize) -> Result<usize> {
+    let mut pc = pc;
+    let mut depth = 0;
+    loop {
+        pc += 1;
+        let inst = insts.get(pc).ok_or(anyhow!("not found instructions"))?;
+        match inst {
+            Instruction::If(_) => { // 3
+                depth += 1;
+            }
+            Instruction::End => {
+                if depth == 0 {
+                    return Ok(pc);
+                } else {
+                    depth -= 1;
+                }
+            }
+            _ => {
+                // do nothing
+            }
+        }
+    }
+}
+
 pub fn stack_unwind(stack: &mut Vec<Value>, sp: usize, arity: usize) -> Result<()> {
     if arity > 0 {
         let Some(value) = stack.pop() else {
```

`if`命令では次のことをやっている。

1. スタックから値を1つ`pop`して評価
2. `false`だった場合は`if`の終わりの`pc`を算出
    - `true`の場合は`pc`を調整する必要はなく、そのままラベルを`push`して続きの命令を処理する
3. `if`がネストすることがあるので、その場合は`depth`で制御

ちなみに`if`の終わりは関数の終わりと同じく`end`命令となっている。これは`block`と`loop`命令も同様である。

なので`end`命令を処理する際に`if`/`block`/`loop`なのか、関数の終わりなのかを判定する必要がある。
判定方法はシンプルで、`if`/`block`/`loop`のときはラベルが積まれるので、ラベルが空の場合は関数の終わりというふうに判定できる。

実際の実装は次のとおり。

src/execution/runtime.rs
```diff
diff --git a/src/execution/runtime.rs b/src/execution/runtime.rs
index f0d90f8..23ee5dc 100644
--- a/src/execution/runtime.rs
+++ b/src/execution/runtime.rs
@@ -188,6 +188,21 @@ impl Runtime {
                     };
                     frame.labels.push(label);
                 }
+                Instruction::Return => match frame.labels.pop() {
+                    Some(label) => {
+                        let Label { pc, sp, arity, .. } = label;
+                        frame.pc = pc as isize;
+                        stack_unwind(&mut self.stack, sp, arity)?;
+                    }
+                    None => {
+                        let frame = self
+                            .call_stack
+                            .pop()
+                            .ok_or(anyhow!("not found value in th stack"))?;
+                        let Frame { sp, arity, .. } = frame;
+                        stack_unwind(&mut self.stack, sp, arity)?;
+                    }
+                },
                 Instruction::End => {
                     let Some(frame) = self.call_stack.pop() else {
                         bail!("not found frame");
@@ -260,7 +275,6 @@ impl Runtime {
                         }
                     }
                 }
-                _ => todo!(),
             }
         }
         Ok(())
```

ラベルがある場合は`if`などの終了を意味し`pc`とスタックを調整する。
スタックを調整する理由だが、`if`のブロック内の処理でスタックに値が溜まったままになっていることがあるためである。

例えば、次のように`if`が`true`になって`3`がスタックに積まれるだけの処理の場合、値がスタックに残ったままなのでそれらを巻き戻す必要がある。

```wat
(if
  (i32.const 1)
  (i32.const 3)
)
```

これで、フィボナッチ関数を実行するための実装は終わったので、テストを追加して実装が問題ないことを確認する。

src/execution/runtime.rs
```diff
diff --git a/src/execution/runtime.rs b/src/execution/runtime.rs
index 23ee5dc..291353f 100644
--- a/src/execution/runtime.rs
+++ b/src/execution/runtime.rs
@@ -439,4 +439,29 @@ mod tests {
         assert_eq!(result, Some(Value::I32(0)));
         Ok(())
     }
+
+    #[test]
+    fn fib() -> Result<()> {
+        let wasm = wat::parse_file("src/fixtures/fib.wat")?;
+        let mut runtime = Runtime::instantiate(wasm)?;
+        let tests = vec![
+            (1, 1),
+            (2, 2),
+            (3, 3),
+            (4, 5),
+            (5, 8),
+            (6, 13),
+            (7, 21),
+            (8, 34),
+            (9, 55),
+            (10, 89),
+        ];
+
+        for (arg, want) in tests {
+            let args = vec![Value::I32(arg)];
+            let result = runtime.call("fib", args)?;
+            assert_eq!(result, Some(Value::I32(want)));
+        }
+        Ok(())
+    }
 }
```

```sh
running 23 tests
test binary::module::tests::decode_fib ... ok
test binary::module::tests::decode_i32_store ... ok
test binary::module::tests::decode_import ... ok
test binary::module::tests::decode_simplest_func ... ok
test binary::module::tests::decode_func_param ... ok
test binary::module::tests::decode_func_add ... ok
test binary::module::tests::decode_data ... ok
test binary::module::tests::decode_memory ... ok
test binary::module::tests::decode_func_local ... ok
test binary::module::tests::decode_func_call ... ok
test binary::module::tests::decode_simplest_module ... ok
test execution::runtime::tests::execute_i32_add ... ok
test execution::runtime::tests::call_imported_func ... ok
test execution::runtime::tests::i32_const ... ok
test execution::runtime::tests::i32_lts ... ok
test execution::runtime::tests::func_call ... ok
test execution::runtime::tests::i32_store ... ok
test execution::runtime::tests::not_found_export_function ... ok
test execution::runtime::tests::i32_sub ... ok
test execution::runtime::tests::local_set ... ok
test execution::store::test::init_memory ... ok
test execution::runtime::tests::not_found_imported_func ... ok
test execution::runtime::tests::fib ... ok
```

## まとめ

これでフィボナッチ関数が動く`Wasm Runtime`が出来上がった。
制御命令の実装はちょっとおもしろかったのではなかろうか？

もし、こういうのも動かしたいなどの要望があればぜひissueに書いてもらえると嬉しい。
気が向くと書くかもしれない。
