---
created: 2023-06-08
last updated: 2023-06-14
status: Draft
reviewers: gregestren
title: Platform-based Flags
authors: katre
discussion thread: 
---

# Overview

Currently, users may need to set several flags in concert as part of a build,
even if they are logically related: the `--platforms` flag sets the target
platform, but related concepts like the Android API level or location of a
custom C malloc library are commonly used with the same target.

Users can get an effect like bundling these flags together by adding a
[`platform_mappings`
file](https://bazel.build/concepts/platforms#platform-mappings) to set the
needed flags based on the value of `--platforms`. However, this is a workaround
to the fact that not every facet of a build can be expressed via platform
constraints.

A simpler alternative would be to allow the `platform` rule to directly list
flags that will be set when the platform is the target platform. This allows
users to centralize definitions of common targets, and with platform
inheritance, can help make the logic more explicit.

# Syntax

A short example of a platform with flags would look like:

```py
platform(
    name = "custom",
    constraint_values = [ … ],
    flags = [
        "--custom_malloc=//label",
        "--other_flag",
        "--//example/starlark:flag=23",
    ],
)
```

In this case, when the `:custom` platform is the target (either via a command
line `--platforms=:custom` flag, or via a transition later during the build),
these flags would be added to the configuration.

An example with inheritance would look like:

```py
platform(
    name = "parent",
    constraint_values = [ … ],
    flags = [
        "--flag=1",
    ],
)
platform(
    name = "custom",
    parents = [":parent"],
    flags = [
        "--flag=2",
    ],
)
```

The new `flags` attribute is a simple list of strings, in the same format they
would be passed at the command line or in the platform mappings file.

## Rejected Alternative: Starlark Dict type

Another alternative is to represent the `flags` attribute as a Starlark dict,
with the keys being flag names and the values being flag values. This might be
slightly clearer, but has a number of problems:

### Mixed types

If the `flags` was a dict, users would expect to be able to specify boolean,
int, or Label-valued flags with the Starlark bool, number, or Label types
respectively. However, Starlark does not allow dicts with mixed types for
values, so users would have to convert the values to string anyway, causing
confusion and errors.

### Parsing already exists

The implementation of this feature has to re-use the existing flag parser,
which already knows how to convert strings to the data types needed for each
flag. Splitting the flag and value would just complicate this and require
deeper changes to the existing parser.

# Semantics

## Flag parsing

Flags will be parsed as they appear on the command line. Relative labels for
label-valued flags will not be supported.

**Open questions:**

* Do flags on the command line override platform flags, or vice versa?

## Flag errors

Any errors will be reported and handled when the platform is used as a target
platform: there will be no -processing of the `flags` attribute to check for
invalid flags.

## Mixed platform-based flags and platform mappings

Platform mappings will continue to be applied after platform-based flags are
added, allowing flags to be migrated one-by-one if desired. If the same flag is
repeated in both the platform and the mappings file, that may cause issues
depending on the flag’s specific semantics.

# Compatibility

This is a new feature, and is intended to obsolete part of the existing
platform mappings file. Projects should be able to easily migrate from platform
mappings to platform-based flags, and until they migrate a specific platform
this feature will have no effect, leading to easy migration and rollback.

