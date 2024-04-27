# Implementation of Runtime ~ Up to Function Execution ~

In this chapter, we will implement a Runtime to execute the following WAT.
Once implemented, a `Wasm Runtime` capable of simple addition will be created.

```wat
(module
  (func (param i32 i32) (result i32)
    (local.get 0)
    (local.get 1)
    i32.add
  )
)
```

The processing flow can be broadly divided as follows:

1. Generate an `execution::Store` using `binary::Module`
2. Generate an `execution::Runtime` using the `execution::Store`
3. Execute a function using `execution::Runtime::call(...)`

## Implementation of Values

The Wasm Runtime we are implementing this time will handle the following two types of values, so let's implement them.

- i32
- i64

First, create the following files under `src`.

- `src/execution/value.rs`
- `src/execution.rs`

Implement them as follows.

src/execution/value.rs
```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum Value {
    I32(i32),
    I64(i64),
}

impl From<i32> for Value {
    fn from(value: i32) -> Self {
        Value::I32(value)
    }
}

impl From<i64> for Value {
    fn from(value: i64) -> Self {
        Value::I64(value)
    }
}

impl std::ops::Add for Value {
    type Output = Self;
    fn add(self, rhs: Self) -> Self::Output {
        match (self, rhs) {
            (Value::I32(left), Value::I32(right)) => Value::I32(left + right),
            (Value::I64(left), Value::I64(right)) => Value::I64(left + right),
            _ => panic!("type mismatch"),
        }
    }
}
```

src/execution.rs
```rust
pub mod value;
```

src/execution/lib.rs
```diff
diff --git a/src/lib.rs b/src/lib.rs
index 96eab66..ec63376 100644
--- a/src/lib.rs
+++ b/src/lib.rs
@@ -1 +1,2 @@
 pub mod binary;
+pub mod execution;
```

Since `i32` and `i64` are different types, they cannot be handled together on the stack.
Therefore, we provide an Enum called `Value` to handle them on the stack.

Additionally, we implement `std::ops::Add` to allow adding `Value` instances together.

## Implementation of `Store`

[`Store`](https://www.w3.org/TR/wasm-core-1/#store%E2%91%A0) is a struct that holds the state of the `Wasm Runtime` during execution.
As defined in the specification, it holds information such as memory, imports, and functions, which are used to process instructions.
If a `Wasm` binary is considered a class, then `Store` can be thought of as an instance of that class.

At this point, it is sufficient to have information about functions, so create the following file to implement `Store`.

src/execution/store.rs
```rust
use crate::binary::{
    instruction::Instruction,
    types::{FuncType, ValueType},
};

#[derive(Clone)]
pub struct Func {
    pub locals: Vec<ValueType>,
    pub body: Vec<Instruction>,
}

#[derive(Clone)]
pub struct InternalFuncInst {
    pub func_type: FuncType,
    pub code: Func,
}

#[derive(Clone)]
pub enum FuncInst {
    Internal(InternalFuncInst),
}

#[derive(Default)]
pub struct Store {
    pub funcs: Vec<FuncInst>,
}
```

`FuncInst` represents the actual function processed by the `Wasm Runtime`.
There are imported functions and functions provided by the Wasm binary.
Since we are not using imported functions this time, we will first implement `InternalFuncInst` representing functions provided by the Wasm binary.

The fields of `InternalFuncInst` are as follows:

- `func_type`: Information about the function's signature (arguments and return values)
- `code`: A `Func` type that holds the function's local variable definitions and instruction sequences

Next, implement a function that takes a `binary::Module` and generates a `Store`.

src/execution/store.rs
```diff
diff --git a/src/execution/store.rs b/src/execution/store.rs
index e488383..5b4e467 100644
--- a/src/execution/store.rs
+++ b/src/execution/store.rs
@@ -1,7 +1,9 @@
 use crate::binary::{
     instruction::Instruction,
+    module::Module,
     types::{FuncType, ValueType},
 };
+use anyhow::{bail, Result};
 
 #[derive(Clone)]
 pub struct Func {
@@ -23,3 +25,44 @@ pub enum FuncInst {
 pub struct Store {
     pub funcs: Vec<FuncInst>,
 }
+
+impl Store {
+    pub fn new(module: Module) -> Result<Self> {
+        let func_type_idxs = match module.function_section {
+            Some(ref idxs) => idxs.clone(),
+            _ => vec![],
+        };
+
+        let mut funcs = vec![];
+
+        if let Some(ref code_section) = module.code_section {
+            for (func_body, type_idx) in code_section.iter().zip(func_type_idxs.into_iter()) {
+                let Some(ref func_types) = module.type_section else {
+                    bail!("not found type_section")
+                };
+
+                let Some(func_type) = func_types.get(type_idx as usize) else {
+                    bail!("not found func type in type_section")
+                };
+
+                let mut locals = Vec::with_capacity(func_body.locals.len());
+                for local in func_body.locals.iter() {
+                    for _ in 0..local.type_count {
+                        locals.push(local.value_type.clone());
+                    }
+                }
+
+                let func = FuncInst::Internal(InternalFuncInst {
+                    func_type: func_type.clone(),
+                    code: Func {
+                        locals,
+                        body: func_body.code.clone(),
+                    },
+                });
+                funcs.push(func);
+            }
+        }
+
+        Ok(Self { funcs })
+    }
+}
```

Each section used in the implementation contains the following data:

| Section            | Description                         |
|--------------------|-------------------------------------|
| `Type Section`     | Information about function signatures|
| `Code Section`     | Information about instructions for each function|
| `Function Section` | Reference information to function signatures|

To briefly explain what `Store::new()` is doing at this point, it is collecting the necessary information for executing functions scattered across each section.
It might be a bit confusing, so let's take a look at the `func_add.wat` and its `binary::Module`.

```rust
Module {
    type_section: Some(vec![FuncType {
        params: vec![ValueType::I32, ValueType::I32],
        results: vec![ValueType::I32],
    }]),
    function_section: Some(vec![0]),
    code_section: Some(vec![Function {
        locals: vec![],
        code: vec![
            Instruction::LocalGet(0),
            Instruction::LocalGet(1),
            Instruction::I32Add,
            Instruction::End
        ],
    }]),
    ..Default::default()
}
```

To know what signature the function in `code_section[0]` has, you need to retrieve the value of `function_section[0`.
This value corresponds to the index in `type_section`, so `type_section[0]` directly provides the signature information for `code_section[0`.
For example, if the value of `function_section[0]` is 1, then `type_section[1]` contains the signature information for `code_section[0`.

## Implementation of `Runtime`

`Runtime` is the `Wasm Runtime` itself, which is a structure that holds the following information:

- `Store`
- Stack
- Call stack

In the previous chapter, we explained about the stack and call stack, so now we will explain how to implement and use them.

First, add `src/execution/runtime.rs` to define `Runtime` and `Frame`.

```rust
use super::{store::Store, value::Value};
use crate::binary::{instruction::Instruction, module::Module};
use anyhow::Result;

#[derive(Default)]
pub struct Frame {
    pub pc: isize,               // プログラムカウンタ
    pub sp: usize,               // スタックポインタ
    pub insts: Vec<Instruction>, // 命令列
    pub arity: usize,            // 戻り値の数
    pub locals: Vec<Value>,      // ローカル変数
}

#[derive(Default)]
pub struct Runtime {
    pub store: Store,
    pub stack: Vec<Value>,
    pub call_stack: Vec<Frame>,
}

impl Runtime {
    pub fn instantiate(wasm: impl AsRef<[u8]>) -> Result<Self> {
        let module = Module::new(wasm.as_ref())?;
        let store = Store::new(module)?;
        Ok(Self {
            store,
            ..Default::default()
        })
    }
}
```

src/execution.rs
```diff
diff --git a/src/execution.rs b/src/execution.rs
index 1a50587..acbafa4 100644
--- a/src/execution.rs
+++ b/src/execution.rs
@@ -1,2 +1,3 @@
+pub mod runtime;
 pub mod store;
 pub mod value;
```

`Runtime::instantiate(...)` is a function that takes a Wasm binary and generates a `Runtime`.

### Implementation of Instruction Processing

Next, implement `Runtime::execute(...)` to execute instructions.

src/execution/runtime.rs
```diff
diff --git a/src/execution/runtime.rs b/src/execution/runtime.rs
index c45d764..9db8415 100644
--- a/src/execution/runtime.rs
+++ b/src/execution/runtime.rs
@@ -1,6 +1,6 @@
 use super::{store::Store, value::Value};
 use crate::binary::{instruction::Instruction, module::Module};
-use anyhow::Result;
+use anyhow::{bail, Result};
 
 #[derive(Default)]
 pub struct Frame {
@@ -27,4 +27,38 @@ impl Runtime {
             ..Default::default()
         })
     }
+
+    fn execute(&mut self) -> Result<()> {
+        loop {
+            let Some(frame) = self.call_stack.last_mut() else { // 1
+                break;
+            };
+
+            frame.pc += 1;
+
+            let Some(inst) = frame.insts.get(frame.pc as usize) else { // 2
+                break;
+            };
+
+            match inst { // 3
+                Instruction::I32Add => {
+                    let (Some(right), Some(left)) = (self.stack.pop(), self.stack.pop()) else {
+                        bail!("not found any value in the stack");
+                    };
+                    let result = left + right;
+                    self.stack.push(result);
+                }
+            }
+        }
+        Ok(())
+    }
 }
```

The `execute(...)` method loops until there are no more instructions pointed to by the program counter.

1. Get the top frame from the call stack.
2. Increment the program counter (pc) and get the next instruction from the frame.
3. Process each instruction:
   For an instruction like `i32.add`, pop two values from the stack, add them, and push the result back onto the stack.

The process is almost the same as the pseudocode shown in the previous chapter.

Next, implement the `local.get` and `end` instructions.

src/execution/runtime.rs
```diff
diff --git a/src/execution/runtime.rs b/src/execution/runtime.rs
index 6d090e9..5bae7fb 100644
--- a/src/execution/runtime.rs
+++ b/src/execution/runtime.rs
@@ -41,6 +41,12 @@ impl Runtime {
             };
 
             match inst {
+                Instruction::LocalGet(idx) => {
+                    let Some(value) = frame.locals.get(*idx as usize) else {
+                        bail!("not found local");
+                    };
+                    self.stack.push(*value);
+                }
                 Instruction::I32Add => {
                     let (Some(right), Some(left)) = (self.stack.pop(), self.stack.pop()) else {
                         bail!("not found any value in the stack");
```

The `local.get` instruction retrieves the value of a local variable and pushes it onto the stack.
`local.get` has an operand that indicates which local variable's value to retrieve, using the index value in `frame.locals`.

src/execution/runtime.rs
```diff
diff --git a/src/execution/runtime.rs b/src/execution/runtime.rs
index 5bae7fb..ceaf3dc 100644
--- a/src/execution/runtime.rs
+++ b/src/execution/runtime.rs
@@ -41,6 +41,13 @@ impl Runtime {
             };
 
             match inst {
+                Instruction::End => {
+                    let Some(frame) = self.call_stack.pop() else { // 1
+                        bail!("not found frame");
+                    };
+                    let Frame { sp, arity, .. } = frame; // 2
+                    stack_unwind(&mut self.stack, sp, arity)?; // 3
+                }
                 Instruction::LocalGet(idx) => {
                     let Some(value) = frame.locals.get(*idx as usize) else {
                         bail!("not found local");
@@ -59,3 +66,16 @@ impl Runtime {
         Ok(())
     }
 }
+
+pub fn stack_unwind(stack: &mut Vec<Value>, sp: usize, arity: usize) -> Result<()> {
+    if arity > 0 { // 3-1
+        let Some(value) = stack.pop() else {
+            bail!("not found return value");
+        };
+        stack.drain(sp..);
+        stack.push(value); // 3-2
+    } else {
+        stack.drain(sp..); // 3-3
+    }
+    Ok(())
+}
```

The `end` instruction signifies the end of function execution and performs the following steps:

1. Pop the frame from the call stack.
2. Retrieve the stack pointer (sp) and arity (number of return values) from the frame information.
3. Rewind the stack:
   - If there are return values, pop one value from the stack and rewind the stack to sp.
   - Push the popped value back onto the stack.
   - If there are no return values, simply rewind the stack to sp.

With the base implementation of `Runtime::execute(...)` complete, the next step is to implement `Runtime::invoke_internal(...)` for pre and post-processing of instruction execution.

src/execution/runtime.rs
```diff
diff --git a/src/execution/runtime.rs b/src/execution/runtime.rs
index ceaf3dc..3356b37 100644
--- a/src/execution/runtime.rs
+++ b/src/execution/runtime.rs
@@ -1,5 +1,8 @@
-use super::{store::Store, value::Value};
-use crate::binary::{instruction::Instruction, module::Module};
+use super::{
+    store::{InternalFuncInst, Store},
+    value::Value,
+};
+use crate::binary::{instruction::Instruction, module::Module, types::ValueType};
 use anyhow::{bail, Result};
 
 #[derive(Default)]
@@ -28,6 +31,43 @@ impl Runtime {
         })
     }
 
+    fn invoke_internal(&mut self, func: InternalFuncInst) -> Result<Option<Value>> {
+        let bottom = self.stack.len() - func.func_type.params.len(); // 1
+        let mut locals = self.stack.split_off(bottom); // 2
+
+        for local in func.code.locals.iter() { // 3
+            match local {
+                ValueType::I32 => locals.push(Value::I32(0)),
+                ValueType::I64 => locals.push(Value::I64(0)),
+            }
+        }
+
+        let arity = func.func_type.results.len(); // 4
+
+        let frame = Frame {
+            pc: -1,
+            sp: self.stack.len(),
+            insts: func.code.body.clone(),
+            arity,
+            locals,
+        };
+
+        self.call_stack.push(frame); // 5
+
+        if let Err(e) = self.execute() { // 6
+            self.cleanup();
+            bail!("failed to execute instructions: {}", e)
+        };
+
+        if arity > 0 { // 7
+            let Some(value) = self.stack.pop() else {
+                bail!("not found return value")
+            };
+            return Ok(Some(value));
+        }
+        Ok(None)
+    }
+
     fn execute(&mut self) -> Result<()> {
         loop {
             let Some(frame) = self.call_stack.last_mut() else {
@@ -65,6 +105,11 @@ impl Runtime {
         }
         Ok(())
     }
+
+    fn cleanup(&mut self) {
+        self.stack = vec![];
+        self.call_stack = vec![];
+    }
 }
 
 pub fn stack_unwind(stack: &mut Vec<Value>, sp: usize, arity: usize) -> Result<()> {
```

In `Runtime::invoke_internal(...)`, the following steps are performed:

1. Get the number of function arguments.
2. Pop values from the stack for each argument.
3. Initialize local variables.
4. Get the number of function return values.
5. Create a frame and push it onto `Runtime::call_stack`.
6. Call `Runtime::execute()` to execute the function.
7. If there are return values, pop them from the stack and return them; otherwise, return `None`.

Next, implement `Runtime::call(...)` which calls `Runtime::invoke_internal(...)` and allows specifying the function to call and passing function arguments.

src/execution/runtime.rs
```diff
diff --git a/src/execution/runtime.rs b/src/execution/runtime.rs
index 3356b37..7cba836 100644
--- a/src/execution/runtime.rs
+++ b/src/execution/runtime.rs
@@ -1,5 +1,5 @@
 use super::{
-    store::{InternalFuncInst, Store},
+    store::{FuncInst, InternalFuncInst, Store},
     value::Value,
 };
 use crate::binary::{instruction::Instruction, module::Module, types::ValueType};
@@ -31,6 +31,18 @@ impl Runtime {
         })
     }
 
+    pub fn call(&mut self, idx: usize, args: Vec<Value>) -> Result<Option<Value>> {
+        let Some(func_inst) = self.store.funcs.get(idx) else { // 1
+            bail!("not found func")
+        };
+        for arg in args { // 2
+            self.stack.push(arg);
+        }
+        match func_inst {
+            FuncInst::Internal(func) => self.invoke_internal(func.clone()), // 3
+        }
+    }
+
     fn invoke_internal(&mut self, func: InternalFuncInst) -> Result<Option<Value>> {
         let bottom = self.stack.len() - func.func_type.params.len();
         let mut locals = self.stack.split_off(bottom);
```

In `Runtime::call(...)`, the following steps are performed:

1. Get the `InternalFuncInst` (function entity) held by `Store` using the specified index.
2. Push arguments onto the stack.
3. Pass the `InternalFuncInst` obtained in step 1 to `Runtime::invoke_internal(...)` for execution and return the result.

With this, a `Wasm Runtime` capable of executing addition functions has been created. Finally, write tests to ensure it functions correctly. 

{/*examples*/}

src/execution/runtime.rs
```diff
diff --git a/src/execution/runtime.rs b/src/execution/runtime.rs
index 7cba836..7fa35d2 100644
--- a/src/execution/runtime.rs
+++ b/src/execution/runtime.rs
@@ -136,3 +136,24 @@ pub fn stack_unwind(stack: &mut Vec<Value>, sp: usize, arity: usize) -> Result<(
     }
     Ok(())
 }
+
+#[cfg(test)]
+mod tests {
+    use super::Runtime;
+    use crate::execution::value::Value;
+    use anyhow::Result;
+
+    #[test]
+    fn execute_i32_add() -> Result<()> {
+        let wasm = wat::parse_file("src/fixtures/func_add.wat")?;
+        let mut runtime = Runtime::instantiate(wasm)?;
+        let tests = vec![(2, 3, 5), (10, 5, 15), (1, 1, 2)];
+
+        for (left, right, want) in tests {
+            let args = vec![Value::I32(left), Value::I32(right)];
+            let result = runtime.call(0, args)?;
+            assert_eq!(result, Some(Value::I32(want)));
+        }
+        Ok(())
+    }
+}
```

If there are no issues, the test should pass as follows.

```sh
running 6 tests
test binary::module::tests::decode_simplest_module ... ok
test binary::module::tests::decode_func_param ... ok
test binary::module::tests::decode_func_local ... ok
test binary::module::tests::decode_simplest_func ... ok
test binary::module::tests::decode_func_add ... ok
test execution::runtime::tests::execute_i32_add ... ok
```

## Summary
In this chapter, we implemented a `Runtime` that can perform addition and confirmed that it works. Now that the template is ready, we will further expand in the next chapters and implement function calls. 


