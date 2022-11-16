---
created: 2022-08-03
last updated: 2022-11-16
status: Dropped
reviewers: @gregestren
title: Platforms on Targets
authors: katre
discussion thread: https://groups.google.com/g/bazel-dev/c/QK7CI__ReDM
---

**Note**: This proposal is unable to satisfy all users and rules, and so is dropped.

# Background

Most Bazel targets, especially library targets, and inherently multi-platform:
they are intended to be built and used on any type of target platform that a
user may need to build them for. Other targets, however, are inherently
single-platform:

-  Many binaries are single-platform: Android or iOS apps, Windows-specific
   executables, or tools that use low-level Linux syscalls
   -  As a subset of binaries, many tests are single-platform
- Some libraries are also single-platform: Android or iOS libraries, or
  libraries that have deep integration with the operating system

For these use cases, it would be useful to be able to directly express the
intended target platform directly in the BUILD file, rather than requiring users
to always pass the correct value on the command line, or use convoluted build
scripts.

# Proposal

In order to simplify the work and easy migration, support for platforms on
targets will be added in three phases:

1. Binary rules
2. Test rules
3. (Potentially) All other rules

Stage 3 may be delayed or dropped based on the progress with the first two, and
whether or not users need this feature.

Bazel will support specifying a single platform per target. Some build rules
(such as builds for Android or iOS) allow builds for so-called "fat" binaries
with multiple CPU architectures. These rules will need their own support for
these richer sets of platforms, presumably via their own flags and attributes,
and will not be handled via this proposal.

## Syntax

To support declaring platforms on targets, a new attribute will be added to
rules by default, called `platform`. The attribute will take a label as a value,
be optional with no default value, and will require the value to provide the
PlatformInfo provider. The attribute will not be configurable (so a select
cannot be used to set the value).

```
platform(
    name = "windows",
    constraint_values = [
        "@platforms//cpu:x86_64",
        "@platforms//os:windows",
    ],
)

cc_binary(
    name = "some_binary",
    srcs = [...],
    deps = [...],
    platform = ":windows",
)
```

## Semantics

The semantics of the new attribute will change based on whether the target is
being built as a top-level target or not.


### As a top-level target

When built directly, the `platform` attribute will be used to set the target
platform used to build this target. If multiple targets with different values
for `platform` are built at once, then there will be effectively a top-level
split transition, with each target taking a different target platform (and thus
using different configurations).

If the `--platforms` command line value is also set, and it conflicts with the
`platform` attribute for a target, that target will not build but will show an
error.


### When not a top-level target

When a target is being configured, and the value of the `platform` attribute is
different from the target platform for the configuration, that will be an error
and the target will not be built.

This can be used to detect and fail currently-allowed but nonsensical builds,
such as an `android_binary` being built as a tool for a `genrule`.

**Open question:** Should there be a flag to switch from an error to a warning,
to allow current builds to continue to succeed during a migration period?


### When are platforms different?

The `--platforms` flag and the `platform` attribute must match exactly to avoid
a conflict. Even if the two platformshave the same constraints, that will still
be a conflict.

**Open question:** Are aliases to the same target allowed?


## Platform attribute and `target_compatible_with`

The `platform` attribute and the `target_compatible_with` attribute have similar
but distinct functionality. The `platform` attribute will set the target
platform for a top-level target, and signal problems with the configuration for
tests. The `target_compatible_with` attribute, on the other hand, defines
constraints for the target platform used for a target, and does not affect the
top-level builds.


## Multiple output directories

Building multiple targets with different `platform` attributes (or with no
`platform` attribute, but using a different `--platforms` flag) will cause each
target to have a distinct configuration and thus a distinct output directory.
Bazel will need to be careful when reporting the output artifacts for these
targets that the different directories do not confuse users.

# Implementation Notes

## Defining the attribute

The attribute can be defined separately for binary rules on
[`BaseRuleClasses.BinaryBaseRule`](https://cs.opensource.google/bazel/bazel/+/master:src/main/java/com/google/devtools/build/lib/analysis/BaseRuleClasses.java;bpv=1;bpt=1;l=493?ss=bazel&q=BaseRuleClasses.BinaryBaseRule&gsn=BinaryBaseRule&gs=kythe%3A%2F%2Fgithub.com%2Fbazelbuild%2Fbazel%3Flang%3Djava%3Fpath%3Dcom.google.devtools.build.lib.analysis.BaseRuleClasses.BinaryBaseRule%2393148b7296fa69e113c4f0dadfd0cf353460da4aca0059ed1cb75c16e7c99057)
(for native rules) and
[`StarlarkRuleClassFunctions.binaryBaseRule`](https://cs.opensource.google/bazel/bazel/+/master:src/main/java/com/google/devtools/build/lib/analysis/starlark/StarlarkRuleClassFunctions.java;bpv=1;bpt=1;l=160?ss=bazel&q=StarlarkRuleClassFunctions.binaryBaseRule&gsn=binaryBaseRule&gs=kythe%3A%2F%2Fgithub.com%2Fbazelbuild%2Fbazel%3Flang%3Djava%3Fpath%3Dcom.google.devtools.build.lib.analysis.starlark.StarlarkRuleClassFunctions%2347117d03b68de9ae1b41031ab4380dd5cd9ede141a9600f0740f447e5db428cf)
(for Starlark rules). Test rules use
[`BaseRuleClasses.TestBaseRule`](https://cs.opensource.google/bazel/bazel/+/master:src/main/java/com/google/devtools/build/lib/analysis/BaseRuleClasses.java;bpv=1;bpt=1;l=179?ss=bazel&q=BaseRuleClasses.BinaryBaseRule&gsn=TestBaseRule&gs=kythe%3A%2F%2Fgithub.com%2Fbazelbuild%2Fbazel%3Flang%3Djava%3Fpath%3Dcom.google.devtools.build.lib.analysis.BaseRuleClasses.TestBaseRule%230f2469a29e6698068c45de31465a889b43dba1783078f4cd675053c316e6d7e7)
and
[`StarlarkRuleClassFunctions.getTestBaseRule`](https://cs.opensource.google/bazel/bazel/+/master:src/main/java/com/google/devtools/build/lib/analysis/starlark/StarlarkRuleClassFunctions.java;bpv=1;bpt=1;l=167?ss=bazel&q=StarlarkRuleClassFunctions.binaryBaseRule&gsn=getTestBaseRule&gs=kythe%3A%2F%2Fgithub.com%2Fbazelbuild%2Fbazel%3Flang%3Djava%3Fpath%3Dcom.google.devtools.build.lib.analysis.starlark.StarlarkRuleClassFunctions%239ed7d5f9d42b79a2e6409bb1304c7be9eba8b2d6c7dc139bbd74fff61334529e).

## Setting the target platform

In either
[`BuildView.update`](https://cs.opensource.google/bazel/bazel/+/master:src/main/java/com/google/devtools/build/lib/analysis/BuildView.java;bpv=1;bpt=1;l=201?q=BuildView.update&ss=bazel&gsn=update&gs=kythe%3A%2F%2Fgithub.com%2Fbazelbuild%2Fbazel%3Flang%3Djava%3Fpath%3Dcom.google.devtools.build.lib.analysis.BuildView%23b2fd296e3a814fe2b9f5ac2ec279e0cea82a8278dd8cc6693438b925adb03811)
or
[`AnalysisUtils.getTargetsWithConfigs`](https://cs.opensource.google/bazel/bazel/+/master:src/main/java/com/google/devtools/build/lib/analysis/AnalysisUtils.java;bpv=1;bpt=1;l=185?q=AnalysisUtils&ss=bazel&gsn=getTargetsWithConfigs&gs=kythe%3A%2F%2Fgithub.com%2Fbazelbuild%2Fbazel%3Flang%3Djava%3Fpath%3Dcom.google.devtools.build.lib.analysis.AnalysisUtils%23607376498c2a7dd4b3f030ebc00dbc6d7ea0aee7c82977286726fadee4952e59),
the configuration used for each target will need to be adjusted to set the
target platform based on the `platform` attribute for the target. Other parts of
`BuildView` and `AnalysisPhaseRunner` will also need to update for the increased
number of top-level configurations that can be present.

## Detecting conflicts

When a target is being configured in
[`ConfiguredTargetFunction`](https://cs.opensource.google/bazel/bazel/+/master:src/main/java/com/google/devtools/build/lib/skyframe/ConfiguredTargetFunction.java;bpv=1;bpt=1;l=117?q=ConfiguredTargetFunction&ss=bazel&gsn=ConfiguredTargetFunction&gs=kythe%3A%2F%2Fgithub.com%2Fbazelbuild%2Fbazel%3Flang%3Djava%3Fpath%3Dcom.google.devtools.build.lib.skyframe.ConfiguredTargetFunction%233d60156145a0da11321529e3ef670a7934e87833b9c94e47c9938d8642896a0b),
the value of the `platform` attribute can be compared to the target platform,
and if they do not match, and error can be reported.

# Backwards Compatibility

Any targets which do not set the `platform` attribute will continue to use the
target platform from the top-level config, either the one set explicitly at the
command line or the default.

# Potential Future Work

Test execution should also pay attention to the `platform` attribute. In the
future, we can explore if there should be a separate `test_platform` attribute
for tests that need to build for one platform and execute on a different
platform.

