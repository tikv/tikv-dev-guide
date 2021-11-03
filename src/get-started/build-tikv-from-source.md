# Build TiKV from Source

TiKV is mostly written in Rust, and has components written in C++ (RocksDB, gRPC). We are currently using the Rust [nightly toolchain](https://rust-lang.github.io/rustup/concepts/toolchains.html). To provide consistency, we use linters and automated formatting tools.

### Prerequisites

To build TiKV you'll need to at least have the following installed:

* [`git`](https://git-scm.com/) - Version control
* [`rustup`] - Rust installer and toolchain manager
* [`make`](https://www.gnu.org/software/make/) - Build tool (run common workflows)
* [`cmake`](https://cmake.org/) - Build tool (required for [gRPC])
* [`awk`](https://www.gnu.org/software/gawk/manual/gawk.html) - Pattern scanning/processing language
* C++ compiler - [`gcc`](https://gcc.gnu.org/) 4.9+ (required for [gRPC])

If you are targeting platforms other than x86_64 linux, you'll also need:

* [`llvm` and `clang`](http://releases.llvm.org/download.html) - Used to generate bindings for different platforms and build native libraries (required for grpcio, rocksdb).

*(Latest version of the above tools should work in most cases. When encountering any trouble of building TiKV, try upgrading to the latest. If it is not helped, do not hesitate to [ask](#ask-for-help).)*

### Getting the repository

```
git clone https://github.com/tikv/tikv.git
cd tikv
# Future instructions assume you are in this directory
```

### Building and testing

TiKV includes a [`Makefile`](https://github.com/tikv/tikv/blob/master/Makefile) that has common workflows and sets up a standard build environment. You can also use [`cargo`], as you do in many other Rust projects. To run `cargo` commands in the same environment as the `Makefile` to avoid re-compilations due to environment changes, you can prefix the command with [`scripts/env`](https://github.com/tikv/tikv/blob/master/scripts/env), for example: `./scripts/env cargo build`.

Furthermore, when building by `make`, `cargo` is configured to use [pipelined compilation](https://internals.rust-lang.org/t/evaluating-pipelined-rustc-compilation/10199) to increase the parallelism of the build. To turn on pipelining while using `cargo` directly, set environment variable `export CARGO_BUILD_PIPELINING=true`.

To build TiKV:

```bash
make build
```

During interactive development, you may prefer using `cargo check`, which will parse, do borrow check, and lint your code, but not actually compile it:

```bash
./scripts/env cargo check --all
```

It is particularly handy alongside `cargo-watch`, which runs a command each time you change a file.

```bash
cargo install cargo-watch
cargo watch -s "./scripts/env cargo check --all"
```

When you're ready to test out your changes, use the `dev` task. It will format your codebase, build with `clippy` enabled, and run tests. In most case, this should be done without any failure before you create a Pull Request. Unfortunately, some tests will fail intermittently or can not pass on your platform. If you're unsure, just [ask](#ask-for-help)!

```bash
make dev
```

You can run the test suite alone, or just run a specific test:

```bash
# Run the full suite
make test
# Run a specific test
./scripts/test $TESTNAME -- --nocapture
```

TiKV follows the Rust community coding style. We use [rustfmt](https://github.com/rust-lang/rustfmt) and [clippy](https://github.com/rust-lang/rust-clippy) to automatically format and lint our codes. Using these tools is included in our CI. They are also part of `make dev`, and you can run them alone:

```bash
# Run Rustfmt
make format
# Run Clippy (note that some lints are ignored, so `cargo clippy` will give many false positives)
make clippy
```

See the [Rust Style Guide](https://github.com/rust-lang/rfcs/blob/master/style-guide/README.md) and the [Rust API Guidelines](https://rust-lang-nursery.github.io/api-guidelines/) for details on the conventions.

Please follow this style to make TiKV easy to review, maintain, and develop.

### Build for Debugging

To reduce compilation time, TiKV builds do not include full debugging information by default &mdash; `release` and `bench` builds include no debuginfo; `dev` and `test` builds include line numbers only. The easiest way to enable debuginfo is to precede build commands with `RUSTFLAGS=-Cdebuginfo=1` (for line numbers), or `RUSTFLAGS=-Cdebuginfo=2` (for full debuginfo). For example,

```bash
RUSTFLAGS=-Cdebuginfo=2 make dev
RUSTFLAGS=-Cdebuginfo=2 ./scripts/env cargo build
```

### Ask for help
If you encounter any problem during your journey, do not hesitate to reach out on the [TiDB Internals forum](https://internals.tidb.io/).


[`rustup`]: https://rustup.rs/
[`cargo`]: https://doc.rust-lang.org/cargo/
[gRPC]: https://github.com/grpc/grpc
[rustfmt]: https://github.com/rust-lang/rustfmt
[clippy]: https://github.com/rust-lang/rust-clippy
