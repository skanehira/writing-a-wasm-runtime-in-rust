# Implementation section decoding

In the previous chapter, we explained the basic implementation methods for decoding. In this chapter, we will implement decoding functions.

To start, we will implement the smallest function that does nothing so that it can be decoded as follows:

```wat
(module
  (func)
)
```

Compiling the above WAT to a Wasm binary results in the following binary structure:

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

## Decoding Sections
To implement the decoding of functions, it is necessary to implement the decoding process for the following sections.

| Section            | Description                          |
|--------------------|--------------------------------------|
| `Type Section`     | Information about function signatures|
| `Code Section`     | Information such as instructions for each function|
| `Function Section` | Reference information to function signatures|

The format of each section is as described in the chapter on the structure of Wasm binaries, so we will explain the implementation while referring to that.

### Decoding Section Headers
Each section has a section header that always includes `section code` and `section size`. Therefore, first create the `src/binary/section.rs` file and define an Enum for `section code`.

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

By looking at this `SectionCode`, we will proceed with the decoding process for each section.

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

Next, we will implement the function `decode_section_header()` to decode the section headers. This function simply retrieves `section code` and `section size` from the input, but there are some new functions, so we will explain them.

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
+    let (input, (code, size)) = pair(le_u8, leb128_u32)(input)?; // 1
+    Ok((
+        input,
+        (
+            SectionCode::from_u8(code).expect("unexpected section code"), // 2
+            size,
+        ),
+    ))
+}
+
 #[cfg(test)]
 mod tests {
     use crate::binary::module::Module;
```

1 `pair()` returns a new parser based on two parsers. The reason for using `pair()` is that the formats of `section code` and `section size` are fixed, so by parsing each of them together, we can process them in a single function call.

If `pair()` is not used, the implementation would look like this:

```rust
let (input, code) = le_u8(input);
let (input, size) = leb128_u32(input);
```

`section code` is fixed at 1 byte, so `le_u8()` is used.
`section size` is an `u32` encoded with LEB128[^1], so when reading the value, `leb128_u32()` is needed.

<div class="warning">

Note that in the `Wasm spec`, all numbers are required to be encoded with LEB128. The author once spent several hours trying to pass tests without knowing this.

</div>

2 `SectionCode::from_u8()` is a function implemented with the `num_derive::FromPrimitive` macro. It is used to convert the read 1-byte number to a `SectionCode`. Without using this, a manual solution like the following would be necessary:

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

Now that the implementation of decoding the section headers is done, we will proceed to implement the framework for decoding.

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

In the above implementation, the following steps are performed:

1. Repeat steps ② to ⑤ until the input `remaining` is empty.
2. Decode the section header to obtain the section code, size, and remaining input.
3. Retrieve a byte sequence of the section size from the remaining input.
    - `take()` is a function that reads a specified amount of input.
    - The read byte sequence is `section_contents`, and the remaining is `rest`.
4. Describe the decoding process for various sections.
5. Reassign `remaining` to `rest` to use it in the next loop.

The task is simple, but it may be difficult to understand at first, so let's consider processing steps 2 to 5 above based on the binary structure.

First, when extracting the binary structure parts of the `Type Section` and `Function Section`, it looks like this:

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

At step 1, `remaining` looks like this:

| remaining                                                       |
|-----------------------------------------------------------------|
| [0x01, 0x04, 0x01, 0x60, 0x0, 0x0, 0x03, 0x02, 0x01, 0x00, ...] |


After step 2, `input` and others look like this:

| section code | section size | input                                               |
|--------------|--------------|-----------------------------------------------------|
| 0x01         | 0x04         | [0x01, 0x60, 0x0, 0x0, 0x03, 0x02, 0x01, 0x00, ...] |

After step 3, `rest` and `section_contents` look like this:

| section_contents       | rest                          |
|------------------------|-------------------------------|
| [0x01, 0x60, 0x0, 0x0] | [0x03, 0x02, 0x01, 0x00, ...] |

In step 4, further decoding is done on `section_contents`.

In step 5, the value of `rest` is assigned to `remaining`, making `remaining` the input for the next section at this point.

| remaining                     |
|-------------------------------|
| [0x03, 0x02, 0x01, 0x00, ...] |

In this way, input is consumed repeatedly to decode each section. Once you understand it, it's simple, but until then, it's good to reread the explanation in this section repeatedly or try writing it yourself.

### Decoding the `Type Section`
Now that the framework is set up, let's proceed to implement the decoding process for the `Type Section`.
The `Type Section` is a section that contains function signature information, where a signature consists of combinations of arguments and return values.

The binary structure is as follows:

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

First, create the `src/binary/types.rs` file and define the structure of the signature.

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

`ValueType` represents the type of arguments.
In this case, since the defined function has no arguments, there is no type information in the binary structure, but according to the specification, `0x7F` represents `i32` and `0x7E` represents `i64`.

<div class="warning">

In the `Wasm Spec`, 4 values are defined: `i32`, `i64`, `f32`, `f64`, but for this case, only `i32` and `i64` are needed, so we will implement only those two in `ValueType`.

</div>

Next, add the `type_section` field to the `Module` struct and start implementing `todo!()`.

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

`decode_type_section()` is the function that actually decodes the `Type Section`, but it becomes a bit complex, so for now, let's make it return fixed data.
We will implement argument and return value decoding along with the next chapter.

src/binary/module.rs
```diff
@@ -77,6 +77,14 @@ fn decode_section_header(input: &[u8]) -> IResult<&[u8], (SectionCode, u32)> {
     ))
 }
 
+fn decode_type_section(_input: &[u8]) -> IResult<&[u8], Vec<FuncType>> {
+    let func_types = vec![FuncType::default()];
+
+    // TODO: Decoding arguments and return values
+
+    Ok((&[], func_types))
+}
+
 #[cfg(test)]
 mod tests {
     use crate::binary::module::Module;
```

### Decoding the `Function Section`
The `Function Section` is an area in the chapter on the structure of Wasm binaries that links to the signature information of functions (the `Type Section`).

The binary structure is as follows:

```
; section "Function" (3)
000000e: 03               ; section code
000000f: 02               ; section size
0000010: 01               ; num functions
0000011: 00               ; function 0 signature index
```

To decode this, first add the `function_section` to the `Module`.

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

The `Function Section` simply holds index information to link the `Type Section` and `Code Section`, so in Rust, it is represented as `Vec<u32>`.

Next, implement `decode_function_section()` as follows.
At the point where `decode_type_section()` is called, the `input` looks like this:
`num functions` represents the number of functions, and we read the index values this many times.

```
0000010: 01               ; num functions
0000011: 00               ; function 0 signature index
```

The implementation will be as follows:

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

Here, 1 reads `num functions`, and 2 reads the index values that many times.

### Decoding the `Code Section`
The `Code Section` is the area where function information is stored.
Function information consists of two parts:

- Number and type information of local variables
- Instruction sequence

In Rust, this is represented as follows:

```rust
// function definition
pub struct Function {
    pub locals: Vec<FunctionLocal>,
    pub code: Vec<Instruction>,
}

// Define the number of local variables and type information
pub struct FunctionLocal {
    pub type_count: u32,       // Number of local variables
    pub value_type: ValueType, // Variable type information
}

// 命令の定義
pub enum Instruction {
    LocalGet(u32),
    End,
    ...
}
```

The binary structure is as follows:
For a function that does nothing, there are no local variables, and only one `end` instruction.
The `end` instruction signifies the end of a function, and even for a function that does nothing, there must always be one `end` instruction.

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

First, create the `src/binary/instruction.rs` file and define the instructions there.

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

Next, define `FunctionLocal` to represent information about local variables.

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

Continuing, define `Function` to represent a function.

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

We would like to implement the decoding process next, but it becomes a bit complex, so for now, let's return a fixed data structure that passes the tests.
We will implement the decoding process in the next chapter.

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
+    // TODO: Decoding local variables and instructions
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

Finally, confirm that the tests pass.
Although adjusting the implementation to pass the tests may seem trivial, it is fine for now as we will implement the decoding process properly in the next chapter.

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

Running the tests confirms that they pass.

```sh
running 2 tests
test binary::module::tests::decode_simplest_module ... ok
test binary::module::tests::decode_simplest_func ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

## Summary
While there are some TODOs remaining, this chapter explained the implementation of function decoding.
You should now have a good understanding of the overall process, and in the next chapter, we will explain the implementation of decoding function arguments and return values, along with addressing the remaining TODOs.

[^1]: A variable-length code compression method for storing numbers of arbitrary size in a small number of bytes.
