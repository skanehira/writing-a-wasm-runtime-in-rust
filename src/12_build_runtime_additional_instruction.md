# Implementation additional instructions

In this chapter, we need the following instructions for the features we are about to implement, so let's implement them first.

| Instruction | Description                                                   |
|-------------|---------------------------------------------------------------|
| `local.set` | `pop` a value from the stack and place it in a local variable |
| `i32.const` | `push` the value of the operand onto the stack                |
| `i32.store` | `pop` a value from the stack and write it to memory           |

### Implementation of `local.set`

The `local.set` instruction has an operand that allows specifying where to place the value in a local variable.

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

src/execution/runtime.rs
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

The operation is simple, placing the `pop` value at the specified index of `frame.locals`.

### Implementation of `i32.const`

`i32.const` has an operand with a value that is pushed onto the stack.

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

Add tests combining with `local.set` to ensure the implementation is correct.

src/fixtures/i32_const.wat
```wat
(module
  (func $i32_const (result i32)
    (i32.const 42)
  )
  (export "i32_const" (func $i32_const))
)
```

src/fixtures/local_set.wat
```wat
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

### Implementation of `i32.store` decoding

`i32.store` is an instruction that `pop` a value from the stack and writes it to the specified memory address.

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
+                _ => todo!(), // Instruction processing is set to TODO to avoid compilation errors
             }
         }
         Ok(())
```

The operand of `i32.store` consists of an offset and alignment value, where `address + offset` becomes the actual location to write the value.
Alignment is necessary for memory boundary checks, but it is out of the scope of this document, so we decode it but do not use it specifically.

With the decoding of the instruction in place, add tests to ensure the implementation is correct.

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

Memory is required to implement the `i32.store` instruction, so we will implement the instruction along with memory initialization in the next chapter.

## Summary

In this chapter, additional instructions were implemented. In the next chapter, we will implement memory initialization functionality in the `Wasm Runtime` to handle memory.
