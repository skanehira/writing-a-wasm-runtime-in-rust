---
Runtimeの実装 ~ 関数の呼び出しまで ~
---

本章では前章までの実装を拡張して、以下の機能を実装していく。

- エクスポートした関数のみを実行する
- 関数を呼び出す

## エクスポートした関数のみを実行する

前章ではインデックスで実行する関数を指定していた。

```rust
let args = vec![Value::I32(left), Value::I32(right)];
let result = runtime.call(0, args)?;
assert_eq!(result, Some(Value::I32(want)));
```

これではバイナリ構造を解析しないと実行したい関数を指定できないので非常に使いづらい。
また、`Wasm spec`ではエクスポートした関数のみを実行できるという仕様があるが、今の実装は仕様を満たしていない。
そのため、本節では関数名を指定して関数を実行できるようにする。

### `Export Section`のデコード実装
`func_add.wat`を次のように修正して、`add`という関数名で関数をエクスポートする。

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

`Export Section`のデコードを実装していないため現時点ではテストが失敗するので、それを実装する。
バイナリ構造に関してはすでに[Wasmバイナリの構造](https://zenn.dev/skanehira/books/writing-wasm-runtime-in-rust/viewer/04_wasm_binary_structure)の章で解説したのでそちらを参照してほしい。

まずエクスポートを表現した型を定義する。

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

`Export::name`はエクスポート名で今回は`add`になる。
`ExportDesc`は関数やメモリなどの実体への参照となっていて、関数の場合は`Store::funcs`のインデックス値になる。
たとえば`ExportDesc::Func(0)`の場合、`add`という名前の関数の実態は`Store::funcs[0]`になる。
本書ではメモリのエクスポートなどは実装しないので、`ExportDesc::Func`のみを実装する。

続けて`Export Section`のデコードを実装する。

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

`decode_export_section(...)`では次のことをやっている。

1. エクスポートの要素数を取得  
   今回は関数1つしかエクスポートしていないので1になる
2. エクスポート名のバイト列の長さを取得
3. 2で取得した長さの分だけバイト列を取得
4. 3で取得したバイト列を文字列に変換する
5. エクスポートの種類（関数、メモリなど）を取得
6. エクスポートした関数など実体への参照を取得
7. `0x00`の場合、エクスポートの種類は関数なので`Export`を生成
8. 配列に7で生成した`Export`を追加
9. 2から8を要素数の分だけ繰り返す

これで`Export Section`のデコードを実装できたので、続けて、`Module`に`export_section`を追加する。

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

テストも修正する。

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

これでテストも通るようになる。

```sh
running 6 tests
test binary::module::tests::decode_simplest_module ... ok
test binary::module::tests::decode_simplest_func ... ok
test binary::module::tests::decode_func_local ... ok
test binary::module::tests::decode_func_param ... ok
test execution::runtime::tests::execute_i32_add ... ok
test binary::module::tests::decode_func_add ... ok
```

### 関数実行の実装

セクションのデコードができたので次は関数名を指定して実行できるように実装する。

まず、`ExportInst`（エクスポートの情報）を持つ`ModuleInst`を定義する。
`ModuleInst::exports`を`HashMap`にすることで、関数名から関数のエクスポート情報を簡単に逆引きできるようにする。

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

次に`Module::export_section`から`ModuleInst`を生成する。

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

続けて`Runtime::call(...)`で関数名を受け取るようにして、`ModuleInst`から関数へのインデックスを取得するように修正する。
合わせてテストも関数名を渡すようにする。

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

これで、次のとおりテストが通る。

```sh
running 6 tests
test binary::module::tests::decode_simplest_func ... ok
test binary::module::tests::decode_func_param ... ok
test binary::module::tests::decode_func_local ... ok
test binary::module::tests::decode_simplest_module ... ok
test binary::module::tests::decode_func_add ... ok
test execution::runtime::tests::execute_i32_add ... ok
```

存在しない関数を指定した場合のテストも追加して確認しておく。

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

## 関数を呼び出す実装

`Wasm`は関数からほかの関数を呼び出すことができる。
もちろん関数自身を再帰的に呼び出すことも可能だ。

本節では次の関数呼び出しができるように実装をしていく。
`call_doubler`関数は引数を一つ受け取り、それを`double`関数に渡して2倍にして返すということをしている。

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

### `call`命令のデコード実装
まず関数呼び出し命令をデコードできるように実装する。

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

関数呼び出し命令はオペランドを持っていて、関数への参照（インデックス）を持っているので、それをデコードする。

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

続けて、カスタムセクションのデコードをスキップして、テストを追加する。

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

カスタムセクションはメタデータ領域のことで、自由に好きなデータを配置できる。
本書では特に使わないが、今回のWATを使うと`wat`クレートがWATをコンパイルする際にカスタムセクションを追加するようになるため、それをスキップする必要がある。

実装が問題なければ、次のとおりテストが通る。

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

### `call`命令処理の実装
デコードの処理は実装できたので、次は命令の処理を実装していく。

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

関数呼び出し命令でやることはシンプルで、`Store`からインデックスで指定された`InternalFuncInst`を取得してフレームを作成してコールスタックに`push`するだけ。

`InternalFuncInst`を受け取り、フレームをコールスタックに`push`する処理は共通なので`Runtime::push_frame(...)`として切り出して、
`Runtime::invoke_internal(...)`と呼び出し命令の処理で使えるようにしている。

最後にテストを追加して、実装が問題ないことを確認する。

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

## まとめ
本章では関数の呼び出しができるように実装した。
次章ではインポートした外部関数を実行できるように実装していく。
