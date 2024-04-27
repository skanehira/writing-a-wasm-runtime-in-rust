# Implementation instruction decoding

In the previous chapter, we implemented the decoding of the smallest function that does nothing, but left the following TODOs:

- Decode arguments and return values of the `Type Section`.
- Decode local variables and instructions of the `Code Section`.

In this chapter, we will proceed with the implementation of these processes.

## Decoding Function Arguments
We will implement the decoding of arguments, so define a function with arguments as follows:

```wat
(module
  (func (param i32 i64)
  )
)
```

The `Type Section` looks like this:

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

Decode this following these steps:

1. Read the number of function signatures, `num types`
    - Repeat steps 2 to 6 this number of times
2. Discard the value of `func`
    - Represents the type of function signature, fixed as `0x60` in the `Wasm spec`
    - Not used in this document
3. Read the number of arguments, `num params`
4. Decode the type information of arguments this number of times
    - Convert to `ValueType` for `0x7F` as `i32` and `0x7E` as `i64`
5. Read the number of return values, `num results`
6. Decode the type information of return values this number of times
    - Convert to `ValueType` for `0x7F` as `i32` and `0x7E` as `i64`

The implementation of the above steps is as follows.
The numbers in the comments correspond to the steps mentioned above.

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

`many0()` is a function that parses input using the provided function until the input ends, returning the remaining input and parsing results as a `Vec`.
Here, we repeatedly apply the function `decode_value_type()` that reads `u8` and converts it to `ValueType`.
By using `many0()`, we eliminate the need for a `for` loop for decoding, simplifying the implementation.

With the implementation completed, we will proceed to implement tests, focusing only on testing arguments at this point.
To test return values, we need to implement instruction decoding separately.

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

If everything is correct, the tests should pass as follows.

```sh
running 3 tests
test binary::module::tests::decode_simplest_module ... ok
test binary::module::tests::decode_func_param ... ok
test binary::module::tests::decode_simplest_func ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

## Decoding Local Variables
Next, we will implement the decoding of local variables.
The WAT code to be used is as follows:

```wat
(module
  (func
    (local i32)
    (local i64 i64)
  )
)
```

The binary structure of the `Code Section` is as follows:

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

Decode this following these steps:

1. Read the number of functions, `num functions`
    - Repeat steps 2 to 5 this number of times
2. Read `func body size`
3. Extract the byte sequence obtained in step 2
    - Used as input for decoding local variables and instructions
4. Decode information of local variables using the byte sequence obtained in step 3
    1. Read the number of local variables, `local decl count`
    2. Repeat steps 4-3 to 4-4 this number of times
    3. Read the number of types, `local type count`
    4. Convert values to `ValueType` this number of times

Decode the remaining byte sequence into instructions.

The decoding of instructions will be implemented in the next section, but the flow is as described above.
The implementation following these steps is as follows.

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
+    // TODO: Decoding instructions
+    body.code = vec![Instruction::End];
+
+    Ok((&[], body))
+}
+
 #[cfg(test)]
 mod tests {
     use crate::binary::{
```

With the implementation completed, we will proceed to implement tests.

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

Prepare test data as well.

src/fixtures/func_local.wat
```wat
(module
  (func
    (local i32)
    (local i64 i64)
  )
)
```

If there are no issues, the test will pass.

```sh
running 4 tests
test binary::module::tests::decode_simplest_module ... ok
test binary::module::tests::decode_simplest_func ... ok
test binary::module::tests::decode_func_param ... ok
test binary::module::tests::decode_func_local ... ok

test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

## Instruction Decoding
In Wasm, instructions are basically composed of two parts: opcode and operand.
The opcode is the identification number of the instruction, indicating what the instruction will do.
The operand indicates the target of the instruction.

For example, the instruction `i32.const` performs the operation of pushing the operand value onto the stack.
In the case of `(i32.const 1)`, it means pushing `1` onto the stack, and `(i32.const 2)` means pushing `2` onto the stack.

There are also instructions that do not have operands.
For example, `i32.add` pops two values from the stack, adds them, and pushes the result back onto the stack, but this instruction does not have an operand.

For each opcode, you can see what operation it performs in the [Index of Instructions](https://www.w3.org/TR/wasm-core-1/#a7-index-of-instructions) in the Wasm Spec.
Although there are quite a number of instructions in the Index, only a few instructions will be implemented in this document.

Returning to the decoding discussion, in this section, we will implement decoding for the following WAT.

```wat
(module
  (func (param i32 i32) (result i32)
    (local.get 0)
    (local.get 1)
    i32.add
  )
)
```

This is a function that takes two arguments, adds them, and returns the result.

`local.get` is an instruction that retrieves an argument and pushes it onto the stack, with the operand being the index of the argument.
For example, to retrieve the first argument, you would use `0`, and for the second argument, you would use `1`.

`i32.add`, as explained earlier, pops two values from the stack and adds them.
As a result, the sum of the two arguments is pushed onto the stack, and when the function returns to its caller, the return value can be obtained by popping the value from the stack.

The binary structure of the `Code Section` is as follows.

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

Since the processing flow was explained in the previous section, only the steps for decoding instructions will be shown here.

5. Decode the remaining byte sequence into instructions
    1. Read 1 byte and convert it to an opcode
    2. Depending on the type of opcode, read the operand
        1. For `local.get`, read an additional 4 bytes, convert to `u32`, and combine with the opcode to convert to an instruction
        2. For `i32.add` and `end`, convert directly to an instruction

Let's implement these steps.

First, create a file to define opcodes in `src/binary/opcode.rs`.

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

Next, add the definitions for instructions.

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
 
-    // TODO: Decoding instructions
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

Then, proceed with implementing the tests.

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

Prepare test data as well.

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

If there are no issues, the test will pass.

```sh
running 5 tests
test binary::module::tests::decode_simplest_module ... ok
test binary::module::tests::decode_simplest_func ... ok
test binary::module::tests::decode_func_param ... ok
test binary::module::tests::decode_func_add ... ok
test binary::module::tests::decode_func_local ... ok

test result: ok. 5 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

## Summary
In this chapter, we explained the implementation of function arguments, return values, local variables, and instruction decoding.
Now that we can decode a basic functioning function, the next step is to explain the mechanism of function execution.
