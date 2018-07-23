---
created: 2018-07-23
last updated: 2018-07-23
status: To be reviewed
reviewers:
  - gregce
title: Toolchain Modes
authors:
  - katre
---

# Background

The current implementation of [toolchain
resolution](https://docs.bazel.build/versions/master/toolchains.html#toolchain-resolution)
is very simplistic: it assumes that there should only ever be one toolchain per
target platform. However, that is not true: there are several reasons why there
might be multiple valid toolchains for a given target platform:
* Generating different types of output: i.e. optimized vs. debug outputs
* Various types of sanitizer enabled: i.e. asan vs tsan vs race detection
* Different underlying compilers: i.e. gcc vs clang
* Targetting different ABI or SDK levels: i.e. jdk8 vs jdk10

Bazel currently allows switching compilation mode using the
[`--compilation_mode`](https://docs.bazel.build/versions/master/user-manual.html#flag--compilation_mode)
flag, but this is only one axis of what toolchain authors need. The cc rules
allow a great deal of customization of the cc toolchain using a combination of
the `--compiler` and `--cpu` flags, and C++ features, but this is not applicable
to other types of toolchain.

Rule authors need the ability to do the following:
* Distinguish between multiple toolchains that are valid for the same target
  platform
* Allow users to specify which toolchain mode to use

# Proposal

Since
[`constraint_value`](https://docs.bazel.build/versions/master/be/platform.html#constraint_value)
labels are used currently to define the requirements of a
[`toolchain`](https://docs.bazel.build/versions/master/be/platform.html#toolchain)'s
execution and target environments, it makes sense to also use constraints to
define the different modes available.

This approach leads to having a separate definition for each toolchain in each
mode. If a rule author wishes to use the same underlying definition for each
mode, it would be possible to use macros to duplicate the actual underlying
targets in each mode, but that is outside the scope of this design.

## Specifying the Desired Toolchain Modes

The simplest and most basic way to specify toolchain modes will be via a new
flag, `--toolchain_modes`, which takes the comma-separated labels of
`constraint_value` targets, and can be repeated.

However, there is also a need to map existing flags (and possibly new flags) to
constraint settings that can be used for toolchain modes as well:

| Legacy (Native) Flag        | Constraint Setting                         |
|-----------------------------|--------------------------------------------|
| `--compilation_mode` (`-c`) | `@bazel_tools//platforms:compilation_mode` |
| `--compiler`                | `@bazel_tools//platforms:compiler`         |

TODO: Identify other legacy native flags that need to be handled.

As part of [Skylark Build
Configuration](https://docs.google.com/document/d/1vc8v-kXjvgZOdQdnxPTaV0rrLxtP2XwnD2tAZlYJOqw),
we need a mechacnism to allow rule authors to declare a mapping between new
Skylark flags and toolchain modes, to allow easier ways to set modes for users.

## Distinguishing Distinct Toolchains

In order to distinguish between different toolchains for the same target
constraints, a new attribute will be added to the
[`toolchain`](https://docs.bazel.build/versions/master/be/platform.html#toolchain)
rule, named `mode_compatible_with`. This set of constraints will be checked
against the mode constraints specified by the user.

[Toolchain
resolution](https://docs.bazel.build/versions/master/toolchains.html#toolchain-resolution)
is updated to compare the user-supplied set of mode constraints with the
toolchain's `mode_compatible_with` set.
1. This serves to remove ineligible toolchains, similar to the use of
   execution platform constraints
2. The first remaining toolchain is used, as happens currently

This has the advantage of clearly separating the constraints that define the
mode from the constraints that define the target environment.

Example:
```
constraint_setting(name = 'compiler')
constraint_value(
  name = 'clang',
  constraint_setting = ':compiler',
)
constraint_value(
  name = 'gcc',
  constraint_setting = ':compiler',
)
toolchain(
  name = 'linux_clang_toolchain',
  toolchain_type = '//path/to:my_toolchain_type',
  exec_compatible_with = [
    '@bazel_tools//platforms:linux',
    '@bazel_tools//platforms:x86_64'],
  mode_compatible_with = [
    ':clang'
  ],
  target_compatible_with = [
    '@bazel_tools//platforms:linux',
    '@bazel_tools//platforms:x86_64'],
  toolchain = ':linux_clang_toolchain_impl',
)
toolchain(
  name = 'linux_gcc_toolchain',
  toolchain_type = '//path/to:my_toolchain_type',
  exec_compatible_with = [
    '@bazel_tools//platforms:linux',
    '@bazel_tools//platforms:x86_64'],
  mode_compatible_with = [
    ':gcc'
  ],
  target_compatible_with = [
    '@bazel_tools//platforms:linux',
    '@bazel_tools//platforms:x86_64'],
  toolchain = ':linux_gcc_toolchain_impl',
)
```

(In practice, macros can be used to simplify redundant toolchain definitions.)

### Alternate Approach: Dynamically add constraints to the target platform

Instead of using a new attribute of the `toolchain` rule, the mode constraints
could simply be added to the existing `target_compatible_with` attribute, and
the toolchain resolution process extended to consider both constraints from the
actual target platform, as well as mode constraints supplied by the user.

This simplifies the toolchain definition, at the expense of making it more
difficult to tell the intended meanings of the various toolchains:
```
# Same constraints as above.
toolchain(
  name = 'linux_clang_toolchain',
  toolchain_type = '//path/to:my_toolchain_type',
  exec_compatible_with = [
    '@bazel_tools//platforms:linux',
    '@bazel_tools//platforms:x86_64'],
  target_compatible_with = [
    ':clang'
    '@bazel_tools//platforms:linux',
    '@bazel_tools//platforms:x86_64'],
  toolchain = ':linux_clang_toolchain_impl',
)
toolchain(
  name = 'linux_gcc_toolchain',
  toolchain_type = '//path/to:my_toolchain_type',
  exec_compatible_with = [
    '@bazel_tools//platforms:linux',
    '@bazel_tools//platforms:x86_64'],
  target_compatible_with = [
    ':gcc'
    '@bazel_tools//platforms:linux',
    '@bazel_tools//platforms:x86_64'],
  toolchain = ':linux_gcc_toolchain_impl',
)
```

