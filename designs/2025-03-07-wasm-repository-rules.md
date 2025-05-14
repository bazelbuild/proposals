---
title: Executing WebAssembly in repository rules
authors: [jmillikin]
created: 2025-03-12
last updated: 2025-05-14
status: [Approved](https://github.com/bazelbuild/bazel/discussions/25537#discussioncomment-13145966)
reviewers:
  - meteorcloudy
  - Wyverald
  - coeuvre
  - meisterT
  - lberki
---

# Summary

Allow WebAssembly modules to be loaded and executed within repository rules,
enabling Bazel rulesets to more easily leverage existing third-party libraries
written in non-Starlark languages.

# Background

## Bazel ruleset helper tools

Rulesets that adapt third-party code to build with Bazel often need to generate
`BUILD` and/or `.bzl` files using data parsed from upstream source files. If
the parsing is too complex to re-implement in Starlark then rulesets use the
[`repository_ctx.execute()`] function to invoke a helper tool for the heavy
lifting. Examples of such tools include `gazelle` (for Go and Protobuf) and
`cargo-bazel` (for Rust). These tools are typically native binaries written in
Go, Rust, or C/C++.

[`repository_ctx.execute()`]: https://bazel.build/rules/lib/builtins/repository_ctx#execute

Example use cases include parsing upstream build configuration written in
[TOML] or [YAML] or XML, parsing Go sources using the Go standard library's
[`go/parser`] package, and extracting files from upstream source archives in
uncommon formats.

[TOML]: https://toml.io/en/
[YAML]: https://yaml.org/
[`go/parser`]: https://pkg.go.dev/go/parser@go1.24.1

This dependency on native binaries during the analysis phase is undesireable
because it imposes a maintenance burden on ruleset authors and limits the
portability of rulesets among host platforms. A repository rule that depends
on a helper binary must inspect [`repository_ctx.os`] and either (1) select
from binaries pre-compiled for various platforms by the ruleset authors, or (2)
reproduce the tool's build logic using only `repository_ctx` operations. Both
approaches tend to cause portability issues for uncommon host platforms such as
RISC-V, non-GNU Linux, or the BSDs.

[`repository_ctx.os`]: https://bazel.build/rules/lib/builtins/repository_ctx#os

Additionally, processes executed in a repository rule do not have the same level
of sandboxing as processes executed from normal rules. This can cause surprising
behavior if the helper tool's code contains hard-coded paths, for example if
it reads configuration from or writes to a cache in the user's home directory.
Non-hermetic behavior may be present without the knowledge of the ruleset
author, such as when it's an undocumented 'feature' of a third-party dependency.

## WebAssembly

[WebAssembly] is an architecture-neutral bytecode designed to be a portable
compilation target for languages that normally compile to native code. Languages
that support compilation to WebAssembly include C/C++ (when compiled with
`clang`), Rust, and a subset of Go (via [TinyGo]). Support for WebAssembly as
a compilation target continues to expand due to its widespread adoption among
web browsers and commercial hosting providers.

[WebAssembly]: https://webassembly.org/
[TinyGo]: https://tinygo.org/

For performance reasons most WebAssembly implementations (interpreters or JITs)
are written in C, Rust, or Go. [Chicory] is an implementation of the WebAssembly
core specification in Java, with its v1.0 release in December 2024. It can be
used to add WebAssembly support to JVM projects without requiring JNI or other
platform-specific code, and at the time of writing has no external dependencies.

[Chicory]: https://github.com/dylibso/chicory

# Proposal

Add functions `repository_ctx.execute_wasm()` and `repository_ctx.load_wasm()`,
which can be used within a repository rule to load a WebAssembly module from a
file and invoke its functions.

Ruleset authors can pre-compile their helper logic into a `.wasm` file, which is
either bundled directly into the ruleset archive or downloaded separately. Since
WebAssembly bytecode is platform-independent there is no need to match the
`.wasm` file to the Bazel host platform.

## `repository_ctx.execute_wasm()`

```starlark
def repository_ctx.execute_wasm(
  module: (str | path | Label | StarlarkWasmModule),
  function: str,
  *,
  input='',
  watch='auto',
): StarlarkWasmExecutionResult

StarlarkWasmExecutionResult.code: int
StarlarkWasmExecutionResult.output: str
```

The primary new functionality is `repository_ctx.execute_wasm()`, which will
instantiate a new WebAssembly VM for the given module, invoke the specified
function (passing in the provided input), and then return the result. The VM
state is not preserved.

Both the input and the output are byte arrays (Starlark type `str`) so that
Bazel itself doesn't need to have an opinion about the format used by rulesets
to communicate to/from their WebAssembly helper modules. The ABI of the invoked
function uses `(addr, len)` tuples to specify where to read/write the
input/output data.

```
func bazel_wasm_example(
  input_ptr: *uint8,
  input_len: uint32,
  output_ptr_ptr: **uint8,
  output_len_ptr: *uint32,
) -> (code: int32)
```

Memory is allocated by the module itself, using a user-specified function
equivalent to `malloc()`. Because a new VM is instantiated for each function
call there is no need for a deallocation function in this proposal.

```
# returns NULL (0x00000000) on failure
func bazel_wasm_allocate(size: uint32) *uint8
```

No mechanism is provided for the WebAssembly code to do I/O or call back into
Bazel. All I/O must be performed from Starlark, using existing APIs such as
[`repository_ctx.file()`] or [`repository_ctx.read()`].

[`repository_ctx.file()`]: https://bazel.build/rules/lib/builtins/repository_ctx#file
[`repository_ctx.read()`]: https://bazel.build/rules/lib/builtins/repository_ctx#read

The Starlark code to execute a WebAssembly function looks something like this:

```starlark
def _toml_to_json(ctx):
    # ...
    toml_data = ctx.read(ctx.attr.toml)
    rc = ctx.execute_wasm(ctx.attr.toml_wasm, "toml_to_json", input = toml_data)
    if rc.return_code != 0:
        fail("toml_to_json failed: " + rc.output)
    ctx.file("example.json", rc.output)

toml_to_json = repository_rule(
    implementation = _toml_to_json,
    attrs = {
        "toml": attr.label(allow_single_file = True),
        "toml_wasm": attr.label(
            default = "//:tomlwasm/toml.wasm",
            allow_single_file = True,
        ),
    },
)
```

## `repository_ctx.load_wasm()`

If the same WebAssembly module is invoked multiple times (e.g. if it has several
functions, or is used to process multiple source files) then repeatedly loading
its bytecode from disk is inefficient. The `repository_ctx.load_wasm()` function
can be used to load the module once, and then use that loaded module in multiple
calls to `repository_ctx.execute_wasm()`.

```starlark
def repository_ctx.load_wasm(
  path: (str | path | Label),
  *,
  allocate_fn='bazel_wasm_allocate',
  watch='auto',
) -> (module: StarlarkWasmModule)
```

This function also acts as a configuration mechanism, so that parameters
related to the WebAssembly module as a whole have a place to live. In this
proposal the only such parameter is `allocate_fn`, which specifies the name of
the function used to allocate memory for the input buffers.

# Backward-compatibility

This proposal does not change existing functionality, so it's fully
backward-compatible with existing Bazel versions.

The proposed API was designed to impose few requirements on Bazel other than the
ability to execute WebAssembly bytecode. Future changes to the implementation
(such as using JNI to invoke a non-JVM implementation of WebAssembly) would not
affect the API exposed to Starlark.

WebAssembly itself has a strong culture of backwards-compatibility due to the
web browser use case. So long as Bazel only uses WebAssembly implementations
that conform to the WebAssembly core specification, the chances of breaking
backwards compatibility due to this feature are very low.

# Alternatives

A variety of alternatives have been suggested in discussions related to this
proposal.

## Download a pre-compiled WebAssembly runtime

There are several WebAssembly implementations written in languages that compile
to native code. These could be distributed as pre-compiled binaries that are
invoked by `repository_ctx.execute()`, with the ruleset inspecting
`repository_ctx.os` to determine which pre-compiled binary to download. The
WebAssembly implementation best suited for this approach is [wazero], which is
written in Go and has no third-party dependencies.

[wazero]: https://github.com/tetratelabs/wazero

Advantages:
- No changes would be required to Bazel itself.
- WebAssembly implementations that compile to native code would likely be faster
  than a pure-JVM implementation such as Chicory.
- The WebAssembly runtime would not be loaded into Bazel's address space.

Disadvantages:
- Rulesets would still be required to select and download pre-compiled native
  binaries, which is the core problem the proposal is intended to fix.
- Requires some person or group to be responsible for providing pre-compiled
  binaries for each relevant combination of OS and CPU.

An astute reader might notice that this approach is entirely independent from
Bazel, and that any ruleset willing to download a pre-compiled WebAssembly
runtime could have done so at any point in the last few years. There isn't
even any need to write the integration, as the wazero project has hosted
pre-compiled binaries on GitHub since its v1.0.0 release in March 2023 (two
years ago).

Thus there is an obvious question of whether any ruleset author would be interested in an approach that requires them to compile their helper logic to
WebAssembly but *also* requires them to download a pre-compiled binary to run
it.

If there is any utility to be found in this approach it would be as a fallback
for rulesets that want to immediately convert their helper logic to WebAssembly
but still need to support older Bazel versions. However, continuing to use the
existing native binaries would most likely be the path of least effort
in such cases.

## Compile a WebAssembly runtime in a repository rule

Instead of downloading a pre-compiled `wazero` binary, the wazero sources and
an appropriate Go toolchain could be downloaded and then a `wazero` binary
could be compiled in the repository rule. This would be similar to the approach
currently used by Gazelle.

Advantages:
- No changes would be required to Bazel itself.
- WebAssembly implementations that compile to native code would likely be faster
  than a pure-JVM implementation such as Chicory.
- The WebAssembly runtime would not be loaded into Bazel's address space.
- May be more portable in practice than pre-compiled binaries, as the Go project
  hosts pre-compiled Go toolchains for numerous platforms.

Disadvantages:
- It is doubtful that ruleset authors working in non-Go languages would be
  willing to accept an analysis-time dependency on the Go toolchain.
- Downloading and unpacking the Go toolchain takes a significant amount of time
  compared to simply instantiating a WebAssembly VM within Bazel.
- Compiling a Go binary within a repository rule requires complex build logic,
  which would need to be kept up to date with changes to the Go toolchain.

Similarly to the previous approach, the fact that this has not been attempted
despite lack of technical blockers indicates a likely lack of interest among
ruleset authors.

## Download a Java JRE and distribute helper tools as `.jar` archives

This is similar to downloading pre-compiled WebAssembly interpreter binaries,
but has three significant additional disadvantages:

- Pre-compiled binaries for the JRE are not available for as many platforms as
  Go can cross-compile to, so the set of supported platforms would be even
  more limited compared to current state.
- Existing tools often use libraries written in languages that are designed to
  compile to native code. There are no well-supported paths by which Go, Rust,
  or C/C++ could be compiled to JVM bytecode.
- The JRE is quite a bit larger than a pre-compiled WebAssembly runtime.
  Downloading it would add significant latency to the analysis phase.

## Download Chicory as a `.jar` and run it in a downloaded Java JRE

Largely the same as above, but solves the issue of poor compatibility with
Go/Rust/C/C++. Other downsides still apply.

## Download Chicory as a `.jar` and run it with Bazel's JRE

Instead of downloading a JRE, this approach would use the Bazel embedded JRE
to execute WebAssembly via Chicory.

The JRE bundled with Bazel is not currently exposed to repository rules, and
the Starlark API used for writing rules does not expose that Bazel is written in Java. Both of these properties seem like good ideas, and violating them would
significantly undermine Bazel's backwards compatibility.

The version of Chicory would be determined by ruleset authors. There would be
no way to predict which parts of the Java standard library might be depended on
by future versions of Chicory, so removing unused standard library classes from
Bazel's embedded JRE would no longer be possible.

## Embed a native WebAssembly runtime, invoke via JNI

Instead of embedding Chicory, embed a different WebAssembly runtime compiled
to native code as a dynamic library (`.dll` / `.so` / `.dylib`) and invoke it
via JNI. The [WebAssembly Micro Runtime (WAMR)] project would be a good
candidate for embedding.

[WebAssembly Micro Runtime (WAMR)]: https://github.com/bytecodealliance/wasm-micro-runtime

This approach would almost certainly result in faster WebAssembly execution than
Chicory's interpreter, but at the cost of increased complexity within the Bazel
build process and in the implementation of `execute_wasm()`.

Since the use of Chicory is not actually exposed to Starlark code, a future
version of Bazel could decide to switch the WebAssembly implementation to use
JNI if needed.

## Embed a native WebAssembly runtime binary under `@bazel_tools`

Similar to some of the above approaches, but it would embed a native executable into the Bazel self-extracting archive and then allow repository rules to
depend on it.

The benefits over JNI are that it would keep the WebAssembly logic in a separate
address space for fault isolation and/or resource accounting, at the cost of
additional build complexity.

## Allow Bazel to directly load JVM bytecode from a downloaded `.jar`

The JVM allows loading bytecode from `.jar` archives at runtime. Bazel could
expose this functionality via new methods on `repository_ctx`.

This approach would allow code in the `.jar` to directly access internal Bazel
APIs, requires the bundled JRE to have the full Java standard library included,
and wouldn't support helper logic written in non-JVM languages.

## Embed Chicory, but have Bazel fork before instantiating the VM

This is a minor variation on the original proposal, which would keep the
Chicory VM state in a separate address space from the main Bazel process.

The advantage is fault isolation and better resource accounting, the
disadvantage is increased complexity in the implementation of `execute_wasm()`.

A future version of Bazel could decide to adopt this approach by forking before
instantiating the WebAssembly VM and then communicating the output back via an
anonymous pipe or other IPC mechanism.

## Extend Starlark

It is currently difficult to write many types of code in Starlark, including
(but not limited to) recursive-descent parsers. Furthermore, support for
Starlark as a compilation target is not currently implemented in any of the
major compiler frameworks. These limitations are the root cause of why Bazel
ruleset authors frequently use helper binaries rather than write their entire
repository rule implementations in Starlark.

As a technical matter, if the necessary features were added to Starlark (such
as recursion, goto, and pointers to linear memory) then any helper tool
currently implemented in Go, Rust, or C/C++ could be rewritten and/or
transpiled to the Starlark language. Such a solution would eliminate the need
for WebAssembly as a second extension mechanism.

However, the author of this proposal believes that -- regardless of technical
possibility -- a desire to write all Bazel extension code in Starlark might not
be universal. It is possible that other ruleset authors would take
extraordinary measures (such as running away from human society to become a
mountain hermit, or switching to a different build system) to avoid writing
XML parsers in Starlark.

## Use WASI to make WebAssembly module execution similar to native binaries

One of the goals of the [WASI] project is to define a common set of POSIX-like
APIs for use by WebAssembly. Bazel could use WASI to offer a Starlark API similar to `repository_ctx.execute()`, with a WebAssembly module being able to
directly interact with the filesystem or network.

[WASI]: https://wasi.dev/

Without rendering judgement about whether such an idea is worth pursuing in the
future, at present it seems like it would be unnecessarily complex. The helper
tools used by Bazel rulesets generally do not need to interact with the
filesystem or network to fulfill their purpose, which also means the API surface
provided by Bazel can be very small.

The few helper tools that do need to interact with external resources (such as
`fetch_repo` from `rules_go`) can continue to be built and executed as native
binaries. If any ruleset authors wish to experiment with WASI, then some of the
other approaches discussed (such as a precompiled WASI-aware runtime binary)
could be used.
