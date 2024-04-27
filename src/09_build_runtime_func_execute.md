---
Runtimeの実装 ~ 関数の実行まで ~
---

本章では次のWATを実行できるようRuntimeを実装していく。
実装できた段階では簡単な足し算ができる`Wasm Runtime`が出来上がる。

```wat
(module
  (func (param i32 i32) (result i32)
    (local.get 0)
    (local.get 1)
    i32.add
  )
)
```

処理の流れは大きく分けると次のようになる。

1. `binary::Module`を使って`execution::Store`を生成する
2. `execution::Store`を使って`execution::Runtime`を生成する
3. `execution::Runtime::call(...)`で関数を実行する

## 値の実装

今回実装するWasm Runtimeでは次の2種類の値を扱うのでそれらを実装していく。

- i32
- i64

まずは`src`配下に次のファイルを作成する。

- `src/execution/value.rs`
- `src/execution.rs`

それぞれ次のように実装する。

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

`i32`と`i64`は異なる型なので、スタックでまとめて扱うことができない。
そのため`Value`というEnumを用意してスタックで扱えるようしている。

また`Value`同士を足し算できるように`std::ops::Add`を実装している。

## `Store`の実装

[`Store`](https://www.w3.org/TR/wasm-core-1/#store%E2%91%A0)は`Wasm Runtime`が実行時に持つ状態を保持するための構造体である。
仕様書ではメモリやインポート、関数などの情報を持つと定義されていて、これらの情報を使って命令の処理をしていく。
`Wasm`バイナリがクラスだとしたら、`Store`はそのクラスのインスタンスと考えることができる。

現時点では関数の情報を持っていれば良いので、次のファイルを作成して`Store`を実装する。

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

`FuncInst`は`Wasm Runtime`が実際に処理する関数の実体となっている。
関数はインポートした関数とWasmバイナリが持つ関数がある。
今回はインポートした関数を使わないので、まずWasmバイナリが持つ関数を表現した`InternalFuncInst`を実装する。

`InternalFuncInst`のフィールドはそれぞれ次のようになっている。

- `func_type`: 関数のシグネチャ（引数と戻り値）情報
- `code`: 関数のローカル変数の定義と命令列を持つ`Func`型

次に、`binary::Module`を受け取り`Store`を生成する関数を実装する。

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

実装で使用する各種セクションはそれぞれ次のデータを持っている。

| セクション         | 概要                                 |
|--------------------|--------------------------------------|
| `Type Section`     | 関数シグネチャの情報                 |
| `Code Section`     | 関数ごとの命令などの情報             |
| `Function Section` | 関数シグネチャへの参照情報           |

現時点の`Store::new()`でやっていることを簡潔に説明すると、各セクションに散らばっている関数を実行する際に必要な情報を集めている。
少し分かりづらいと思うので、次の`func_add.wat`の`binary::Module`をみていこう。

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

`code_section[0]`の関数がどんなシグネチャを持っているかを知るには`function_section[0]`の値を取得する必要がある。
その値は`type_section`のインデックスになっているので、そのまま`type_section[0]`が`code_section[0]`のシグネチャ情報になる。
例えば`function_section[0]`の値が1だった場合、`type_section[1]`が`code_section[0]`のシグネチャ情報になる。

## `Runtime`の実装

`Runtime`は`Wasm Runtime`そのもので、次の情報を持つ構造体となっている。

- `Store`
- スタック
- コールスタック

前章ではスタックとコールスタックについて解説したので、それをどのように実装して使うのかについて解説していく。

まず`src/execution/runtime.rs`を追加して`Runtime`と`Frame`などを定義する。

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

`Runtime::instantiate(...)`はWasmバイナリを受け取り`Runtime`を生成する関数である。

### 命令処理の実装

次に、命令を実行する`Runtime::execute(...)`を実装する。

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

`execute(...)`がプログラムカウンタが指す命令がなくなるまでループで次のことを行っている。

1. コールスタックの一番上にあるフレームを取得する  
2. pc（プログラムカウンタ）をインクリメントして、次の命令をフレームから取得する  
3. 命令ごとの処理  
   `i32.add`という命令の場合は、スタックから値を2つ`pop`して、それらを足した結果をスタックに`push`する

やっていることは前の章で示した擬似コードとほぼ同じである。

続けて`local.get`と`end`命令も実装する。

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

`local.get`命令はローカル変数の値を取得してスタックに`push`する命令である。
`local.get`はオペランドを持っていて、これはどのローカル変数の値を取得するかを示す値で`frame.locals`のインデックス値である。

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

`end`命令は関数の実行終了を意味をしていて、この命令の場合は次の処理を行っている。

1. コールスタックからフレームを`pop`する
2. フレーム情報から`sp`（スタックポインタ）と`arity`（戻り値の数）を取得
3. スタックを巻き戻す
   1. 戻り値がある場合、スタックから値を1つ`pop`してから`sp`までスタックを巻き戻す
   2. `pop`した値をスタックに`push`する
   3. 戻り値がない場合は単純に`sp`までスタックを巻き戻す

`Runtime::execute(...)`のベース実装はこれで終わりなので、次に命令実行の前と後処理をする`Runtime::invoke_internal(...)`を実装する。

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

`Runtime::invoke_internal(...)`では次のことをやっている。

1. 関数の引数の数を取得
2. 引数の数だけスタックから値を`pop`する
3. ローカル変数を初期化
4. 関数の戻り値の数を取得
5. フレームを作成して、`Runtime::call_stack`に`push`する
6. `Runtime::execute()`を呼び出し関数を実行する
7. 戻り値がある場合はスタックから`pop`して返す、ない場合は`None`を返す

続けて`Runtime::invoke_internal(...)`を呼び出す`Runtime::call(...)`を実装して、呼び出す関数の指定と関数の引数を渡せるようにする。

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

`Runtime::call(...)`では次の処理を行っている。

1. 指定されたインデックスを使って`Store`が持っている`InternalFuncInst`（関数の実体）を取得
2. 引数をスタックに`push`
3. 1で取得した`InternalFuncInst`を`Runtime::invoke_internal(...)`に渡して実行して、結果を返す

これで足し算の関数を実行できる`Wasm Runtime`ができたので、
最後にテストを書いて正しく動くことを確認する。

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

問題なければ次のとおり、テストが通る。

```sh
running 6 tests
test binary::module::tests::decode_simplest_module ... ok
test binary::module::tests::decode_func_param ... ok
test binary::module::tests::decode_func_local ... ok
test binary::module::tests::decode_simplest_func ... ok
test binary::module::tests::decode_func_add ... ok
test execution::runtime::tests::execute_i32_add ... ok
```

## まとめ
本章では足し算ができる`Runtime`を実装して動いたところまで確認できた。
これで雛形ができたので次章以降はさらに拡張して、関数の呼び出しを実装していく。
