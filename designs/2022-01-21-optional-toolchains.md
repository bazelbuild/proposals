---
created: 2022-01-21
last updated: 2022-01-27
status: In Review
reviewers:
  - gregce
  - trybka
title: Optional Toolchains
authors: katre
discussion thread: https://groups.google.com/g/bazel-dev/c/PdRF3Lmrln8/m/dzQk3vFkAQAJ
---

# Abstract

When
[toolchain resolution](https://docs.bazel.build/versions/5.0.0/toolchains.html)
were introduced, any rule that declared it required a toolchain type treated
that as **mandatory**: if a matching toolchain could not be found, that was an
error. One of the first toolchain-related feature requests was to relax that and
[make toolchains **optional**](https://github.com/bazelbuild/bazel/issues/3601),
allowing the rule implementation to decide what action to take if the toolchain
is not present.

One consequence of mandatory toolchains is the existence of "dummy" toolchains,
which exist only to satisfy toolchain resolution but are instead treated as
errors by rules. Because these are intended as a workaround for the lack of
optional toolchains, great care must be taken to ensure they are the
lowest-priority toolchains, or else they will hide actual useful toolchains.

In order to make toolchain resolution more useful, and remove these wasteful
"dummy" toolchains, it is time to introduce optional toolchains.

# Proposal

There are two possible ways to implement this feature.

## API Changes

A new API will be added to define a rule's requested toolchain types, which can
specify whether the toolchain type is mandatory or not.

In addition, the legacy string/label format will also be accepted, and will
assume that the toolchain type is mandatory. This allows rules to make no
changes and continue to have the same behavior as is currently implemented.

Current rules use this:

```py
foo_library = rule(
    implementation = _impl,
    attrs = { ... },
    toolchain_types = ["//toolchain:type1", "//toolchain:type2"],
    ...
)
```

With this API, the rule creation is instead:

```py
foo_library = rule(
    implementation = _impl,
    attrs = { ... },
    toolchain_types = [
        config.toolchain_type(
            "//toolchain:type1",
            mandatory = True,
        ),
        config.toolchain_type(
            "//toolchain:type2",
            mandatory = False,
        ),
    ],
    ...
)
```

This allows rule authors to decide which toolchain types are optional and which
are mandatory. Any bare labels or strings will be treated as mandatory (allowing
for easier migration).

The full API for `config.toolchain_type` would be:

| Parameter             | Type            | Meaning                           |
| --------------------- | --------------- | --------------------------------- |
| Positional            | String or Label | The toolchain type to be used.    |
| `mandatory`           | Boolean         | Whether or not the toolchain type must be found during  toolchain resolution. |

The `mandatory` argument to `config.toolchain_type` will default to True.

Other parameters can be added as needed.

**Note:** Any rule still using the current syntax will have all toolchain types
considered to be mandatory.

#### Sidebar: Why not a dict?

One option that seems simpler at first is to just use existing Starlark data
structures, such as lists and dicts, instead of a new `config.toolchain_type`
function:

```py
    toolchain_types = {
        "//toolchain:type1": True,
        "//toolchain:type2": {
            "mandatory": False,
        },
    },
```

While this seems simpler, it is much more confusing to read, and more difficult
to adapt in the future, so an explicit API is preferred.

## Changes to Toolchain Resolution Logic

The changes to toolchain resolution are fairly simple, and not directly related
to the API that rules use to define which toolchains are mandatory.

Currently, the toolchain resolution logic is this:

1.  For each toolchain type requested, create a map from execution platform to
    concrete toolchain (this happens in
    [`SingleToolchainResolutionFunction`](https://cs.opensource.google/bazel/bazel/+/master:src/main/java/com/google/devtools/build/lib/skyframe/SingleToolchainResolutionFunction.java;drc=77ebecce81128103d66b071cf12fed434b9a6c1c)).
2.  Find the highest-priority execution platform that satisfies every toolchain
    type (this happens in
    [`ToolchainResolutionFunction`](https://cs.opensource.google/bazel/bazel/+/master:src/main/java/com/google/devtools/build/lib/skyframe/ToolchainResolutionFunction.java)).
3.  This is now the selected execution platform and concrete toolchains.

To handle optional toolchains, step 2 will need to be changed. Instead of
finding the highest priority execution platform that specifies *all* toolchain
types, we find the highest priority execution platform that satisfies *the most*
toolchain types. Any execution platform that is missing a mandatory toolchain
type will still be removed from consideration.

"Priority" for both execution platforms and toolchains is by order of
registration.

1.  First, those added via flags (`--extra_execution_platforms` and
    `--extra_toolchains`)
2.  Then, those added in the WORKSPACE, or in files loaded from the WORKSPACE
    (via `register_execution_platforms` and `register_toolchains`)
3.  Finally, the host platform is always added as the lowest priority execution
    platform

As a simplified example, consider the following situation (which ignores the
host platform):

1.  `//platform:exec1` is registered first.
    1.  Toolchain `//toolchain:concrete2` (with type `//toolchain:type2`)
        matches this platform.
1.  `//platform:exec2` is registered second.
    1.  Toolchain `//toolchain:concrete3` (with type `//toolchain:type1`)
        matches this platform.
    1.  Toolchain `//toolchain:concrete4` (with type `//toolchain:type2`)
        matches this platform.

If the single toolchain type `//toolchain:type1` is requested, then the possible
execution platforms (step 1 above) are:

1.  `//platform:exec1`, with no toolchain.
2.  `//platform:exec2`, with `//toolchain:concrete3`.

Since `//platform:exec2` has more toolchain types matched, it is selected, even
tough it is lower priority than `//platform:exec1`.

If, on the other hand, `//toolchain:type2` is requested, the possible execution
platforms are:

1.  `//platform:exec1`, with `//toolchain:concrete2`.
2.  `//platform:exec2`, with `//toolchain:concrete4`.

In this case, both have all toolchain types fulfilled, so the higher-priority
`//platform:exec1` is selected.

Finally, if *both* `//toolchain:type1` and `//toolchain:type2` are requested,
then (following similar logic to the first example) the selected execution
platform will be `//platform:exec2` (with concrete toolchains
`//toolchain:concrete3` and `//toolchain:concrete4`).

**Open Question**: Are there any cases where users *would* prefer the
higher-priority execution platform to be selected, even with fewer toolchains
matched?

# Migration and Backwards Compatibility

With the new API for `toolchains`, rule authors who want to use optional
toolchains will need to check which Bazel version the rules are used with and
then decide which form to use. However, rule authors who are happy to continue
using mandatory toolchains will be able to continue as before, with no worry
that `ctx.toolchains[toolchain_type]` will evaluate to `None`.

Rule authors who choose to use optional toolchains should update their rule
implementations to handle `ctx.toolchains[toolchain_type]` evaluating to `None`.

# Possible future work

One possibility for future changes to update how missing toolchains are handled.
Instead of Bazel generating an error for missing optional toolchains (as happens
now), in the future Bazel could simply indicate that a target which cannot find
all mandatory toolchains is incompatible with the current target platform, and
use the existing support for
[skipping incompatible targets](https://docs.bazel.build/versions/5.0.0/platforms.html#skipping-incompatible-targets).

Instead of an API change, it could also be possible for rule implementations to
return the
[`IncompatiblePlatformProvider`](https://docs.bazel.build/versions/5.0.0/skylark/lib/IncompatiblePlatformProvider.html)
to indicate that a target which is missing a toolchain cannot be built.

# Alternatives Considered

## API Changes

### Option 1: All toolchains are optional

**Note:** This option was not selected, because the richer API makes migration
simpler and is a better match for existing APIs (such as attribute definition).

This option presents the simplest API: all toolchain types declared by a rule in
the `toolchains` parameter are optional, and could potentially not be present
when the rule implementation is executed. Rule implementations should check the
result of `ctx.toolchains[toolchain_type]` and raise an error if the value is
`None` and this cannot be handled.

Existing rules will need to migrate to handle this case, although this is
entirely for error handling, as any build that succeeds with mandatory
toolchains will also succeed with optional toolchains.
