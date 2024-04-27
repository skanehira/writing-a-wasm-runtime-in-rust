# Implementation function call

In this chapter, we will extend the implementation from the previous chapter to implement the following features.

- Execute only exported functions
- Call functions

## Execute only exported functions

In the previous chapter, we specified the function to be executed by index.

```rust
let args = vec![Value::I32(left), Value::I32(right)];
let result = runtime.call(0, args)?;
assert_eq!(result, Some(Value::I32(want)));
```

It is very inconvenient because you cannot specify the function you want to execute without analyzing the binary structure.
Also, according to the `Wasm spec`, it is specified that only exported functions can be executed, but the current implementation does not meet this specification.
Therefore, in this section, we will enable executing functions by specifying the function name.

### Implementation of decoding the `Export Section`
Modify `func_add.wat` as follows to export a function named `add`.

src/fixtures/func_add.wat
```diff
diff --git a/src/fixtures/func_add.wat b/src/fixtures/func_add.wat
index ce14757..99678c4 100644
--- a/src/fixtures/func_add.wat
+++ b/src/fixtures/func_add.wat
@@ -1,5 +1,5 @@
 (module
-  (func (param i32 i32) (result i32)
+  (func (export "add") (param i32 i32) (result i32) 
     (local.get 0)
     (local.get 1)
     i32.add
```

Since the decoding of the `Export Section` has not been implemented, the test currently fails, so let's implement it.
Regarding the binary structure, it has already been explained in the chapter [Wasm Binary Structure](https://zenn.dev/skanehira/books/writing-wasm-runtime-in-rust/viewer/04_wasm_binary_structure), so please refer to it.

First, define the type representing exports.

src/binary/types.rs
```diff
diff --git a/src/binary/types.rs b/src/binary/types.rs
index 7707a97..191c34b 100644
--- a/src/binary/types.rs
+++ b/src/binary/types.rs
@@ -25,3 +25,14 @@ pub struct FunctionLocal {
     pub type_count: u32,
     pub value_type: ValueType,
 }
+
+#[derive(Debug, Clone, PartialEq, Eq)]
+pub enum ExportDesc {
+    Func(u32),
+}
+
+#[derive(Debug, PartialEq, Eq)]
+pub struct Export {
+    pub name: String,
+    pub desc: ExportDesc,
+}
```

`Export::name` is the export name, which will be `add` in this case.
`ExportDesc` is a reference to the entity such as a function or memory, and for functions, it will be the index value of `Store::funcs`.
For example, in the case of `ExportDesc::Func(0)`, the entity of the function named `add` will be `Store::funcs[0]`.
In this book, we will not implement exporting memories, so we will only implement `ExportDesc::Func`.

Next, implement the decoding of the `Export Section`.

src/binary/module.rs
```diff
diff --git a/src/binary/module.rs b/src/binary/module.rs
index 5ea23a1..949aea9 100644
--- a/src/binary/module.rs
+++ b/src/binary/module.rs
@@ -2,7 +2,7 @@ use super::{
     instruction::Instruction,
     opcode::Opcode,
     section::{Function, SectionCode},
-    types::{FuncType, FunctionLocal, ValueType},
+    types::{Export, ExportDesc, FuncType, FunctionLocal, ValueType},
 };
 use nom::{
     bytes::complete::{tag, take},
@@ -194,6 +194,27 @@ fn decode_instructions(input: &[u8]) -> IResult<&[u8], Instruction> {
     Ok((rest, inst))
 }
 
+fn decode_export_section(input: &[u8]) -> IResult<&[u8], Vec<Export>> {
+    let (mut input, count) = leb128_u32(input)?; // 1
+    let mut exports = vec![];
+
+    for _ in 0..count { // 9
+        let (rest, name_len) = leb128_u32(input)?; // 2
+        let (rest, name_bytes) = take(name_len)(rest)?; // 3
+        let name = String::from_utf8(name_bytes.to_vec()).expect("invalid utf-8 string"); // 4
+        let (rest, export_kind) = le_u8(rest)?; // 5
+        let (rest, idx) = leb128_u32(rest)?; // 6
+        let desc = match export_kind { // 7
+            0x00 => ExportDesc::Func(idx),
+            _ => unimplemented!("unsupported export kind: {:X}", export_kind),
+        };
+        exports.push(Export { name, desc }); // 8
+        input = rest;
+    }
+
+    Ok((input, exports))
+}
+
 #[cfg(test)]
 mod tests {
     use crate::binary::{
```

In `decode_export_section(...)`, the following steps are performed:

1. Obtain the number of exports
   In this case, there is only one exported function, so it will be 1.
2. Obtain the length of the byte sequence for the export name
3. Obtain the byte sequence for the length obtained in step 2
4. Convert the byte sequence obtained in step 3 to a string
5. Obtain the type of export (function, memory, etc.)
6. Obtain a reference to the entity exported, such as a function
7. If it is `0x00`, the export type is a function, so generate an `Export`
8. Add the `Export` generated in step 7 to an array
9. Repeat steps 2 to 8 for the number of elements

With this, the decoding of the `Export Section` has been implemented. Next, add `export_section` to `Module` as follows.

src/binary/module.rs
```diff
diff --git a/src/binary/module.rs b/src/binary/module.rs
index d14704f..41c9a89 100644
--- a/src/binary/module.rs
+++ b/src/binary/module.rs
@@ -2,7 +2,7 @@ use super::{
     instruction::Instruction,
     opcode::Opcode,
     section::{Function, SectionCode},
-    types::{FuncType, FunctionLocal, ValueType},
+    types::{Export, ExportDesc, FuncType, FunctionLocal, ValueType},
 };
 use nom::{
     bytes::complete::{tag, take},
@@ -21,6 +21,7 @@ pub struct Module {
     pub type_section: Option<Vec<FuncType>>,
     pub function_section: Option<Vec<u32>>,
     pub code_section: Option<Vec<Function>>,
+    pub export_section: Option<Vec<Export>>,
 }
 
 impl Default for Module {
@@ -31,6 +32,7 @@ impl Default for Module {
             type_section: None,
             function_section: None,
             code_section: None,
+            export_section: None,
         }
     }
 }
@@ -72,6 +74,10 @@ impl Module {
                             let (_, funcs) = decode_code_section(section_contents)?;
                             module.code_section = Some(funcs);
                         }
+                        SectionCode::Export => {
+                            let (_, exports) = decode_export_section(section_contents)?;
+                            module.export_section = Some(exports);
+                        }
                         _ => todo!(),
                     };
 
@@ -221,7 +227,7 @@ mod tests {
         instruction::Instruction,
         module::Module,
         section::Function,
-        types::{FuncType, FunctionLocal, ValueType},
+        types::{Export, ExportDesc, FuncType, FunctionLocal, ValueType},
     };
     use anyhow::Result;
```

Also, update the tests.

src/
```diff
diff --git a/src/binary/module.rs b/src/binary/module.rs
index 41c9a89..9ba5afc 100644
--- a/src/binary/module.rs
+++ b/src/binary/module.rs
@@ -329,6 +329,10 @@ mod tests {
                         Instruction::End
                     ],
                 }]),
+                export_section: Some(vec![Export {
+                    name: "add".into(),
+                    desc: ExportDesc::Func(0),
+                }]),
                 ..Default::default()
             }
         );
```

Now the tests should pass.

```sh
running 6 tests
test binary::module::tests::decode_simplest_module ... ok
test binary::module::tests::decode_simplest_func ... ok
test binary::module::tests::decode_func_local ... ok
test binary::module::tests::decode_func_param ... ok
test execution::runtime::tests::execute_i32_add ... ok
test binary::module::tests::decode_func_add ... ok
```

### Implementation of function execution

Now that the section decoding is done, let's implement the ability to execute functions by specifying the function name.

First, define `ModuleInst` that holds `ExportInst` (export information).
By making `ModuleInst::exports` a `HashMap`, it becomes easy to look up function export information from the function name.

src/execution/store.rs
```diff
diff --git a/src/execution/store.rs b/src/execution/store.rs
index 2cd9821..d103fa0 100644
--- a/src/execution/store.rs
+++ b/src/execution/store.rs
@@ -1,7 +1,9 @@
+use std::collections::HashMap;
+
 use crate::binary::{
     instruction::Instruction,
     module::Module,
-    types::{FuncType, ValueType},
+    types::{ExportDesc, FuncType, ValueType},
 };
 use anyhow::{bail, Result};
 
@@ -21,9 +23,20 @@ pub enum FuncInst {
     Internal(InternalFuncInst),
 }
 
+pub struct ExportInst {
+    pub name: String,
+    pub desc: ExportDesc,
+}
+
+#[derive(Default)]
+pub struct ModuleInst {
+    pub exports: HashMap<String, ExportInst>,
+}
+
 #[derive(Default)]
 pub struct Store {
     pub funcs: Vec<FuncInst>,
+    pub module: ModuleInst,
 }
 
 impl Store {
```

Next, generate `ModuleInst` from `Module::export_section`.

src/execution/store.rs
```diff
diff --git a/src/execution/store.rs b/src/execution/store.rs
index d103fa0..3f6ecb2 100644
--- a/src/execution/store.rs
+++ b/src/execution/store.rs
@@ -76,6 +76,22 @@ impl Store {
             }
         }
 
-        Ok(Self { funcs })
+        let mut exports = HashMap::default();
+        if let Some(ref sections) = module.export_section {
+            for export in sections {
+                let name = export.name.clone();
+                let export_inst = ExportInst {
+                    name: name.clone(),
+                    desc: export.desc.clone(),
+                };
+                exports.insert(name, export_inst);
+            }
+        };
+        let module_inst = ModuleInst { exports };
+
+        Ok(Self {
+            funcs,
+            module: module_inst,
+        })
     }
 }
```

```markdown
Continuing, modify to receive the function name in `Runtime::call(...)` and update to retrieve the index from `ModuleInst` to the function.
Also, update the tests to pass the function name.

src/execution/runtime.rs
```diff
diff --git a/src/execution/runtime.rs b/src/execution/runtime.rs
index 4fe757a..1885646 100644
--- a/src/execution/runtime.rs
+++ b/src/execution/runtime.rs
@@ -2,8 +2,12 @@ use super::{
     store::{FuncInst, InternalFuncInst, Store},
     value::Value,
 };
-use crate::binary::{instruction::Instruction, module::Module, types::ValueType};
-use anyhow::{bail, Result};
+use crate::binary::{
+    instruction::Instruction,
+    module::Module,
+    types::{ExportDesc, ValueType},
+};
+use anyhow::{anyhow, bail, Result};
 
 #[derive(Default)]
 pub struct Frame {
@@ -31,7 +35,17 @@ impl Runtime {
         })
     }
 
-    pub fn call(&mut self, idx: usize, args: Vec<Value>) -> Result<Option<Value>> {
+    pub fn call(&mut self, name: impl Into<String>, args: Vec<Value>) -> Result<Option<Value>> {
+        let idx = match self
+            .store
+            .module
+            .exports
+            .get(&name.into())
+            .ok_or(anyhow!("not found export function"))?
+            .desc
+        {
+            ExportDesc::Func(idx) => idx as usize,
+        };
         let Some(func_inst) = self.store.funcs.get(idx) else {
             bail!("not found func")
         };
@@ -151,7 +165,7 @@ mod tests {
 
         for (left, right, want) in tests {
             let args = vec![Value::I32(left), Value::I32(right)];
-            let result = runtime.call(0, args)?;
+            let result = runtime.call("add", args)?;
             assert_eq!(result, Some(Value::I32(want)));
         }
         Ok(())
```

With this change, the following test will pass.

```sh
running 6 tests
test binary::module::tests::decode_simplest_func ... ok
test binary::module::tests::decode_func_param ... ok
test binary::module::tests::decode_func_local ... ok
test binary::module::tests::decode_simplest_module ... ok
test binary::module::tests::decode_func_add ... ok
test execution::runtime::tests::execute_i32_add ... ok
```

Additionally, add a test for specifying a non-existent function to verify.

src/execution/runtime.rs
```diff
diff --git a/src/execution/runtime.rs b/src/execution/runtime.rs
index 1885646..acc48cf 100644
--- a/src/execution/runtime.rs
+++ b/src/execution/runtime.rs
@@ -170,4 +170,13 @@ mod tests {
         }
         Ok(())
     }
+
+    #[test]
+    fn not_found_export_function() -> Result<()> {
+        let wasm = wat::parse_file("src/fixtures/func_add.wat")?;
+        let mut runtime = Runtime::instantiate(wasm)?;
+        let result = runtime.call("fooooo", vec![]);
+        assert!(result.is_err());
+        Ok(())
+    }
 }
```

```sh
running 1 test
test execution::runtime::tests::not_found_export_function ... ok
```

## Implementation of Function Calls

In `Wasm`, functions can call other functions. Of course, it is also possible for a function to call itself recursively.

In this section, we will implement the ability to make the following function calls. The `call_doubler` function takes one argument, passes it to the `double` function, and returns double the value.

```wat:src/fixtures/func_call.wat
(module
  (func (export "call_doubler") (param i32) (result i32) 
    (local.get 0)
    (call $double)
  )
  (func $double (param i32) (result i32)
    (local.get 0)
    (local.get 0)
    i32.add
  )
)
```

### Implementation of `call` Instruction Decoding

First, implement the decoding of function call instructions.

src/binary/opcode.rs
```diff
diff --git a/src/binary/opcode.rs b/src/binary/opcode.rs
index 98c075e..5d0a2b7 100644
--- a/src/binary/opcode.rs
+++ b/src/binary/opcode.rs
@@ -5,4 +5,5 @@ pub enum Opcode {
     End = 0x0B,
     LocalGet = 0x20,
     I32Add = 0x6A,
+    Call = 0x10,
 }
```

src/binary/instruction.rs
```diff
diff --git a/src/binary/instruction.rs b/src/binary/instruction.rs
index 1307caa..c9c6584 100644
--- a/src/binary/instruction.rs
+++ b/src/binary/instruction.rs
@@ -3,4 +3,5 @@ pub enum Instruction {
     End,
     LocalGet(u32),
     I32Add,
+    Call(u32),
 }
```

The function call instruction has an operand that holds a reference (index) to the function, so decode that.

src/binary/module.rs
```diff
diff --git a/src/binary/module.rs b/src/binary/module.rs
index 9ba5afc..3a3316c 100644
--- a/src/binary/module.rs
+++ b/src/binary/module.rs
@@ -196,6 +196,10 @@ fn decode_instructions(input: &[u8]) -> IResult<&[u8], Instruction> {
         }
         Opcode::I32Add => (input, Instruction::I32Add),
         Opcode::End => (input, Instruction::End),
+        Opcode::Call => {
+            let (rest, idx) = leb128_u32(input)?;
+            (rest, Instruction::Call(idx))
+        }
     };
     Ok((rest, inst))
 }
```

Next, skip decoding the custom sections and add tests.

src/binary/section.rs
```diff
diff --git a/src/binary/section.rs b/src/binary/section.rs
index a9a11b3..44e9884 100644
--- a/src/binary/section.rs
+++ b/src/binary/section.rs
@@ -3,6 +3,7 @@ use num_derive::FromPrimitive;
 
 #[derive(Debug, PartialEq, Eq, FromPrimitive)]
 pub enum SectionCode {
+    Custom = 0x00,
     Type = 0x01,
     Import = 0x02,
     Function = 0x03,
```

src/execution/runtime.rs
```diff
diff --git a/src/execution/runtime.rs b/src/execution/runtime.rs
index acc48cf..bc2a20b 100644
--- a/src/execution/runtime.rs
+++ b/src/execution/runtime.rs
@@ -127,6 +127,7 @@ impl Runtime {
                     let result = left + right;
                     self.stack.push(result);
                 }
+                _ => todo!(),
             }
         }
         Ok(())
```

src/binary/module.rs
```diff
diff --git a/src/binary/module.rs b/src/binary/module.rs
index 3a3316c..5bf739d 100644
--- a/src/binary/module.rs
+++ b/src/binary/module.rs
@@ -62,6 +62,9 @@ impl Module {
                     let (rest, section_contents) = take(size)(input)?;
 
                     match code {
+                        SectionCode::Custom => {
+                            // skip
+                        }
                         SectionCode::Type => {
                             let (_, types) = decode_type_section(section_contents)?;
                             module.type_section = Some(types);
@@ -342,4 +345,45 @@ mod tests {
         );
         Ok(())
     }
+
+    #[test]
+    fn decode_func_call() -> Result<()> {
+        let wasm = wat::parse_file("src/fixtures/func_call.wat")?;
+        let module = Module::new(&wasm)?;
+        assert_eq!(
+            module,
+            Module {
+                type_section: Some(vec![FuncType {
+                    params: vec![ValueType::I32],
+                    results: vec![ValueType::I32],
+                },]),
+                function_section: Some(vec![0, 0]),
+                code_section: Some(vec![
+                    Function {
+                        locals: vec![],
+                        code: vec![
+                            Instruction::LocalGet(0),
+                            Instruction::Call(1),
+                            Instruction::End
+                        ],
+                    },
+                    Function {
+                        locals: vec![],
+                        code: vec![
+                            Instruction::LocalGet(0),
+                            Instruction::LocalGet(0),
+                            Instruction::I32Add,
+                            Instruction::End
+                        ],
+                    }
+                ]),
+                export_section: Some(vec![Export {
+                    name: "call_doubler".into(),
+                    desc: ExportDesc::Func(0),
+                }]),
+                ..Default::default()
+            }
+        );
+        Ok(())
+    }
 }
```

Custom sections refer to the metadata area where you can freely place any data. Although not specifically used in this book, when using the `wat` crate with WAT, it adds custom sections during compilation, so it is necessary to skip them.

If the implementation is correct, the following test will pass.

```sh
running 8 tests
test binary::module::tests::decode_simplest_module ... ok
test binary::module::tests::decode_simplest_func ... ok
test binary::module::tests::decode_func_param ... ok
test binary::module::tests::decode_func_local ... ok
test binary::module::tests::decode_func_add ... ok
test binary::module::tests::decode_func_call ... ok
test execution::runtime::tests::not_found_export_function ... ok
test execution::runtime::tests::execute_i32_add ... ok
```

### Implementation of `call` Instruction Processing

Now that the decoding process is implemented, proceed to implement the instruction processing.

src/execution/runtime.rs
```diff
diff --git a/src/execution/runtime.rs b/src/execution/runtime.rs
index bc2a20b..f65d61e 100644
--- a/src/execution/runtime.rs
+++ b/src/execution/runtime.rs
@@ -57,7 +57,7 @@ impl Runtime {
         }
     }
 
-    fn invoke_internal(&mut self, func: InternalFuncInst) -> Result<Option<Value>> {
+    fn push_frame(&mut self, func: &InternalFuncInst) {
         let bottom = self.stack.len() - func.func_type.params.len();
         let mut locals = self.stack.split_off(bottom);
 
@@ -79,6 +79,12 @@ impl Runtime {
         };
 
         self.call_stack.push(frame);
+    }
+
+    fn invoke_internal(&mut self, func: InternalFuncInst) -> Result<Option<Value>> {
+        let arity = func.func_type.results.len();
+
+        self.push_frame(&func);
 
         if let Err(e) = self.execute() {
             self.cleanup();
@@ -127,7 +133,14 @@ impl Runtime {
                     let result = left + right;
                     self.stack.push(result);
                 }
-                _ => todo!(),
+                Instruction::Call(idx) => {
+                    let Some(func) = self.store.funcs.get(*idx as usize) else {
+                        bail!("not found func");
+                    };
+                    match func {
+                        FuncInst::Internal(func) => self.push_frame(&func.clone()),
+                    }
+                }
             }
         }
         Ok(())
```

The task for a function call instruction is simple: retrieve the `InternalFuncInst` specified by the index from the `Store`, create a frame, and `push` it onto the call stack.

Since the process of receiving `InternalFuncInst` and pushing a frame to the call stack is common, it is extracted as `Runtime::push_frame(...)` to be used in `Runtime::invoke_internal(...)` for the call instruction processing.

Finally, add tests to ensure the implementation is correct.

src/execution/runtime.rs
```diff
diff --git a/src/execution/runtime.rs b/src/execution/runtime.rs
index f65d61e..509ec05 100644
--- a/src/execution/runtime.rs
+++ b/src/execution/runtime.rs
@@ -193,4 +193,18 @@ mod tests {
         assert!(result.is_err());
         Ok(())
     }
+
+    #[test]
+    fn func_call() -> Result<()> {
+        let wasm = wat::parse_file("src/fixtures/func_call.wat")?;
+        let mut runtime = Runtime::instantiate(wasm)?;
+        let tests = vec![(2, 4), (10, 20), (1, 2)];
+
+        for (arg, want) in tests {
+            let args = vec![Value::I32(arg)];
+            let result = runtime.call("call_doubler", args)?;
+            assert_eq!(result, Some(Value::I32(want)));
+        }
+        Ok(())
+    }
 }
```

```sh
running 9 tests
test binary::module::tests::decode_func_local ... ok
test binary::module::tests::decode_simplest_module ... ok
test execution::runtime::tests::not_found_export_function ... ok
test binary::module::tests::decode_func_param ... ok
test binary::module::tests::decode_func_call ... ok
test binary::module::tests::decode_func_add ... ok
test execution::runtime::tests::func_call ... ok
test execution::runtime::tests::execute_i32_add ... ok
test binary::module::tests::decode_simplest_func ... ok
```

## Summary

In this chapter, the implementation of function calls was completed. In the next chapter, we will implement the execution of imported external functions.
