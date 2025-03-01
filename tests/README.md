# Testing wit-bindgen

There are a few pre-requisites to testing the project. You only need the language compilers that you wish to run test against.

- WASI SDK
  - Download from wasi-sdk releases page. If you're using Windows, you need the one with mingw in its name.
  - `curl -LO https://github.com/WebAssembly/wasi-sdk/releases/download/wasi-sdk-20/wasi-sdk-20.0-linux.tar.gz`
  - Create an environment variable called `WASI_SDK_PATH`` giving the path where you extracted the WASI SDK download, i.e., the directory containing `bin`/`lib`/`share`` folders.
- Compilers for the target language:
  - Go + TinyGo - https://tinygo.org/ (v0.27.0+)
  - Rust - wasi target: `rustup target add wasm32-wasi`
  - Java - TeaVM-WASI `ci/download-teamvm.sh`
  - C - [Clang](https://clang.llvm.org/)
  - C# - [Dotnet 8](https://dotnet.microsoft.com/en-us/download/dotnet/8.0)

There are two suites of tests: [codegen](#testing-wit-bindgen---codegen) and [runtime](#testing-wit-bindgen---runtime).  To run all possible tests, across all supported languages, ensure the dependency above are installed then run:

```
cargo test --workspace
```

To run just `codegen` tests for a single language (replace rust with language of choice: `go`, `c`, `csharp`, etc.):

```
cargo test -p wit-bindgen-rust
```

To run just `runtime` tests for a single language (replace rust with language of choice: `go`, `c`, `csharp`, etc.):

```bash
cargo test -p wit-bindgen-cli --no-default-features -F rust
```

Read on to learn more about the testing layout. It's all a bit convoluted so feel free to ask questions on [Zulip](../README.md#about) or open an issue if you're lost.

## Testing wit-bindgen - `codegen`

Any tests placed into the `tests/codegen` directory should be raw `*.wit`
files. These files will be executed in all code generators by default most
likely, and the purpose of these files is to execute language-specific
validation for each bindings generator. Basically if there's a bug where
something generates invalid code then this is probably where the test should go.
Note that this directory can have whatever it wants since nothing implements the
interfaces or tries to call them.

The tests are generated by a macro `codegen_tests` in [crates/test-helpers](../crates/test-helpers/).

## Testing wit-bindgen - `runtime`

Otherwise tests are organized in `tests/runtime/*`. Inside this directory is a
directory-per-test. These tests are somewhat heavyweight so you may want to
extend existing tests, but it's also fine to add new tests at any time.

The purpose of this directory is to contain code that's actually compiled to
wasm and executed on hosts. The code compiled-to-wasm can be one of:

* `wasm.rs` - compiled with Rust to WebAssembly
* `wasm.c` - compiled with Clang
* `wasm.java` - compiled with TeaVM-WASI

Existence of these files indicates that the language should be supported for the
test, and if a file is missing then it's skipped when running other tests. Each
`wasm.*` file is run inside each of the host file under `tests/runtime` directory.

For example, `tests/runtime/variants.rs` is the host file for `tests/runtime/variants/`

Each of these hosts can also be omitted if the host doesn't implement the test
or something like that. Otherwise for each host that exists when the host's
crate generator crate is tested it will run all these tests.

## Testing Layout

If you're adding a test, all you should generally have to do is edit files in
`tests/runtime/*`. If you're adding a new test it *should* be along the lines of
just dropping some files in there, but currently if you're adding a
Rust-compiled-to-wasm binary you'll need to edit
`crates/test-rust-wasm/Cargo.toml` and add a corresponding binary to
`crates/test-rust-wasm/src/bin/*.rs` (in the same manner as the other tests).
Other than this though all other generators should automatically pick up new
tests.

The actual way tests are hooked up looks roughly like:

* All generator crates have a `codegen.rs` and a `runtime.rs` integration test
  file (typically defined in the crate's own `tests/*.rs` directory).
* All generator crates depend on `crates/test-helpers`. This crate will walk the
  appropriate directory in the top-level `tests/*` directory.
* The `test-helpers` crate will generate appropriate `#[test]` functions to
  execute tests. For example the JS generator will run `eslint` or `tsc`. Rust
  tests for `codegen` are simply that they compile.
* The `test-helpers` crate also builds wasm files at build time, both the Rust
  and C versions. These are then encoded into the generated `#[test]` functions
  to get executed when testing.
