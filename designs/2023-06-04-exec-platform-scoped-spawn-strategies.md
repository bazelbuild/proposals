---
created: 2023-06-04
last updated: 2023-08-12
status: Draft
reviewers:
  -
title: Execution platform scoped spawn strategies
authors:
  - Silic0nS0ldier
---


# Abstract

<!-- This section gives a short summary of the proposal. -->
Bazel has the primitives to correctly configure a multi-platform build with regard to inputs, but not spawn strategies. This proposal seeks to address the capability gap by building on existing concepts.


# Background

<!-- This section can give context. For example, it can explain the current state and
have pointers to previous design documents. -->
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

In this scenario _all_ actions will be executed on the remote or pulled from its cache. If `//:foo-bin` has previously been built with the exact same inputs (and configuration???) the build will pass thanks to the cache hit, otherwise the build will fail (e.g. attempted to run darwin executable on Linux or found no suitable executor).

There are ways to work around this problem, but all have costs.
- ...separate builds for platforms not buildable with remote...
- ...somehow customise mnemonics or description...

For this trivial scenario the challenges can be overcome with relative ease, however this does not scale well. ...dependencies...


# Proposal

<!-- The actual proposal. This should be detailed enough for users to understand the
change, its motivations, and the rationale. When appropriate, include rejected
alternatives (and why they were rejected).

The current file is a template. Copy it, update it to your needs. Feel free to add or
remove sections. -->
Support scoping of existing spawn strategy flags to a specific execution platform.

e.g.

```ini
# //.bazelrc
build --spawn_strategy=//:darwin_arm64=worker,sandboxed,local
build --strategy=FOO=//:darwin_arm64=local
build --strategy_regexp=//foo.*\.cc=//:darwin_arm64=local
```

The motivation behind this approach is;
1. Implementation simplicity. A prototype should not require any significant changes to Bazel internals.
2. Existing strategy flags are easy for scripts to interact with. e.g. an opt-in remote support script can be run before a build ensure a tunnel is opened and write out of a `.bazelrc` file which is read with `try-import`.

...

The likihood of misconfiguration can be reduced with;
- `--[no]require_platform_scoped_strategies` to make inclusion of platforms in spawn strategy flags mandatory.
- `--[no]exhaustive_platform_strategies` to make specifiction of spawn strategies for every registered execution platform required.


## Priority

Currently;
- Description (`--stategy_regexp`), unless mnemonic is `TestRunner`.
- Mnemonic (`--strategy`).
- Defaults (`--spawn_strategy` or Bazel generated defaults).

This will become;
- Platform + description.
- Description.
- Platform + mnemonic.
- Mnemonic.
- Platform defaults (user specified only).
- Defaults.

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

# Backward-compatibility

<!-- Describe here how this proposal impacts backward compatibility. If the proposal
is implemented, can it possibly break any user? -->
All changes are opt-in and additive. Compatibility breaks would indicate an implementation flaw. Builtin rules may have some unknowns.
