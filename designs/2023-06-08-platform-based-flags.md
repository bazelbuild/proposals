---
created: 2023-06-08
last updated: 2023-08-14
status: Approved
reviewers:
  - gregestren
  - sdtwigg
title: Platform-based Flags
authors:
  - katre
discussion thread: https://github.com/bazelbuild/bazel/discussions/18672
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

The new `flags` attribute should not be configurable.

Some flags should be disallowed in the `flags` attribute:

-  `--platforms`: This leads to circular evaluation
   -  Possibly other platform-like flags (such as `--android_platforms`) should also be disallowed?

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

When the target platform is changed, the flags are applied to the
configuration. There are two possible paths, depending on whether the target
platform is being set at the top level or via a transition.

Regardless of which of these two cases happen, there are some consistent
semantics:

-  If platform-based flags are used, platform mapping will be skipped
   -  This avoids confusion about the source of flag values.
-  Setting the target platform to the same value does not re-apply any flags
   -  Any flags which are present on the platform but have changed will not be
      reset.
   -  Platform mapping will still be skipped.

## Flag parsing

Flags will be parsed as they appear on the command line. Relative labels for
label-valued flags will not be supported.

## Flag errors

Any errors will be reported and handled when the platform is used as a target
platform: there will be no -processing of the `flags` attribute to check for
invalid flags.

## Initial Configuration

When the top-level target platform is set during flag parsing and Bazel
startup, Bazel needs to consider the possibility that there are multiple
`--platforms` flags present (either directly from the command line, from a
bazelrc file, or using the default value).

With `--platforms`, the last value is the one kept, and therefore should be the
one to use to load flags. Bazel should __not__ add flags from `--platforms`
flags that are not actually used for the configuration.

Ideally, the flags from the selected target platform should have the same
priority as the `--platforms` flag itself. Specifically, this means that later
uses of the same flag should override/add to the flag (based on whether
`allowMultiple` is true).

Given the following files:

```sh
$ cat project.bazelrc
config:foo --java_header_compilation=true
config:foo --platforms=//:foo_platform
config:foo --java_deps=true

$ cat BUILD.bazel
platform(
    name = "foo_platform",
    flags = [
        "--java_header_compilation=false",
        "--java_deps=false",
    ],
)
```

When calling `bazel build --config=foo //some:target`, the flags have the
values:

-  `--platforms=//:foo_platform` (from the bazelrc config)
-  `--java_header_compilation=false` (from the `platform.flags`)
-  `--java_deps=true` (initially set by `platform.flags` but then overridden in
   the bazelrc)

**Note:** These semantics may be too complicated for Bazel's current options
parsing. If that turns out to be the case, Bazel will need to detect that flags
are set both explicitly at the command line/bazelrc and in the platform, and
emit a warning.

## Transitions

When the `--platforms` flag is changed in a transition (whether native or
Starlark), flags from the platform (if any) need to be applied. However, any
flag values from the actual transition should override the values from the
platform. Because of this, the order of changes for a transition is:

1. Check if the value of `--platforms` has changed.
   1. If it has, find the flags for the new platform and apply them to the
      configuration.
2. Apply the rest of the flag changes.

This is relatively simple for Starlark transitions, which return a simple list
of flag changes. For native transitions, which instead operate directly on
[`BuildOptionsView`
instances](https://cs.opensource.google/bazel/bazel/+/master:src/main/java/com/google/devtools/build/lib/analysis/config/BuildOptionsView.java?ss=bazel&q=BuildOptionsView),
processing will be more difficult. If it turns out to be difficult to apply
these semantics, then we may need to change the ~30 native transitions to use
an API more similar to Starlark transitions.

### Transitions That Unset Platforms

Some transitions currently unset the `--platforms` flag in order to signal that
the actual target platform should be selected by the platform mappings system.
In this case, the platform's flags will be applied after the transition's
flags.

This is a problem if the platform's flags do not match what is in the mappings
file. As an example:

```sh
$ cat platforms/BUILD
platform(
    name = "device",
    constraint_values = [...],
    flags = [
        "--cpu=arm32",
        "--//custom_flag=true",
    ],
)

$ cat platform_mappings
flags:
    --cpu=arm64
        //platforms:device
```

If a transition unsets `--platforms` and sets `--cpu=arm64`, the selected
platform (due to the platform mappings) will be `//platforms:device`, which
will then apply its flags and change `--cpu` to `--cpu=arm32`. This is
confusing, but can be detected and a warning issued, and is arguably an error
that should be corrected.

A more difficult case is where the transition, in addition to unsetting
`--platforms` and setting `--cpu=arm64`, also sets `--//custom_flag=false`.
This change will also be overridden by the platform's flags, and will not issue
a warning.

# Compatibility

This is a new feature, and is intended to obsolete part of the existing
platform mappings file. Projects should be able to easily migrate from platform
mappings to platform-based flags, and until they migrate a specific platform
this feature will have no effect, leading to easy migration and rollback.

