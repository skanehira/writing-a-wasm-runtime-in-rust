---
Runtimeの実装 ~ 命令の追加実装 ~
---

本章ではこれから実装する機能で次の命令が必要になってくるので、先にそれらを実装する。

| 命令        | 概要                                                 |
|-------------|------------------------------------------------------|
| `local.set` | スタックから値を1つ`pop`してローカル変数に配置する |
| `i32.const` | オペランドの値をスタックに`push`する                 |
| `i32.store` | スタックから値を`pop`してメモリに書き込む            |

### `local.set`の実装

`local.set`命令ではオペランドを持っていて、ローカル変数のどの場所に値を配置するかを指定できるようになっている。

src/binary/opcode.rs
```diff
diff --git a/src/binary/opcode.rs b/src/binary/opcode.rs
index 5d0a2b7..55d30c1 100644
--- a/src/binary/opcode.rs
+++ b/src/binary/opcode.rs
@@ -4,6 +4,7 @@ use num_derive::FromPrimitive;
 pub enum Opcode {
     End = 0x0B,
     LocalGet = 0x20,
+    LocalSet = 0x21,
     I32Add = 0x6A,
     Call = 0x10,
 }
```

src/binary/instruction.rs
```diff
diff --git a/src/binary/instruction.rs b/src/binary/instruction.rs
index c9c6584..81d0d95 100644
--- a/src/binary/instruction.rs
+++ b/src/binary/instruction.rs
@@ -2,6 +2,7 @@
 pub enum Instruction {
     End,
     LocalGet(u32),
+    LocalSet(u32),
     I32Add,
     Call(u32),
 }
```

src/binary/module.rs
```diff
diff --git a/src/binary/module.rs b/src/binary/module.rs
index 40f20fd..8426948 100644
--- a/src/binary/module.rs
+++ b/src/binary/module.rs
@@ -217,6 +217,10 @@ fn decode_instructions(input: &[u8]) -> IResult<&[u8], Instruction> {
             let (rest, idx) = leb128_u32(input)?;
             (rest, Instruction::LocalGet(idx))
         }
+        Opcode::LocalSet => {
+            let (rest, idx) = leb128_u32(input)?;
+            (rest, Instruction::LocalSet(idx))
+        }
         Opcode::I32Add => (input, Instruction::I32Add),
         Opcode::End => (input, Instruction::End),
         Opcode::Call => {
```

src/binary/module.rs
```diff
diff --git a/src/execution/runtime.rs b/src/execution/runtime.rs
index 3c492bc..c52de7b 100644
--- a/src/execution/runtime.rs
+++ b/src/execution/runtime.rs
@@ -154,6 +154,13 @@ impl Runtime {
                     };
                     self.stack.push(*value);
                 }
+                Instruction::LocalSet(idx) => {
+                    let Some(value) = self.stack.pop() else {
+                        bail!("not found value in the stack");
+                    };
+                    let idx = *idx as usize;
+                    frame.locals[idx] = value;
+                }
                 Instruction::I32Add => {
                     let (Some(right), Some(left)) = (self.stack.pop(), self.stack.pop()) else {
                         bail!("not found any value in the stack");
```

やっていることはシンプルで、指定された`frame.locals`のインデックスに`pop`した値を配置する。

### `i32.const`の実装

`i32.const`はオペランドに値を持っていて、この値をスタックに`push`する。

src/binary/opcode.rs
```diff
diff --git a/src/binary/opcode.rs b/src/binary/opcode.rs
index 55d30c1..1eb29fd 100644
--- a/src/binary/opcode.rs
+++ b/src/binary/opcode.rs
@@ -5,6 +5,7 @@ pub enum Opcode {
     End = 0x0B,
     LocalGet = 0x20,
     LocalSet = 0x21,
+    I32Const = 0x41,
     I32Add = 0x6A,
     Call = 0x10,
 }
```

src/binary/instruction.rs
```diff
diff --git a/src/binary/instruction.rs b/src/binary/instruction.rs
index 81d0d95..0e392c5 100644
--- a/src/binary/instruction.rs
+++ b/src/binary/instruction.rs
@@ -3,6 +3,7 @@ pub enum Instruction {
     End,
     LocalGet(u32),
     LocalSet(u32),
+    I32Const(i32),
     I32Add,
     Call(u32),
 }
```

src/binary/module.rs
```diff
diff --git a/src/binary/module.rs b/src/binary/module.rs
index 8426948..26b56b3 100644
--- a/src/binary/module.rs
+++ b/src/binary/module.rs
@@ -14,7 +14,7 @@ use nom::{
     sequence::pair,
     IResult,
 };
-use nom_leb128::leb128_u32;
+use nom_leb128::{leb128_i32, leb128_u32};
 use num_traits::FromPrimitive as _;
 
 #[derive(Debug, PartialEq, Eq)]
@@ -221,6 +221,10 @@ fn decode_instructions(input: &[u8]) -> IResult<&[u8], Instruction> {
             let (rest, idx) = leb128_u32(input)?;
             (rest, Instruction::LocalSet(idx))
         }
+        Opcode::I32Const => {
+            let (rest, value) = leb128_i32(input)?;
+            (rest, Instruction::I32Const(value))
+        }
         Opcode::I32Add => (input, Instruction::I32Add),
         Opcode::End => (input, Instruction::End),
         Opcode::Call => {
```

src/execution/runtime.rs
```diff
diff --git a/src/execution/runtime.rs b/src/execution/runtime.rs
index c52de7b..9044e25 100644
--- a/src/execution/runtime.rs
+++ b/src/execution/runtime.rs
@@ -161,6 +161,7 @@ impl Runtime {
                     let idx = *idx as usize;
                     frame.locals[idx] = value;
                 }
+                Instruction::I32Const(value) => self.stack.push(Value::I32(*value)),
                 Instruction::I32Add => {
                     let (Some(right), Some(left)) = (self.stack.pop(), self.stack.pop()) else {
                         bail!("not found any value in the stack");
```

`local.set`と組み合わせたテストを追加して実装が問題ないことを確認する。

```wat:src/fixtures/i32_const.wat
(module
  (func $i32_const (result i32)
    (i32.const 42)
  )
  (export "i32_const" (func $i32_const))
)
```

```wat:src/fixtures/local_set.wat
(module
  (func $local_set (result i32)
    (local $x i32)
    (local.set $x (i32.const 42))
    (local.get 0)
  )
  (export "local_set" (func $local_set))
)
```

src/execution/runtime.rs
```diff
diff --git a/src/execution/runtime.rs b/src/execution/runtime.rs
index 9044e25..1b18d77 100644
--- a/src/execution/runtime.rs
+++ b/src/execution/runtime.rs
@@ -277,4 +277,22 @@ mod tests {
         assert!(result.is_err());
         Ok(())
     }
+    
+    #[test]
+    fn i32_const() -> Result<()> {
+        let wasm = wat::parse_file("src/fixtures/i32_const.wat")?;
+        let mut runtime = Runtime::instantiate(wasm)?;
+        let result = runtime.call("i32_const", vec![])?;
+        assert_eq!(result, Some(Value::I32(42)));
+        Ok(())
+    }
+
+    #[test]
+    fn local_set() -> Result<()> {
+        let wasm = wat::parse_file("src/fixtures/local_set.wat")?;
+        let mut runtime = Runtime::instantiate(wasm)?;
+        let result = runtime.call("local_set", vec![])?;
+        assert_eq!(result, Some(Value::I32(42)));
+        Ok(())
+    }
 }
```

### `i32.store`の実装

`i32.store`はスタックから値を`pop`して、指定したメモリアドレスに値を書き込む命令となっている。

src/binary/opcode.rs
```diff
diff --git a/src/binary/opcode.rs b/src/binary/opcode.rs
index 1eb29fd..1e0931b 100644
--- a/src/binary/opcode.rs
+++ b/src/binary/opcode.rs
@@ -5,6 +5,7 @@ pub enum Opcode {
     End = 0x0B,
     LocalGet = 0x20,
     LocalSet = 0x21,
+    I32Store = 0x36,
     I32Const = 0x41,
     I32Add = 0x6A,
     Call = 0x10,
```

src/binary/instruction.rs
```diff
diff --git a/src/binary/instruction.rs b/src/binary/instruction.rs
index 0e392c5..326db0a 100644
--- a/src/binary/instruction.rs
+++ b/src/binary/instruction.rs
@@ -3,6 +3,7 @@ pub enum Instruction {
     End,
     LocalGet(u32),
     LocalSet(u32),
+    I32Store { align: u32, offset: u32 },
     I32Const(i32),
     I32Add,
     Call(u32),
```

src/binary/module.rs
```diff
diff --git a/src/binary/module.rs b/src/binary/module.rs
index 26b56b3..f55db9b 100644
--- a/src/binary/module.rs
+++ b/src/binary/module.rs
@@ -221,6 +221,11 @@ fn decode_instructions(input: &[u8]) -> IResult<&[u8], Instruction> {
             let (rest, idx) = leb128_u32(input)?;
             (rest, Instruction::LocalSet(idx))
         }
+        Opcode::I32Store => {
+            let (rest, align) = leb128_u32(input)?;
+            let (rest, offset) = leb128_u32(rest)?;
+            (rest, Instruction::I32Store { align, offset })
+        }
         Opcode::I32Const => {
             let (rest, value) = leb128_i32(input)?;
             (rest, Instruction::I32Const(value))
```

src/execution/runtime.rs
```diff
diff --git a/src/execution/runtime.rs b/src/execution/runtime.rs
index bcb8288..b5b7417 100644
--- a/src/execution/runtime.rs
+++ b/src/execution/runtime.rs
@@ -183,6 +183,7 @@ impl Runtime {
                         }
                     }
                 }
+                _ => todo!(), // コンパイルエラーを回避するため命令処理は一旦TODO
             }
         }
         Ok(())
```

`i32.store`のオペランドはオフセットとアライメントの値になっていて、この`アドレス + オフセット`が実際に値を書き込む場所になる。
アライメントはメモリ境界チェックのために必要だが本書のスコープ外なので、デコードはするが特に使わない。

これで命令のデコードができるようになったので、テストを追加して実装が問題ないことを確認する。

src/execution/runtime.rs
```diff
diff --git a/src/binary/module.rs b/src/binary/module.rs
index f55db9b..c2a8c39 100644
--- a/src/binary/module.rs
+++ b/src/binary/module.rs
@@ -604,4 +604,35 @@ mod tests {
         }
         Ok(())
     }
+
+    #[test]
+    fn decode_i32_store() -> Result<()> {
+        let wasm = wat::parse_str(
+            "(module (func (i32.store offset=4 (i32.const 4))))",
+        )?;
+        let module = Module::new(&wasm)?;
+        assert_eq!(
+            module,
+            Module {
+                type_section: Some(vec![FuncType {
+                    params: vec![],
+                    results: vec![],
+                }]),
+                function_section: Some(vec![0]),
+                code_section: Some(vec![Function {
+                    locals: vec![],
+                    code: vec![
+                        Instruction::I32Const(4),
+                        Instruction::I32Store {
+                            align: 2,
+                            offset: 4
+                        },
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

```sh
running 18 tests
test binary::module::tests::decode_simplest_module ... ok
test binary::module::tests::decode_simplest_func ... ok
test binary::module::tests::decode_func_param ... ok
test binary::module::tests::decode_memory ... ok
test binary::module::tests::decode_func_add ... ok
test binary::module::tests::decode_i32_store ... ok
test binary::module::tests::decode_func_call ... ok
test binary::module::tests::decode_import ... ok
test binary::module::tests::decode_func_local ... ok
test binary::module::tests::decode_data ... ok
test execution::runtime::tests::call_imported_func ... ok
test execution::runtime::tests::execute_i32_add ... ok
test execution::runtime::tests::i32_const ... ok
test execution::runtime::tests::not_found_export_function ... ok
test execution::runtime::tests::func_call ... ok
test execution::runtime::tests::local_set ... ok
test execution::runtime::tests::not_found_imported_func ... ok
test execution::store::test::init_memory ... ok
```

続けて`i32.store`の命令を実装していく。

```diff
diff --git a/src/execution/value.rs b/src/execution/value.rs
index b24fd25..21d364d 100644
--- a/src/execution/value.rs
+++ b/src/execution/value.rs
@@ -10,6 +10,15 @@ impl From<i32> for Value {
     }
 }
 
+impl From<Value> for i32 {
+    fn from(value: Value) -> Self {
+        match value {
+            Value::I32(value) => value,
+            _ => panic!("type mismatch"),
+        }
+    }
+}
+
 impl From<i64> for Value {
     fn from(value: i64) -> Self {
         Value::I64(value)
```

```diff
diff --git a/src/execution/runtime.rs b/src/execution/runtime.rs
index b5b7417..3584fdf 100644
--- a/src/execution/runtime.rs
+++ b/src/execution/runtime.rs
@@ -1,3 +1,5 @@
+use std::mem::size_of;
+
 use super::{
     import::Import,
     store::{ExternalFuncInst, FuncInst, InternalFuncInst, Store},
@@ -161,6 +163,22 @@ impl Runtime {
                     let idx = *idx as usize;
                     frame.locals[idx] = value;
                 }
+                Instruction::I32Store { align: _, offset } => {
+                    let (Some(value), Some(addr)) = (self.stack.pop(), self.stack.pop()) else { // 1
+                        bail!("not found any value in the stack");
+                    };
+                    let addr = Into::<i32>::into(addr) as usize;
+                    let offset = (*offset) as usize;
+                    let at = addr + offset; // 2
+                    let end = at + size_of::<i32>(); // 3
+                    let memory = self
+                        .store
+                        .memories
+                        .get_mut(0)
+                        .ok_or(anyhow!("not found memory"))?;
+                    let value: i32 = value.into();
+                    memory.data[at..end].copy_from_slice(&value.to_le_bytes()); // 4
+                }
                 Instruction::I32Const(value) => self.stack.push(Value::I32(*value)),
                 Instruction::I32Add => {
                     let (Some(right), Some(left)) = (self.stack.pop(), self.stack.pop()) else {
```

命令の処理では次のことをやっている。

1. スタックからメモリに書き込む値とアドレスを取得
2. 1のアドレス + オフセットで書き込み先のインデックスを取得
3. メモリへの書き込み範囲を算出する
4. 値をリトルエンディアンのバイト列に変換して、2と3で計算した範囲にデータをコピー

最後にテストを追加して実装に問題ないことを確認する。

```wat:src/fixtures/i32_store.wat
(module
  (memory 1)
  (func $i32_store
    (i32.const 0)
    (i32.const 42)
    (i32.store)
  )
  (export "i32_store" (func $i32_store))
)
```

`i32_store.wat`ではアドレスが`0`で値が`42`、オフセットは`0`なので最終的にメモリの`0`番地に対して`42`を書き込むことになる。
そのためテストでは`0`番地が`42`になっていることを確認する。

src/execution/runtime.rs
```diff
diff --git a/src/execution/runtime.rs b/src/execution/runtime.rs
index 19b766c..79aa5e5 100644
--- a/src/execution/runtime.rs
+++ b/src/execution/runtime.rs
@@ -313,4 +313,14 @@ mod tests {
         assert_eq!(result, Some(Value::I32(42)));
         Ok(())
     }
+
+    #[test]
+    fn i32_store() -> Result<()> {
+        let wasm = wat::parse_file("src/fixtures/i32_store.wat")?;
+        let mut runtime = Runtime::instantiate(wasm)?;
+        runtime.call("i32_store", vec![])?;
+        let memory = &runtime.store.memories[0].data;
+        assert_eq!(memory[0], 42);
+        Ok(())
+    }
 }
```

```sh
running 19 tests
test binary::module::tests::decode_func_param ... ok
test binary::module::tests::decode_simplest_func ... ok
test binary::module::tests::decode_simplest_module ... ok
test binary::module::tests::decode_i32_store ... ok
test binary::module::tests::decode_import ... ok
test binary::module::tests::decode_func_call ... ok
test binary::module::tests::decode_func_local ... ok
test binary::module::tests::decode_func_add ... ok
test binary::module::tests::decode_memory ... ok
test binary::module::tests::decode_data ... ok
test execution::runtime::tests::execute_i32_add ... ok
test execution::runtime::tests::i32_const ... ok
test execution::runtime::tests::call_imported_func ... ok
test execution::runtime::tests::func_call ... ok
test execution::runtime::tests::i32_store ... ok
test execution::runtime::tests::local_set ... ok
test execution::runtime::tests::not_found_export_function ... ok
test execution::runtime::tests::not_found_imported_func ... ok
test execution::store::test::init_memory ... ok
```

## まとめ

本章では追加で必要な命令を実装した。
次章は`Wasm Runtime`でメモリを扱えるように、メモリ初期化の機能を実装していく。
