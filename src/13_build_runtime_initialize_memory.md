---
Runtimeの実装 ~ メモリ初期化まで ~
---

本章では`Wasm Runtime`が持つメモリを初期化してデータを配置する機能を実装する。
最終的には次のWATで`Wasm Runtime`のメモリを初期化できるようになる。

```WAT:src/fixtures/memory.wat
(module
  (memory 1)
  (data (i32.const 0) "hello")
  (data (i32.const 5) "world")
)
```

## `Memory Section`のデコード実装

まず`Memory Section`をデコードできるようにしていく。
`Memory Section`は`Wasm Runtime`初期化時にどれくらいメモリを確保するかの情報を持っているので、このセクションをデコードすることでメモリを確保できるようになる。

詳細なバイナリ構造は[Wasmバイナリの構造](https://zenn.dev/skanehira/books/writing-wasm-runtime-in-rust/viewer/04_wasm_binary_structure#memory-section)の章を参照。

最初にメモリの情報を持つ`Memory`を定義する。

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

続けて、デコード処理を実装する。

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

デコード処理は次のことをやっている。

1. メモリの数を取得
    - version 1ではメモリは1つしか扱えないので、このまま読み飛ばしている
2. `flags`と`min`を読み取る
    - `flags`はページ数の上限指定があるかどうかを確認するための値
    - `min`は初期のページ数
3. `flags`が`0`の場合はページ数上限の指定がないことを意味し、`1`の場合は上限があるのでその値を読み取る

これでメモリ確保に必要な情報が揃ったので、次は`Runtime`でメモリを持てるように実装する。

src/execution/store.rs
```diff
diff --git a/src/execution/store.rs b/src/execution/store.rs
index 5666a39..efadc19 100644
--- a/src/execution/store.rs
+++ b/src/execution/store.rs
@@ -7,6 +7,8 @@ use crate::binary::{
 };
 use anyhow::{bail, Result};
 
+pub const PAGE_SIZE: u32 = 65536; // 64Ki
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

`MemoryInst`はメモリの実態で、`data`が実際に`Wasm Runtime`が操作できるメモリ領域となっている。
見ての通り、ただの可変長配列である。

`Wasm Runtime`が操作できるメモリは`MemoryInst::data`だけなので、他のプログラムとメモリ領域が被ってしまうことは発生しない。
セキュアと言われる理由の一つである。

メモリ確保の処理は次のことをやっている。

1. `memory.limits.min`はページ数なので、それとページサイズを掛けて確保する最小のメモリ量を計算
2. 1で計算したサイズ分の配列を0埋めで初期化する

ちなみに、今回実装する範囲では特に使わないが`memory.limits.max`はメモリの上限チェックに使用する。
`Wasm`では`memory.grow`命令を使ってメモリを増やせるが、増やせる上限を超えているかどうかをチェックするために使用する。

続けてテストで問題なくデコードできることを確認する。

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

## `Data Section`のデコード実装

`Data Section`は`Runtime`でメモリを確保したあとに配置するデータが定義されている領域なので、それをデコードしてメモリに配置する実装をしていく。
詳細なバイナリ構造は[Wasmバイナリの構造](https://zenn.dev/skanehira/books/writing-wasm-runtime-in-rust/viewer/04_wasm_binary_structure#data-section)を参照。

まず次の構造体を用意する。

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

それぞれのフィールドは次のようになっている。

- `memory_index`: データの配置先のメモリを指している
  - version 1ではメモリは1つしか扱えないので基本的に`0`になる
- `offset`: 配置するデータのメモリ上の先頭位置、`0`の場合は`0`バイト目から配置する
- `init`: 配置するデータのバイト列そのもの

続けてデコード処理を実装する。

src/binary/module.rs
```diff
diff --git a/src/binary/module.rs b/src/binary/module.rs
index 3e59d35..46db8dc 100644
--- a/src/binary/module.rs
+++ b/src/binary/module.rs
@@ -3,7 +3,8 @@ use super::{
     opcode::Opcode,
     section::{Function, SectionCode},
     types::{
-        Export, ExportDesc, FuncType, FunctionLocal, Import, ImportDesc, Limits, Memory, ValueType,
+        Data, Export, ExportDesc, FuncType, FunctionLocal, Import, ImportDesc, Limits, Memory,
+        ValueType,
     },
 };
 use nom::{
@@ -21,6 +22,7 @@ pub struct Module {
     pub magic: String,
     pub version: u32,
     pub memory_section: Option<Vec<Memory>>,
+    pub data_section: Option<Vec<Data>>,
     pub type_section: Option<Vec<FuncType>>,
     pub function_section: Option<Vec<u32>>,
     pub code_section: Option<Vec<Function>>,
@@ -34,6 +36,7 @@ impl Default for Module {
             magic: "\0asm".to_string(),
             version: 1,
             memory_section: None,
+            data_section: None,
             type_section: None,
             function_section: None,
             code_section: None,
@@ -75,6 +78,10 @@ impl Module {
                             let (_, memory) = decode_memory_section(section_contents)?;
                             module.memory_section = Some(vec![memory]);
                         }
+                        SectionCode::Data => {
+                            let (_, data) = deocde_data_section(section_contents)?;
+                            module.data_section = Some(data);
+                        }
                         SectionCode::Type => {
                             let (_, types) = decode_type_section(section_contents)?;
                             module.type_section = Some(types);
@@ -95,7 +102,6 @@ impl Module {
                             let (_, imports) = decode_import_section(section_contents)?;
                             module.import_section = Some(imports);
                         }
-                        _ => todo!(),
                     };
 
                     remaining = rest;
@@ -286,6 +292,31 @@ fn decode_limits(input: &[u8]) -> IResult<&[u8], Limits> {
     Ok((input, Limits { min, max }))
 }
 
+fn decode_expr(input: &[u8]) -> IResult<&[u8], u32> {
+    let (input, _) = leb128_u32(input)?;
+    let (input, offset) = leb128_u32(input)?;
+    let (input, _) = leb128_u32(input)?;
+    Ok((input, offset))
+}
+
+fn deocde_data_section(input: &[u8]) -> IResult<&[u8], Vec<Data>> {
+    let (mut input, count) = leb128_u32(input)?; // 1
+    let mut data = vec![];
+    for _ in 0..count {
+        let (rest, memory_index) = leb128_u32(input)?;
+        let (rest, offset) = decode_expr(rest)?; // 2
+        let (rest, size) = leb128_u32(rest)?; // 3
+        let (rest, init) = take(size)(rest)?; // 4
+        data.push(Data {
+            memory_index,
+            offset,
+            init: init.into(),
+        });
+        input = rest;
+    }
+    Ok((input, data))
+}
+
 fn decode_name(input: &[u8]) -> IResult<&[u8], String> {
     let (input, size) = leb128_u32(input)?;
     let (input, name) = take(size)(input)?;
```

デコード処理では次のことをやっている。

1. `segument`の個数を取得
  `(data ...)`が1 segumentになるので、複数定義されている場合は複数回処理をする必要がある
2. `offset`を計算する
  本来は`decode_expr(...)`で命令列を処理して`offset`の値を計算する必要があるが、今回は`[i32.const, 1, end]`の命令列を前提にした実装で留める
3. データのサイズを取得する
4. 3のサイズ分のバイト列を取得、これが実際のデータになる

続けて、テストを追加して実装が問題ないことを確認する。

src/binary/module.rs
```diff
diff --git a/src/binary/module.rs b/src/binary/module.rs
index c0c1aff..40f20fd 100644
--- a/src/binary/module.rs
+++ b/src/binary/module.rs
@@ -333,7 +333,7 @@ mod tests {
         module::Module,
         section::Function,
         types::{
-            Export, ExportDesc, FuncType, FunctionLocal, Import, ImportDesc, Limits, Memory,
+            Data, Export, ExportDesc, FuncType, FunctionLocal, Import, ImportDesc, Limits, Memory,
             ValueType,
         },
     };
@@ -547,4 +547,48 @@ mod tests {
         }
         Ok(())
     }
+
+    #[test]
+    fn decode_data() -> Result<()> {
+        let tests = vec![
+            (
+                "(module (memory 1) (data (i32.const 0) \"hello\"))",
+                vec![Data {
+                    memory_index: 0,
+                    offset: 0,
+                    init: "hello".as_bytes().to_vec(),
+                }],
+            ),
+            (
+                "(module (memory 1) (data (i32.const 0) \"hello\") (data (i32.const 5) \"world\"))",
+                vec![
+                    Data {
+                        memory_index: 0,
+                        offset: 0,
+                        init: b"hello".into(),
+                    },
+                    Data {
+                        memory_index: 0,
+                        offset: 5,
+                        init: b"world".into(),
+                    },
+                ],
+            ),
+        ];
+
+        for (wasm, data) in tests {
+            let module = Module::new(&wat::parse_str(wasm)?)?;
+            assert_eq!(
+                module,
+                Module {
+                    memory_section: Some(vec![Memory {
+                        limits: Limits { min: 1, max: None }
+                    }]),
+                    data_section: Some(data),
+                    ..Default::default()
+                }
+            );
+        }
+        Ok(())
+    }
 }
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

メモリデータを取得できるようになったので、最後に`Runtime`のメモリ上にデータを配置していく。

src/execution/store.rs
```diff
diff --git a/src/execution/store.rs b/src/execution/store.rs
index efadc19..cad96ca 100644
--- a/src/execution/store.rs
+++ b/src/execution/store.rs
@@ -5,7 +5,7 @@ use crate::binary::{
     module::Module,
     types::{ExportDesc, FuncType, ImportDesc, ValueType},
 };
-use anyhow::{bail, Result};
+use anyhow::{anyhow, bail, Result};
 
 pub const PAGE_SIZE: u32 = 65536; // 64Ki
 
@@ -146,6 +146,22 @@ impl Store {
             }
         }
 
+        if let Some(ref sections) = module.data_section {
+            for data in sections {
+                let memory = memories
+                    .get_mut(data.memory_index as usize)
+                    .ok_or(anyhow!("not found memory"))?;
+
+                let offset = data.offset as usize;
+                let init = &data.init;
+
+                if offset + init.len() > memory.data.len() {
+                    bail!("data is too large to fit in memory");
+                }
+                memory.data[offset..offset + init.len()].copy_from_slice(init);
+            }
+        }
+
         Ok(Self {
             funcs,
             memories,
```

やっていることはシンプルで、メモリに`Data Section`のデータを指定した場所にコピーしている。
最後にテストを追加して実装が問題ないことを確認する。

src/execution/store.rs
```diff
diff --git a/src/execution/store.rs b/src/execution/store.rs
index cad96ca..1bb1192 100644
--- a/src/execution/store.rs
+++ b/src/execution/store.rs
@@ -169,3 +169,32 @@ impl Store {
         })
     }
 }
+
+#[cfg(test)]
+mod test {
+    use super::Store;
+    use crate::binary::module::Module;
+    use anyhow::Result;
+
+    #[test]
+    fn init_memory() -> Result<()> {
+        let wasm = wat::parse_file("src/fixtures/memory.wat")?;
+        let module = Module::new(&wasm)?;
+        let store = Store::new(module)?;
+        assert_eq!(store.memories.len(), 1);
+        assert_eq!(store.memories[0].data.len(), 65536);
+        assert_eq!(&store.memories[0].data[0..5], b"hello");
+        assert_eq!(&store.memories[0].data[5..10], b"world");
+        Ok(())
+    }
+}
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

## まとめ
本章ではメモリ初期化の機能を実装した。
これでメモリ上に任意のデータを配置できるようになったので、
次章ではメモリ上に`Hello, World!`を配置して、それを出力できるようにしていく。
