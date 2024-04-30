# Implementation memory initialization

In this chapter, we will implement the functionality to initialize the memory held by the `Wasm Runtime` and place data in it.
Ultimately, we will be able to initialize the memory of the `Wasm Runtime` with the following WAT.

We will also implement the `i32.store` instruction left in the previous chapter.

```WAT:src/fixtures/memory.wat
(module
  (memory 1)
  (data (i32.const 0) "hello")
  (data (i32.const 5) "world")
)
```

## Implementation of `Memory Section` Decoding

First, we will work on decoding the `Memory Section`.
The `Memory Section` contains information on how much memory to allocate during `Wasm Runtime` initialization, so by decoding this section, we will be able to allocate memory.

For detailed binary structure, refer to the chapter on [Wasm Binary Structure](https://skanehira.github.io/writing-a-wasm-runtime-in-rust/04_wasm_binary_structure.html#memory-section).

First, define the `Memory` that holds memory information.

/src/binary/types.rs
```diff
diff --git a/src/binary/types.rs b/src/binary/types.rs
index 64912f8..3d620f3 100644
--- a/src/binary/types.rs
+++ b/src/binary/types.rs
@@ -48,3 +48,14 @@ pub struct Import {
     pub field: String,
     pub desc: ImportDesc,
 }
+
+#[derive(Debug, Clone, PartialEq, Eq)]
+pub struct Limits {
+    pub min: u32,
+    pub max: Option<u32>,
+}
+
+#[derive(Debug, Clone, PartialEq, Eq)]
+pub struct Memory {
+    pub limits: Limits,
+}
```

Next, implement the decoding process.

src/binary/module.rs
```diff
diff --git a/src/binary/module.rs b/src/binary/module.rs
index 2a8dbfa..d7b56ba 100644
--- a/src/binary/module.rs
+++ b/src/binary/module.rs
@@ -2,7 +2,9 @@ use super::{
     instruction::Instruction,
     opcode::Opcode,
     section::{Function, SectionCode},
-    types::{Export, ExportDesc, FuncType, FunctionLocal, Import, ImportDesc, ValueType},
+    types::{
+        Export, ExportDesc, FuncType, FunctionLocal, Import, ImportDesc, Limits, Memory, ValueType,
+    },
 };
 use nom::{
     bytes::complete::{tag, take},
@@ -18,6 +20,7 @@ use num_traits::FromPrimitive as _;
 pub struct Module {
     pub magic: String,
     pub version: u32,
+    pub memory_section: Option<Vec<Memory>>,
     pub type_section: Option<Vec<FuncType>>,
     pub function_section: Option<Vec<u32>>,
     pub code_section: Option<Vec<Function>>,
@@ -30,6 +33,7 @@ impl Default for Module {
         Self {
             magic: "\0asm".to_string(),
             version: 1,
+            memory_section: None,
             type_section: None,
             function_section: None,
             code_section: None,
@@ -67,6 +71,10 @@ impl Module {
                         SectionCode::Custom => {
                             // skip
                         }
+                        SectionCode::Memory => {
+                            let (_, memory) = decode_memory_section(section_contents)?;
+                            module.memory_section = Some(vec![memory]);
+                        }
                         SectionCode::Type => {
                             let (_, types) = decode_type_section(section_contents)?;
                             module.type_section = Some(types);
@@ -260,6 +268,24 @@ fn decode_import_section(input: &[u8]) -> IResult<&[u8], Vec<Import>> {
     Ok((&[], imports))
 }
 
+fn decode_memory_section(input: &[u8]) -> IResult<&[u8], Memory> {
+    let (input, _) = leb128_u32(input)?; // 1
+    let (_, limits) = decode_limits(input)?;
+    Ok((input, Memory { limits }))
+}
+
+fn decode_limits(input: &[u8]) -> IResult<&[u8], Limits> {
+    let (input, (flags, min)) = pair(leb128_u32, leb128_u32)(input)?; // 2
+    let max = if flags == 0 { // 3
+        None
+    } else {
+        let (_, max) = leb128_u32(input)?;
+        Some(max)
+    };
+
+    Ok((input, Limits { min, max }))
+}
+
 fn decode_name(input: &[u8]) -> IResult<&[u8], String> {
     let (input, size) = leb128_u32(input)?;
     let (input, name) = take(size)(input)?;
```

The decoding process does the following:

1. Retrieve the number of memories
    - In version 1, only one memory can be handled, so it is currently being skipped
2. Read `flags` and `min`
    - `flags` is a value used to check if there is an upper limit specification for the number of pages
    - `min` is the initial number of pages
3. If `flags` is `0`, it means there is no specification for the upper limit of pages, and if it is `1`, there is a limit, so read that value

With this information necessary for memory allocation gathered, the next step is to implement the ability for the `Runtime` to have memory.

src/execution/store.rs
```diff
diff --git a/src/execution/store.rs b/src/execution/store.rs
index 5666a39..efadc19 100644
--- a/src/execution/store.rs
+++ b/src/execution/store.rs
@@ -7,6 +7,8 @@ use crate::binary::{
 };
 use anyhow::{bail, Result};
 
+pub const PAGE_SIZE: u32 = 65536; // 64KiB
+
 #[derive(Clone)]
 pub struct Func {
     pub locals: Vec<ValueType>,
@@ -42,10 +44,17 @@ pub struct ModuleInst {
     pub exports: HashMap<String, ExportInst>,
 }
 
+#[derive(Default, Debug, Clone)]
+pub struct MemoryInst {
+    pub data: Vec<u8>,
+    pub max: Option<u32>,
+}
+
 #[derive(Default)]
 pub struct Store {
     pub funcs: Vec<FuncInst>,
     pub module: ModuleInst,
+    pub memories: Vec<MemoryInst>,
 }
 
 impl Store {
@@ -56,6 +65,7 @@ impl Store {
         };
 
         let mut funcs = vec![];
+        let mut memories = vec![];
 
         if let Some(ref import_section) = module.import_section {
             for import in import_section {
@@ -125,8 +135,20 @@ impl Store {
         };
         let module_inst = ModuleInst { exports };
 
+        if let Some(ref sections) = module.memory_section {
+            for memory in sections {
+                let min = memory.limits.min * PAGE_SIZE; // 1
+                let memory = MemoryInst {
+                    data: vec![0; min as usize], // 2
+                    max: memory.limits.max,
+                };
+                memories.push(memory);
+            }
+        }
+
         Ok(Self {
             funcs,
+            memories,
             module: module_inst,
         })
     }
```

`MemoryInst` represents the actual memory, where `data` is the memory area that the `Wasm Runtime` can operate on.
As you can see, it is simply a variable-length array.

The memory that the `Wasm Runtime` can operate on is only `MemoryInst::data`, so there will be no overlap with memory areas of other programs, which is one of the reasons it is considered secure.

The memory allocation process does the following:

1. Since `memory.limits.min` represents the number of pages, calculate the minimum amount of memory to allocate by multiplying it with the page size
2. Initialize an array of the calculated size with zeros

Additionally, although not used extensively in the scope of this implementation, `memory.limits.max` is used for checking the upper limit of memory.
In `Wasm`, the `memory.grow` instruction is used to increase memory, and this is used to check if the increase exceeds the allowable limit.

Next, confirm that decoding works correctly in tests.

src/binary/module.rs
```diff
diff --git a/src/binary/module.rs b/src/binary/module.rs
index 935654b..3e59d35 100644
--- a/src/binary/module.rs
+++ b/src/binary/module.rs
@@ -301,7 +301,10 @@ mod tests {
         instruction::Instruction,
         module::Module,
         section::Function,
-        types::{Export, ExportDesc, FuncType, FunctionLocal, Import, ImportDesc, ValueType},
+        types::{
+            Export, ExportDesc, FuncType, FunctionLocal, Import, ImportDesc, Limits, Memory,
+            ValueType,
+        },
     };
     use anyhow::Result;
 
@@ -488,4 +491,29 @@ mod tests {
         );
         Ok(())
     }
+
+    #[test]
+    fn decode_memory() -> Result<()> {
+        let tests = vec![
+            ("(module (memory 1))", Limits { min: 1, max: None }),
+            (
+                "(module (memory 1 2))",
+                Limits {
+                    min: 1,
+                    max: Some(2),
+                },
+            ),
+        ];
+        for (wasm, limits) in tests {
+            let module = Module::new(&wat::parse_str(wasm)?)?;
+            assert_eq!(
+                module,
+                Module {
+                    memory_section: Some(vec![Memory { limits }]),
+                    ..Default::default()
+                }
+            );
+        }
+        Ok(())
+    }
 }
```

```sh
running 13 tests
test binary::module::tests::decode_simplest_module ... ok
test binary::module::tests::decode_memory ... ok
test binary::module::tests::decode_func_param ... ok
test binary::module::tests::decode_simplest_func ... ok
test binary::module::tests::decode_func_call ... ok
test execution::runtime::tests::not_found_export_function ... ok
test binary::module::tests::decode_func_add ... ok
test execution::runtime::tests::execute_i32_add ... ok
test execution::runtime::tests::func_call ... ok
test binary::module::tests::decode_func_local ... ok
test binary::module::tests::decode_import ... ok
test execution::runtime::tests::not_found_imported_func ... ok
test execution::runtime::tests::call_imported_func ... ok
```

## Implementation of `Data Section` Decoding

The `Data Section` defines the area where data to be placed in memory after memory allocation by the `Runtime` is specified. We will proceed with implementing the decoding to place this data in memory.
For detailed binary structure, refer to the [Wasm Binary Structure](https://skanehira.github.io/writing-a-wasm-runtime-in-rust/04_wasm_binary_structure.html#data-section).

First, prepare the following structure.

src/binary/types.rs
```diff
diff --git a/src/binary/types.rs b/src/binary/types.rs
index 3d620f3..bff2cd4 100644
--- a/src/binary/types.rs
+++ b/src/binary/types.rs
@@ -59,3 +59,10 @@ pub struct Limits {
 pub struct Memory {
     pub limits: Limits,
 }
+
+#[derive(Debug, Clone, PartialEq, Eq)]
+pub struct Data {
+    pub memory_index: u32,
+    pub offset: u32,
+    pub init: Vec<u8>,
+}
```

Each field is as follows:

- `memory_index`: Points to the memory where the data will be placed
  - In version 1, since only one memory can be handled, it will generally be `0`
- `offset`: The starting position in memory where the data will be placed; if `0`, the data will be placed starting from byte `0`
- `init`: The byte sequence of the data to be placed

Next, implement the decoding process.

```diff
src/binary/module.rs
```

In the decoding process, the following steps are performed:

1. Obtain the number of `segments`
   If `(data ...)` becomes one segment, then if multiple are defined, processing needs to be done multiple times.
2. Calculate the `offset`
   Normally, the value of `offset` needs to be calculated by processing the instruction sequence in `decode_expr(...)`, but for now, the implementation is based on the assumption of the instruction sequence `[i32.const, 1, end]`.
3. Obtain the size of the data.
4. Retrieve a byte array of the size from step 3, which becomes the actual data.

Next, add tests to ensure the implementation is correct.

```diff
src/binary/module.rs
```

```sh
running 14 tests
test binary::module::tests::decode_simplest_module ... ok
test binary::module::tests::decode_memory ... ok
test binary::module::tests::decode_simplest_func ... ok
test binary::module::tests::decode_func_add ... ok
test binary::module::tests::decode_func_param ... ok
test binary::module::tests::decode_import ... ok
test binary::module::tests::decode_func_call ... ok
test binary::module::tests::decode_func_local ... ok
test binary::module::tests::decode_data ... ok
test execution::runtime::tests::call_imported_func ... ok
test execution::runtime::tests::execute_i32_add ... ok
test execution::runtime::tests::not_found_export_function ... ok
test execution::runtime::tests::func_call ... ok
test execution::runtime::tests::not_found_imported_func ... ok
```

Now that we can retrieve memory data, we will proceed to place the data on the `Runtime` memory.

```diff
src/execution/store.rs
```

The process is simple, copying the data from the `Data Section` to the specified location in memory.
Finally, add tests to ensure the implementation is correct.

```diff
src/execution/store.rs
```

```sh
running 15 tests
test binary::module::tests::decode_func_param ... ok
test binary::module::tests::decode_func_local ... ok
test binary::module::tests::decode_simplest_module ... ok
test binary::module::tests::decode_simplest_func ... ok
test binary::module::tests::decode_memory ... ok
test binary::module::tests::decode_import ... ok
test binary::module::tests::decode_func_add ... ok
test execution::runtime::tests::call_imported_func ... ok
test binary::module::tests::decode_func_call ... ok
test binary::module::tests::decode_data ... ok
test execution::runtime::tests::execute_i32_add ... ok
test execution::runtime::tests::not_found_export_function ... ok
test execution::store::test::init_memory ... ok
test execution::runtime::tests::func_call ... ok
test execution::runtime::tests::not_found_imported_func ... ok
```

## Implementation of `i32.store`

Now that we can initialize memory, we will implement the `i32.store` command.

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

The instruction processing involves the following.

1. Get the value and address to write from the stack to memory.
2. Obtain the index of the write destination by adding the address from step 1 with the offset.
3. Calculate the range for writing to memory.
4. Convert the value to a little-endian byte array and copy the data to the calculated range from steps 2 and 3.

Finally, add tests to ensure the implementation is correct.

src/fixtures/i32_store.wat
```wat
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

In `i32_store.wat`, the address is `0`, the value is `42`, and the offset is `0`, so ultimately `42` will be written to memory address `0`. Therefore, the test will confirm that memory address `0` contains `42`.

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

## Summary
In this chapter, the functionality for initializing memory has been implemented.
With this, we can now place arbitrary data on memory, so in the next chapter, we will place `Hello, World!` on memory and work towards being able to output it.
