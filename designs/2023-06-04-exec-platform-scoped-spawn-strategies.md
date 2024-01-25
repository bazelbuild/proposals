---
created: 2023-06-04
last updated: 2024-01-25
status: Under Review
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

# Valid as execution platform on host only
platform(name = "darwin_arm64", ...)

# Valid as exeution platform on remote only
platform(name = "linux_amd64", ...)
platform(name = "win_amd64", ...)

foo_binary(
    name = "foo-bin",
)
```

```ini
# //.bazelrc
build --spawn_strategy=remote
build --host_platform=//:darwin_amd64
build --extra_exec_platforms=//:linux_amd64,//:win_amd64
# + extras required for remote execution
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

Make it possible to declare spawn strategies supported by an execution platform.

```ini
# //.bazelrc
# Allow usage of "worker", "sandboxed" or "local" when exec platform is "//:darwin_arm64"
build --allowed_strategies_exec_platform//:darwin_arm64=worker,sandboxed,local
# Allow usage of "remote" when exec platform is "//:linux_amd64"
build --allowed_strategies_exec_platform=//:linux_amd64=remote
```

The goal behind this proposal is to address the capability gap (picking appropriate spawn strategies for execution platforms) while work on a more comprehensive solution continues ([#19904](https://github.com/bazelbuild/bazel/issues/19904)).
As such;
* Scope is narrow by design.
* API surface changes need to be minimal and optional (no migration).
* Implementation (in Bazel) needs to be simple, to minimise the backward compatibility burden if/when the subsystem is redesigned.

This does not seek to solve;
* `--remote_local_fallback` triggering fallbacks when remote and local environments are inconsistent. ([#7202](https://github.com/bazelbuild/bazel/issues/7202)) ([#15519](https://github.com/bazelbuild/bazel/issues/15519))
* Unifying execution strategies and platforms, though this proposal seeks to close the gap. ([#11432](https://github.com/bazelbuild/bazel/issues/11432))
  Specifically, this proposal will not allow Bazel to pick another execution platform when no spawn strategy is usable.

## Flags

The likihood of misconfiguration can be reduced with;
- `--[no]strict_exec_platform_strategies` to require that every exec platform (including host) have a strategies allowlist (checked after workspace evaluation).

## Dynamic Execution

Strategies supplied for the `dynamic` meta-strategy via `--dynamic_remote_strategy` and `--dynamic_local_strategy` will follow the same strategy support rules.

## Risks

### Actions Lacking Execution Platform

It is theoretically possible for an action which relies on spawn strategies (`ctx.actions.run` and `ctx.actions.run_shell`) to lack an execution platform, though almost certainly a bug (`exec_properties` wouldn't be included for `remote` spawns).

An [investigation](https://github.com/bazelbuild/bazel/issues/20505#issuecomment-1856144402) by [`@katre`](https://github.com/katre) indicates that at present only file write actions (`ctx.actions.write`, `ctx.actions.symlink`, `ctx.actions.expand_template`) lack a platform (their platform is inherently always that of the host), however there are no guards to enforce this.

An option is to accept the risk that a bug could sneak in later, and fix it when discovered. Alternatively Bazel could be refactored internally so that invocations of `getExecutionPlatform` never return `null`.

Source code references;
`SpawnStrategyRegistry#getStrategies(Spawn,EventHandler)` can retrieve a `PlatformInfo` instance for the execution platform via `Spawn#getExecutionPlatform()`, however it is annotated as nullable. A quick search showed `null` being returned in the following places.
- `ActionTemplate#getExecutionPlatform()`, described as a placeholder action that expands out to a list later.
- `MiddlemanAction#getExecutionPlatform()`, a non-executable action.
- `RuleContext#getExecutionPlatform()`, null if `getToolchainContext()` returns null.
- `RuleContext#getExecutionPlatform(String)`, null if `getToolchainContext()` returns null and no toolchain exists for the specified exec group.
- `SymlinkAction#getExecutionPlatform()`, a non-executable action.
- `SolibSymlinkAction#getExecutionPlatform()`, a non-executable action.
- `TestActionKeyCacher#getExecutionPlatform()`, test implementation detail.
- `ResourceOwnerStub#getExecutionPlatform()`, nullable but throws, test implementation detail.
- `FakeResourecOwner#getExecutionPlatform()`, test implementation detail.

# Backward-compatibility

All changes are opt-in and additive. Compatibility breaks would indicate an implementation flaw. Builtin rules may have some unknowns.

# Alternatives Considered

## Declare Support in Platform Declaration

A variation on the proposal where supported strategies are declared directly on `platform` via a new `supported_strategies` attribute was considered and ultimately rejected as;

* The new attribute may be used by rulesets, increasing the cost of removal if/when the system is redesigned and implicitly increasing the scope of the proposal.
* Platforms used as execution platforms may be defined in an external repository, making them harder to configure.

It is also worth considering the current status of the `platform(...)` abstraction, which is currently 2 things;
1. A collection of constraints. (`constraints`)
2. An opaque collection of properties for remote execution. (`exec_properties`)

(1) has value for all target and execution platform use cases, while (2) is limited to a subset of execution platform use cases (configuring remote execution services). It is probable the `exec_properties` attribute will be relocated in a future redesign of the spawn strategy subsystem.

## Extending Existing Spawn Strategy Flags

The initial version of this proposal sought to extend the existing spawn strategy flags.

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

This was rejected as the existing interface was already complicated and the change would increase complexity. As the existing interface is already complex, there is also a risk the required changes would lead to a compatibility break (a-la Hyrum's Law).
