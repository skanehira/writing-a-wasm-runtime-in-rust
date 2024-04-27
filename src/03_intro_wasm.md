---
Introduction to Wasm
---

This chapter will use a language that can be compiled into Wasm binary called WAT (WebAssembly Text Format) to experience running Wasm in practice.

For a detailed explanation of WAT, refer to the very clear explanation in MDN's [Understanding the text format of WebAssembly](https://developer.mozilla.org/en-US/docs/WebAssembly/Understanding_the_text_format). Once you have a good understanding up to "First Function Body," you should generally have no trouble following the explanations in the rest of this chapter.

## Environment
This document will explain using the following environment:

- OS: macOS Ventura
- CPU: Apple M1 Pro (ARM64)

## Prerequisites

### Installing wabt
First, install a toolset called [wabt](https://github.com/WebAssembly/wabt).
Below are the installation steps using Homebrew on macOS, but for installation methods on platforms other than macOS, please refer to the repository.

```sh
$ brew install wabt
```

In this chapter, we will use `wat2wasm` to convert WAT to Wasm binary.
The version at the time of writing is as follows.

```sh
$ wat2wasm --version
1.0.33
```

### Installing Wasmtime
To execute compiled Wasm binaries, install Wasmtime.
Below are the installation steps for macOS and Linux, but for installation on Windows, please refer to the [official documentation](https://docs.wasmtime.dev/cli-install.html#installing-wasmtime).

```sh
$ curl https://wasmtime.dev/install.sh -sSf | bash
```

The version at the time of writing is as follows.

```sh
$ wasmtime --version
wasmtime-cli 12.0.1
```

## Trying to Execute a Wasm Binary
First, create an `add.wat` file and paste the following code.
This code defines a function that takes two arguments and returns the result of their addition.

```wabt
(module
  (func (export "add") (param $a i32) (param $b i32) (result i32)
    (local.get $a)
    (local.get $b)
    i32.add
  )
)
```

Next, use `wat2wasm` to output the Wasm binary and execute it using `wasmtime`.
`wat2wasm` is a CLI that compiles `WAT` to Wasm binary.

```sh
# Compile
$ wat2wasm add.wat      
$ ls
 add.wasm
 add.wat
# Execute function
$ wasmtime add.wasm --invoke add 1 2
warning: using `--invoke` with a function that takes arguments is experimental and may break in the future
warning: using `--invoke` with a function that returns values is experimental and may break in the future
3
```

## Supplement on Stack Machine
Although MDN explains the stack machine, I felt it was slightly lacking, so here is a supplement.
Looking at the instruction list of the code we used earlier, it appears as follows:

```wat
(local.get $a)
(local.get $b)
i32.add
```

Here, `local.get` pushes the value of the argument onto the stack, and `i32.add` pops two values from the stack, adds them, and pushes the result back onto the stack.
When the function returns to the caller, if there is a return value, it is popped from the stack.

In pseudo-Rust code, this would look something like:

```rust
// Stack to store values to process
let mut stack: Vec<i32> = vec![];
// Area to hold function local variables
let mut locals: Vec<i32> = vec![];

// A loop that processes instructions
loop {
    let instruction = fetch_inst();

    match instruction {
        inst::LocalGet => {
          let value = locals.pop();
          stack.push(value);
        }
        inst::I32Add => {
          let right = stack.pop();
          let left = stack.pop();
          stack.push(left + right);
        }
        ...
    }
}

return stack.pop();
```

In this way, the Wasm Runtime performs very simple calculations using a stack machine.

<div class="warning">

The actual implementation is more complex, but fundamentally, it repeats the process as described above.

</div>

## Summary
In this chapter, we briefly ran Wasm and touched on the implementation using pseudo code.
While most explanations about WAT are deferred to MDN, which is much clearer than what the author could write, if you are unsure, please revisit it repeatedly.

The next chapter will explain the structure of Wasm binaries as preparation before implementing the Wasm Runtime.

