---
created: 2023-06-04
last updated: 2023-12-07
status: Draft
reviewers:
  - katre
title: Execution Platform Scoped Spawn Strategies
authors:
  - Silic0nS0ldier
---

# Abstract

Bazel has the primitives to correctly configure a multi-platform build with regard to inputs, but not spawn strategies. This proposal seeks to address the capability gap by building on existing concepts.

# Background

There are several spawn strategies;
- `remote`
- `worker`
- `sandboxed`
- `linux-sandbox`
- `processwrapper-sandbox`
- `local`

And several flags to control which spawn strategy is selected;
- `--spawn_strategy`, how actions are spawned by default.<br/>
  `--spawn_strategy=remote,worker,sandboxed,local`
- `--strategy`, how an action with the specified mnemonic is spawned.<br/>
  `--strategy=FOO=local,BAR=remote`
- `--strategy_regexp`, how an action whose description matches the specified regular expression is spawned.<br/>
  `--strategy_regexp=//foo.*\.cc,-//foo/bar=local`

Collectively the flags and available spawn strategies offer a great deal of control, but not enough to cover all multi-platform build scenarios.

For example;

```starlark
# //BUILD.bazel

# Unsupported on remote
platform(
    name = "darwin_arm64",
    ...
)

platform(
    name = "linux_amd64",
    ...
)

platform(
    name = "win_amd64",
    ...
)

foo_binary(
    name = "foo-bin"
)

```

```ini
# //.bazelrc
build --spawn_strategy=remote
build --host_platform=//:darwin_amd64
build --extra_exec_platforms=...
# extra bits for remote connection
```

```sh
# Shell at //
bazel build //:foo-bin --platforms=//:darwin_arm64,//:linux_amd64,//:win_amd64
```

> [!NOTE]
> Currently `--platforms` only supports [one target platform](https://github.com/bazelbuild/bazel/issues/19807), this example is illustrative.
> In practise multi-platform builds occur when transitions are used.

In this scenario _all_ actions will be executed on the remote or pulled from its cache. If `//:foo-bin` has previously been built with the exact same inputs (subject to configuration impacting output paths) the build will pass thanks to the cache hit, otherwise the build will fail (e.g. attempted to run darwin executable on Linux or found no suitable executor).

There are ways to work around this problem, but all have costs.
- Build targets and platforms with unique execution requirements in a separate invocation.
  This increases build command complexity.
- Customise mnemonics and/or descriptions such that they vary across execution platforms, and target spawn strategies accordingly.
  This keeps commands simple, but increases build configuration complexity. Patching rulesets may also be necessary.

For this trivial scenario the challenges can be overcome with relative ease, however this does not scale well.

# Proposal

Support scoping of existing spawn strategy flags to a specific execution platform.

e.g.

```ini
# //.bazelrc
# When the exec platform is "//:darwin_arm64" use one of "worker,sandboxed,local" by default
build --spawn_strategy=//:darwin_arm64=worker,sandboxed,local
# For actions with the "FOO" mnemonic and exec platform "//:darwin_arm64" use "local"
build --strategy=FOO=//:darwin_arm64=local
# For actions with a description which matches regex "//foo.*\.cc" and exec platform "//:darwin_arm64" use "local"
build --strategy_regexp=//foo.*\.cc=//:darwin_arm64=local
```

The motivation behind this approach is;
1. Implementation simplicity. A prototype should not require any significant changes to Bazel internals.
2. Existing strategy flags are easy for scripts to interact with. e.g. an opt-in remote support script can be run before a build to ensure a tunnel is opened and write out a `.bazelrc` file which is read with `try-import`.

## Flags

The likihood of misconfiguration can be reduced with;
- `--[no]require_platform_scoped_strategies` to make inclusion of platforms in spawn strategy flags mandatory.
- `--[no]exhaustive_platform_strategies` to make specifiction of spawn strategies for every registered execution platform required.

## Priority

Currently;
1. Description (`--stategy_regexp=<regex>=...`), unless mnemonic is `TestRunner`.
2. Mnemonic (`--strategy=<mnemonic>=...`).
3. Defaults (`--spawn_strategy=...` or Bazel generated defaults).

This will become;
1. Platform + description (`--stategy_regexp=<platform=<regex>=...`), unless mnemonic is `TestRunner`.
2. Description (`--stategy_regexp=<regex>=...`), unless mnemonic is `TestRunner`.
3. Platform + mnemonic (`--strategy=<platform>=<mnemonic>=...`).
4. Mnemonic (`--strategy=<mnemonic>=...`).
5. Platform defaults (`--spawn_strategy=<platform>=...`).
6. Defaults (`--spawn_strategy=...` or Bazel generated defaults).

## Risks

`SpawnStrategyRegistry#getStrategies(Spawn,EventHandler)` can retrieve a `PlatformInfo` instance for the execution platform via `Spawn#getExecutionPlatform()`, however it is annotated as nullable. A quick search showed `null` being returned in the following places.
- `ActionTemplate#getExecutionPlatform()`, described as a placeholder action that expands out to a list later.
- `MiddlemanAction#getExecutionPlatform()`, a non-executable action.
- `RuleContext#getExecutionPlatform()`, null if `getToolchainContext()` returns null.
- `RuleContext#getExecutionPlatform(String)`, null if `getToolchainContext()` returns null and no toolchain exists for the specified exec group.
- `SymlinkAction#getExecutionPlatform()`, a non-executable action.
- `SolibSymlinkAction#getExecutionPlatform()`, a non-executable action.
- `TestActionKeyCacher#getExecutionPlatform()`, test implementation detail.
- `ResourceOwnerStub#getExecutionPlatform()`, nullable but throws, test implementation detail.
- `FakeResourecOwner#getExecutionPlatform`, test implementation detail.

It is plausible that for all `Spawn` interface implementations `null` is never returned by `getExecutionPlatform()`. If this can be proven, the nullable annotations should be removed to reflect the changed requirements. That being the execution platform must be known so the correct spawn strategy can be selected. At least 1 unreferenced code path noted that `null` is reserved for the host platform.

## Dynamic Execution

The current draft of this proposal has not deeply explored interactions with dynamic execution, however since dynamic execution relies on using the same configuration for local and remote spawns this is likely to be a complementary addition.

The `dynamic` spawn strategy type itself naturally fits with the changes proposed to "standard" flags. The dynamic execution specific flags would likely be changed in a similar manner.

e.g.

```ini
build --spawn_strategy=//:linux_amd64=dynamic
# There is currently only 1 remote strategy, so this is technically unnecessary
build --dynamic_remote_strategy=//:linux_amd64=<mnemonic>=<strategy>
build --dynamic_remote_strategy=//:linux_amd64=<strategy> # default for exec platform
build --dynamic_local_strategy=//:linux_amd64=<mnemonic>=<strategy>
build --dynamic_local_strategy=//:linux_amd64=<strategy> # default for exec platform
```

# Backward-compatibility

All changes are opt-in and additive. Compatibility breaks would indicate an implementation flaw. Builtin rules may have some unknowns.
