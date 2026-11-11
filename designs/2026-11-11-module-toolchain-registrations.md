# List MODULE toolchain registrations in mod graph

| | |
| --- | --- |
| Author | [Tamir Duberstein](https://github.com/tamird) |
| Last major change | 2026-11-11 |
| Status | Draft |
| Lead reviewer | To be assigned |
| Discussion | [Tracking issue](https://github.com/bazelbuild/bazel/issues/23104) |

## Motivation

A tool that inventories SDKs or computes the impact of a compiler change needs to know which toolchain declarations the resolved modules register. Two modules can give the same apparent repository name different meanings. A wildcard can expand to multiple declarations. Even when the declarations and compiler implementations stay unchanged, reversing their registration order can change the selected compiler.

## Proposed interface

Add `--include_toolchain_registrations` to `bazel mod graph --output=json`. When requested, add an array named `moduleToolchainRegistrations` to the root JSON object. For example, two explicit registrations from the main module could produce the following field in the graph object.

```json
{
  "moduleToolchainRegistrations": ["//:first", "//:second"]
}
```

Use the same module traversal, per-module repository mapping, target-pattern expansion and wildcard filter that Bazel uses for MODULE toolchain registrations. Return canonical labels in registration order, keeping the first occurrence of each label. Explicit labels retain their identity, including aliases and explicit non-toolchain targets. Clients can follow an alias and inspect the native declaration's type and implementation. If a pattern cannot be fully expanded, fail the command before writing a partial JSON result. Successfully expanding zero registrations yields an empty array.

The list describes the module contribution to resolution. Configuration-specific `--extra_toolchains`, platform eligibility, and the winning implementation still require the appropriate configuration. Execution-platform registrations would need a separate interface.

## Alternatives and cost

Reading MODULE files directly would require clients to reproduce evaluation, repository mappings and wildcard expansion. Querying for every native `toolchain` rule also returns declarations that nobody registered and loses registration order. A configured query can answer which compiler a particular consumer uses, but requires an analysis universe and does not enumerate MODULE registrations.

The optional field reuses Bazel's existing registration and target-pattern machinery. Expanding a pattern can load packages and fetch repositories. Default graph output performs no additional expansion and keeps its current schema. Existing JSON consumers require no migration.

## Validation

An integration test builds a consumer that writes the name provided by its selected toolchain. Swapping two registration entries changes both the ordered array and the output from `a` to `b`. Additional cases cover repository mappings, aliases, wildcards, duplicate registrations, missing targets and partial wildcard expansion.

Related proposals expose toolchain types used by rules. Clients can combine these interfaces to associate candidate declarations with types. [Ordinary query inputs](https://github.com/bazelbuild/bazel/pull/32431) and [structured type requirements](https://github.com/bazelbuild/bazel/pull/32438).
