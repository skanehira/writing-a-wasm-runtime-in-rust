# Extra Edition ~ Until the Fibonacci Function Works ~

In this chapter, we will add additional implementations to make the Fibonacci function executable as an extra edition.
For those who find the output of `Hello, World!` a bit unsatisfying, this will be a perfect dessert.

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

## About Binary Structure

We will explain what the binary structure looks like when there are flow control instructions like `if`.
The following is the binary structure of the function part.

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

In the above, the following instructions are used for the unimplemented part.

| Instruction | Description                                      |
|-------------|--------------------------------------------------|
| `i32.sub`   | Pop two values from the stack and subtract them  |
| `i32.lt_s`  | Pop two values from the stack and compare with `<`|
| `if`        | Pop one value from the stack, evaluate, and process branching |
| `return`    | Return to the caller                             |

Let's first look at the instruction sequence for the `if` part.

```wat
(if
  (i32.lt_s (local.get $n) (i32.const 2))
  (return (i32.const 1))
)
```

The instruction sequence is as follows.

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

It pushes the argument and `2` onto the stack, uses `i32.lt_s` to compare with `<`, and pushes the result onto the stack.
By the way, in `Wasm`, if the comparison result is `0`, it is `false`; otherwise, it is `true`.

Next, after popping the comparison result from the stack, if it is `true`, it proceeds; if it is `false`, it jumps to `end`.
Following `if` is `void`, but in `Wasm`, `if` can return a value, and `void` means no return value.
This part becomes the operand of the `if` instruction.

In this case, if it is `true`, it pushes `1` onto the stack and returns to the caller using `return`.

Next, let's look at the part after `return`.

```wat
(return
  (i32.add
    (call $fib (i32.sub (local.get $n) (i32.const 2)))
    (call $fib (i32.sub (local.get $n) (i32.const 1)))
  )
)
```

The instruction sequence is as follows.

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

It might be a bit hard to understand, so to clarify which instruction corresponds to which operation, it looks like this.

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

## Instruction Decoding Implementation

Now that we understand the binary structure, let's implement it.

First, add the opcode.

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

Next, define the `Block` used in the `if` instruction.

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

In `Wasm`, the `if` instruction represents the number of return values with `Block`, where `Block::Void` means no return value, and `Block::Value` means there is a return value.
Since there are no return values in this case, it will mostly be `Block::Void`.

Additionally, besides the `If` instruction, there are also `block` and `loop` instructions, which similarly have operands for return values, so `Block` is used for those as well.

Next, define the instructions.

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

`decode_block(...)` is a function that generates `Block`, and if the next after `if` is `0x40`, it means no return value, otherwise it becomes the value of the number of return values.
In version 1, since only one return value can be returned, it effectively becomes `1`.

For the part of implementing instructions, let's leave it as TODO for now.

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

Now you can write tests for decoding, ensuring that the implementation is correct.

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

## Implementation of Commands

Now that the decoding process is complete, let's move on to implementing the commands.
We will start by implementing the simplest command, `i32.sub`.

### Implementation of `i32.sub`
First, we will implement `std::ops::Sub` to enable subtraction between `Value` instances.

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

Add tests to ensure the implementation is correct.

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

### Implementation of `i32.lt_s`

The `lt_s` command compares two `i32` values using `<`. We will first implement `std::cmp::Ordering` to enable this comparison. Additionally, since the result of the comparison will be a `bool`, we will implement `From<bool> for Value` to push this result onto the stack.

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

Next, we will implement the command and add tests to ensure the implementation is correct.

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

### Implementation of `if`/`return`

Next, we will implement flow control commands for `if`.

First, we will add a `Label`. `Label` is used for controlling blocks that have return values, such as `if` or `block`.

For example, in the case of `if`, there may be an `else` branch (which we are not implementing in this case). If the evaluation result is `false`, we need to jump to the `else` command, and this behavior is achieved using `Label`.

Additionally, when exiting the `if` block, we need to rewind the stack, which is also achieved using `Label`.

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

Next, we will proceed with the command processing implementation.

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

In the `if` command, we are doing the following:

1. Pop a value from the stack for evaluation.
2. If it is `false`, calculate the end `pc` of the `if` command.
    - If it is `true`, there is no need to adjust the `pc`; we simply push the label and continue processing the next command.
3. Since `if` commands can be nested, we control this with `depth`.

It is worth noting that the end of an `if` command is similar to the end of a function, marked by the `end` command. This is also the case for `block` and `loop` commands.

Therefore, when processing the `end` command, we need to determine whether it is the end of an `if`/`block`/`loop` or the end of a function. This determination is simple: if the label stack is empty, it signifies the end of a function.

The actual implementation is as follows.

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

If a label exists, it signifies the end of an `if` command, and we adjust the `pc` and stack accordingly. The reason for adjusting the stack is that there may be values left on the stack from processing within the `if` block.

For example, if the `if` evaluates to `true` and only pushes `3` onto the stack, there will be values left on the stack that need to be rewound.

```wat
(if
  (i32.const 1)
  (i32.const 3)
)
```

With this, the implementation for running the Fibonacci function is complete. Now, let's add tests to ensure that the implementation is correct.

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

## Summary

With this, a `Wasm Runtime` capable of running the Fibonacci function has been created.
The implementation of control instructions was quite interesting, wasn't it?

If you have any requests such as wanting to run something like this, please feel free to write it in an issue. I would be happy to consider writing about it when I feel like it.
