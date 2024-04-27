# Introduction

Wasm (WebAssembly) is a [virtual instruction set](https://en.wikipedia.org/wiki/Instruction_set_architecture) that can be executed in modern major browsers.

It has the following main features:

- Can be executed in major browsers
- Not dependent on OS or CPU
- Runs in a secure sandbox environment
- Provides performance close to native[^1]
- Can be compiled from multiple languages (Rust, Go, C, etc.)

With a Runtime environment, Wasm binaries can be executed not only in browsers but also on the server-side. For example, Wasm can be adopted in an application's plugin system or utilized in serverless application development.

The interest in Wasm, which is expected to continue to grow, is likely shared by many who are curious about its operation principles. In this document, after introducing Wasm and explaining its use cases, we aim to understand the operational principles by implementing a small Runtime from scratch using Rust to output `Hello World`.

Even though understanding a small Runtime may require some effort, we will explain each aspect step by step, so let's proceed together without rushing.

Things you need to understand for the small Runtime include:

- Understanding the data structure of Wasm binaries
- Understanding the instruction set of Wasm used
- Understanding the mechanism of instruction processing
- Implementing Wasm binary decoding
- Implementing instruction processing

By the way, the Runtime to be implemented adheres to the specifications of version 1 ([specification](https://www.w3.org/TR/wasm-core-1/)). The specification may be challenging to read, but for those interested, we encourage you to continue implementing based on the explanations provided in this document.

## Target Audience {/examples/}

The target audience of this document is as follows:

- Those who understand the basics of Rust and can read and write in it
- Those interested in Wasm
- Those who want to understand how Wasm is executed

## Glossary {/examples/}

The terms used in this document are as follows:

- Wasm
  Abbreviation for WebAssembly
  Refers to the entire ecosystem
- Wasm binary
  Refers to `*.wasm` files
  Contains bytecode
- Wasm Runtime
  Environment that can execute Wasm, also known as an interpreter
  This document will implement a Runtime that can read and execute `*.wasm` files
- Wasm Spec
  Refers to the specifications of Wasm
  This document adheres to the specifications of [version 1](https://www.w3.org/TR/wasm-core-1/)

## About This Document {/examples/}

The manuscript of this document is available in this [repository](https://github.com/skanehira/writing-a-wasm-runtime-in-rust), so if you find any confusing parts or strange explanations, please submit a pull request.

## About the Author {/examples/}

[skanehira](https://github.com/skanehira)

[^1]: Strictly depends on the implementation of the Runtime
