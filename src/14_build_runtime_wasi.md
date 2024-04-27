---
Implementation of Runtime ~ Up to Outputting "Hello, World!" ~
---

In this chapter, we will implement the `fd_write` function of `WASI` to be able to output `Hello, World!`.
Eventually, we will be able to execute the following WAT.

```WAT:src/fixtures/hello_world.wat
(module
  (import "wasi_snapshot_preview1" "fd_write"
    (func $fd_write (param i32 i32 i32 i32) (result i32))
  )
  (memory 1)
  (data (i32.const 0) "Hello, World!\n")

  (func $hello_world (result i32)
    (local $iovs i32)

    (i32.store (i32.const 16) (i32.const 0))
    (i32.store (i32.const 20) (i32.const 14))

    (local.set $iovs (i32.const 16))

    (call $fd_write
      (i32.const 1)
      (local.get $iovs)
      (i32.const 1)
      (i32.const 24)
    )
  )
  (export "_start" (func $hello_world))
)
```

## Implementation of `WASI`'s `fd_write()`

`WASI` has not yet reached version 1 and has versions `0.1` and `0.2`.
`0.1` is commonly referred to as `wasi_snapshot_preview1` and mainly defines system call functions.
For more details, refer to [here](https://wasi.dev).

The `fd_write()` defined in `wasi_snapshot_preview1` reads data from the `Wasm Runtime` memory and writes data to standard output or standard error output.
Therefore, by implementing this function, we can complete the `Wasm Runtime` capable of outputting `Hello, World!`.

Let's take a look at the import of the WAT at the beginning again.

```wat
(import "wasi_snapshot_preview1" "fd_write"
  (func $fd_write (param i32 i32 i32 i32) (result i32))
)
```

The arguments and return values are as follows:

- 1st argument: File descriptor to write to, `1` for standard output, `2` for standard error output
- 2nd argument: Starting position for reading memory
- 3rd argument: Number of times to read memory, the value of the 2nd argument is incremented by 4 bytes each time
- 4th argument: Location to store the number of bytes written to the output, memory index value

Now that we understand the meaning of the arguments, let's explain what `$hello_world` is doing.

```wat
(func $hello_world (result i32)
  (local $iovs i32)

  (i32.store (i32.const 16) (i32.const 0)) ;; 1
  (i32.store (i32.const 20) (i32.const 14)) ;; 2

  (local.set $iovs (i32.const 16)) ;; 3

  (call $fd_write ;; 4
    (i32.const 1)
    (local.get $iovs)
    (i32.const 1)
    (i32.const 24)
  )
)
```

1. Write `0` to the 16th byte of memory
   `0` is the start of the memory data to be written
2. Write `14` to the 20th byte of memory
   `14` is the number of bytes of memory data to be written
   In other words, it reads from `0` and writes out `14` bytes
3. Set the value `16` to the declared local variable
   `16` is the value of the starting position for reading memory, `fd_write()` reads memory from this position
4. Call `fd_write` and specify to write the number of bytes written to `fd 1` to the 24th byte of memory

The explanation may be a bit confusing, but essentially, it arranges the value range of memory data to be written to `fd`, and `fd_write()` reads the value range and writes out the data within that range.

Now that you understand what is being done, let's proceed with the implementation.

First, create `src/execution/wasi.rs` and prepare a structure representing `wasi_snapshot_preview1` as follows.

src/execution.rs
```diff
diff --git a/src/execution.rs b/src/execution.rs
index 5d6aec6..02686e0 100644
--- a/src/execution.rs
+++ b/src/execution.rs
@@ -2,3 +2,4 @@ pub mod import;
 pub mod runtime;
 pub mod store;
 pub mod value;
+pub mod wasi;
```

src/execution/wasi.rs
```rust
use std::{fs::File, os::fd::FromRawFd};

#[derive(Default)]
pub struct WasiSnapshotPreview1 {
    pub file_table: Vec<Box<File>>,
}

impl WasiSnapshotPreview1 {
    pub fn new() -> Self {
        unsafe {
            Self {
                file_table: vec![
                    Box::new(File::from_raw_fd(0)),
                    Box::new(File::from_raw_fd(1)),
                    Box::new(File::from_raw_fd(2)),
                ],
            }
        }
    }
}
```

`WasiSnapshotPreview1` holds a file table, by default having `stdin/stdout/stderr`.
`WASI` has a function `path_open()` to open files, and when doing so, it will add to the array of file tables, although this is beyond the scope of this document, we prepare the data structure in anticipation of that.

Next, implement `WasiSnapshotPreview1::invoke(...)` to be able to execute `WASI` functions from the `Wasm Runtime`.

src/execution/wasi.rs
```diff
diff --git a/src/execution/wasi.rs b/src/execution/wasi.rs
index a75dc9c..b0da928 100644
--- a/src/execution/wasi.rs
+++ b/src/execution/wasi.rs
@@ -1,5 +1,8 @@
+use anyhow::Result;
 use std::{fs::File, os::fd::FromRawFd};
 
+use super::{store::Store, value::Value};
+
 #[derive(Default)]
 pub struct WasiSnapshotPreview1 {
     pub file_table: Vec<Box<File>>,
@@ -17,4 +20,20 @@ impl WasiSnapshotPreview1 {
             }
         }
     }
+
+    pub fn invoke(
+        &mut self,
+        store: &mut Store,
+        func: &str,
+        args: Vec<Value>,
+    ) -> Result<Option<Value>> {
+        match func {
+            "fd_write" => self.fd_write(store, args),
+            _ => unimplemented!("{}", func),
+        }
+    }
+
+    pub fn fd_write(&mut self, store: &mut Store, args: Vec<Value>) -> Result<Option<Value>> {
+        // TODO
+        Ok(Some(0.into()))
+    }
 }
```

`WasiSnapshotPreview1::invoke(...)` allows calling `WASI` functions based on the specified function name, and when adding more `WASI` functions in the future, you will need to add branches to the `match` statement.

Continuing, implement the process of reading data from memory and writing to `fd`.

src/execution/wasi.rs
```diff
diff --git a/src/execution/wasi.rs b/src/execution/wasi.rs
index b0da928..6283250 100644
--- a/src/execution/wasi.rs
+++ b/src/execution/wasi.rs
@@ -1,5 +1,5 @@
 use anyhow::Result;
-use std::{fs::File, os::fd::FromRawFd};
+use std::{fs::File, io::prelude::*, os::fd::FromRawFd};
 
 use super::{store::Store, value::Value};
 
@@ -34,6 +34,49 @@ impl WasiSnapshotPreview1 {
     }
 
     pub fn fd_write(&mut self, store: &mut Store, args: Vec<Value>) -> Result<Option<Value>> {
+        let args: Vec<i32> = args.into_iter().map(Into::into).collect();
+
+        let fd = args[0];
+        let mut iovs = args[1] as usize;
+        let iovs_len = args[2];
+        let rp = args[3] as usize;
+
+        let file = self
+            .file_table
+            .get_mut(fd as usize)
+            .ok_or(anyhow::anyhow!("not found fd"))?;
+
+        let memory = store
+            .memories
+            .get_mut(0)
+            .ok_or(anyhow::anyhow!("not found memory"))?;
+
+        let mut nwritten = 0;
+
+        for _ in 0..iovs_len { // 5
+            let start = memory_read(&memory.data, iovs)? as usize; // 1
+            iovs += 4;
+
+            let len: i32 = memory_read(&memory.data, iovs)?; // 2
+            iovs += 4;
+
+            let end = start + len as usize; // 3
+            nwritten += file.write(&memory.data[start..end])?; // 4
+        }
+
+        memory_write(&mut memory.data, rp, &nwritten.to_le_bytes())?; // 5
+
         Ok(Some(0.into()))
     }
 }
+
+fn memory_read(buf: &[u8], start: usize) -> Result<i32> {
+    let end = start + 4;
+    Ok(<i32>::from_le_bytes(buf[start..end].try_into()?))
+}
+
+fn memory_write(buf: &mut [u8], start: usize, data: &[u8]) -> Result<()> {
+    let end = start + data.len();
+    buf[start..end].copy_from_slice(data);
+    Ok(())
+}
```

The process in `WasiSnapshotPreview1::fd_write(...)` may be a bit unclear, so let's explain it while showing the byte sequence.

First, let's take a look at the memory state when `WasiSnapshotPreview1::fd_write(...)` is called.

| 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 | 11 | 12 | 13 | 14 | 15 | 16 | 17 | 18 | 19 | 20 | ... |
|---|---|---|---|---|---|---|---|---|---|----|----|----|----|----|----|----|----|----|----|----|-----|
| H | e | l | l | o | , |   | W | o | r | l  | d  | !  | \n | 0  | 0  | 0  | 0  | 0  | 0  | 14 | ... |

First, we retrieve the starting position of the data to be written at the position specified by `iovs`, which is `16`.
Since `iovs` is `16`, the value is `0`, which was placed using `(i32.store (i32.const 16) (i32.const 0))`.

Since memory is aligned by 4 bytes, we add +4 to `iovs` to get the length of the data to be written at step 2.
The `14` at position `20` is the value placed using `(i32.store (i32.const 20) (i32.const 14))`.

Now that we know the length of the data to be written, we calculate the range of memory data to be extracted (0-14 bytes) at step 3 and write that byte sequence to `fd` at step 4.
This process is repeated for the number of times specified by `iovs_len`, and at step 5, the total number of bytes written is placed at the memory address specified by `rp`.

After step 5, the memory state will be as follows:

| ... | 14 | 15 | 16 | 17 | 18 | 19 | 20 | ... | 24 | ... |
|-----|----|----|----|----|----|----|----|-----|----|-----|
| ... | 0  | 0  | 0  | 0  | 0  | 0  | 14 | ... | 13 | ... |

With the implementation of `WasiSnapshotPreview1::fd_write(...)`, we can now proceed to call it.

src/execution/runtime.rs
```diff
diff --git a/src/execution/runtime.rs b/src/execution/runtime.rs
index 4fb8807..6fba7e7 100644
--- a/src/execution/runtime.rs
+++ b/src/execution/runtime.rs
@@ -4,6 +4,7 @@ use super::{
     import::Import,
     store::{ExternalFuncInst, FuncInst, InternalFuncInst, Store},
     value::Value,
+    wasi::WasiSnapshotPreview1,
 };
 use crate::binary::{
     instruction::Instruction,
@@ -27,6 +28,7 @@ pub struct Runtime {
     pub stack: Vec<Value>,
     pub call_stack: Vec<Frame>,
     pub import: Import,
+    pub wasi: Option<WasiSnapshotPreview1>,
 }
 
 impl Runtime {
@@ -120,6 +122,13 @@ impl Runtime {
         let args = self
             .stack
             .split_off(self.stack.len() - func.func_type.params.len());
+
+        if func.module == "wasi_snapshot_preview1" {
+            if let Some(wasi) = &mut self.wasi {
+                return wasi.invoke(&mut self.store, &func.func, args);
+            }
+        }
+
         let module = self
             .import
             .get_mut(&func.module)
```

Next, ensure that when creating a `Runtime`, an instance of `WasiSnapshotPreview1` can be passed.

src/execution/runtime.rs
```diff
diff --git a/src/execution/runtime.rs b/src/execution/runtime.rs
index 6fba7e7..573539f 100644
--- a/src/execution/runtime.rs
+++ b/src/execution/runtime.rs
@@ -41,6 +41,19 @@ impl Runtime {
         })
     }
 
+    pub fn instantiate_with_wasi(
+        wasm: impl AsRef<[u8]>,
+        wasi: WasiSnapshotPreview1,
+    ) -> Result<Self> {
+        let module = Module::new(wasm.as_ref())?;
+        let store = Store::new(module)?;
+        Ok(Self {
+            store,
+            wasi: Some(wasi),
+            ..Default::default()
+        })
+    }
+
     pub fn add_import(
         &mut self,
         module_name: impl Into<String>,
```

Finally, add the process to read and execute the compiled `hello_world.wasm` from `hello_world.wat` using `wat2wasm` to `main.rs`.

src/main.rs
```diff
diff --git a/src/main.rs b/src/main.rs
index e7a11a9..fd8f527 100644
--- a/src/main.rs
+++ b/src/main.rs
@@ -1,3 +1,10 @@
-fn main() {
-    println!("Hello, world!");
+use anyhow::Result;
+use tinywasm::execution::{runtime::Runtime, wasi::WasiSnapshotPreview1};
+
+fn main() -> Result<()> {
+    let wasi = WasiSnapshotPreview1::new();
+    let wasm = include_bytes!("./fixtures/hello_world.wasm");
+    let mut runtime = Runtime::instantiate_with_wasi(wasm, wasi)?;
+    runtime.call("_start", vec![]).unwrap();
+    Ok(())
 }
```

If the implementation is correct, `Hello, World!` should be output as follows:

```sh
$ cargo run -q
Hello, World!
```

## Summary
With this, a small `Wasm Runtime` capable of outputting `Hello, World!` has been completed.
Although there was much to learn, the tasks themselves may not have been as difficult as initially thought.

The instructions implemented in this book are minimal, but they are sufficient to understand the mechanism of a functioning `Wasm Runtime` at the implementation level.
If you are interested in challenging yourself to implement a complete `Wasm Runtime`, I encourage you to read the specifications and give it a try.
It may be challenging, but the sense of accomplishment when it runs successfully is truly rewarding.

For reference, by implementing version 1 instructions and a certain level of `WASI`, you can achieve the following:

https://zenn.dev/skanehira/articles/2023-09-18-rust-wasm-runtime-containerd
https://zenn.dev/skanehira/articles/2023-12-02-wasm-risp

Lastly, I would like to express my gratitude for reading this book.
If you found it valuable, please consider sharing it on social media platforms.
