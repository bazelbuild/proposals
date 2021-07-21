---
created: 2021-07-08
last updated: 2021-07-21
status: Dropped
reviewers:
title: Monorepo Toolchain Registration
authors:
  - katre
discussion thread:
---

# Monorepo Toolchain Registration

## DROPPED

This proposal has been dropped as not providing enough benefit to be worth
finalizing and implementing.

# Original Proposal

Although the current method of [registering
toolchains](https://docs.bazel.build/versions/main/toolchains.html#registering-and-building-with-toolchains)
is very flexible, it is also very centralized. The current system is to:

- First check the list of toolchains given via the `--extra_toolchains` flag.
- Then check the toolchains registered in the WORKSPACE file (potentially via
  macros), in the order that they are registered.

This gives the user control over which toolchains are available, but requires
that all toolchains be registered in a single, central location. One failure
mode we've seen inside Google is that updates to WORKSPACE cause every target to
require re-analysis, which is very expensive given the scale of our monorepo.

In a monorepo, there is therefore a desire to register toolchains in a
non-centralized, per toolchain location. Conveniently, there is such a target:
the `toolchain_type` target that is used to define the type of toolchain.
Projects that do not wish to opt in to this style can continue to use the
previous system, or blend the two.

This would only be appropriate for true monorepos, where the `toolchain_type`
targets used to define toolchains are available to be edited, instead of being
loaded via an external repository.

# Proposal

The proposal therefore is to add a new (non-dependency) label list attribute to
the existing `toolchain_type` rule definition, called `registered_toolchains`.
This would give the path to targets of the `toolchain` rule, exactly the same as
would be registered via the WORKSPACE `register_toolchains` function.

It would be an error to use `registered_toolchains` to list toolchains of a
different type than the actual `toolchain_type` the attribute is for. This can
be checked and reported when the toolchain type is used.

Example:

```python
toolchain_type(
    name = "my_toolchain_type",
    registered_toolchains = [
        ":toolchain_declaration",
    ],
)

my_toolchain(
    name = "toolchain_impl",
    ...
)

toolchain(
    name = "toolchain_declaration",
    toolchain_type = ":my_toolchain_type",
    toolchain = ":toolchain_impl",
    ...
)
```

In
[SingleToolchainResolutionFunction](https://cs.opensource.google/bazel/bazel/+/master:src/main/java/com/google/devtools/build/lib/skyframe/SingleToolchainResolutionFunction.java),
in addition to adding the toolchains from `--extra_toolchains` and
`register_toolchains` (returned from
[RegisteredToolchainsFunction](https://cs.opensource.google/bazel/bazel/+/master:src/main/java/com/google/devtools/build/lib/skyframe/RegisteredToolchainsFunction.java)),
we can add the toolchains that are directly declared on the `toolchain_type`.

## Migration

Migration is fairly simple: repositories that want to use the new system can add
the toolchains currently registered in WORKSPACE to the `toolchain_type`
declaration, and then remove the `register_toolchains` calls in WORKSPACE later.

