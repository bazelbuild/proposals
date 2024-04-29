---
created: 2024-04-26
last updated: 2024-04-29
status: under review
reviewers:
  - gregestren
title: Selectively Enabling Execution Platforms
authors:
  - katre
discussion thread: https://github.com/bazelbuild/bazel/discussions/22170
---

# Overview

Currently, execution platforms are [registered in the `WORKSPACE` or
`MODULE.bazel`
files](https://bazel.build/rules/lib/globals/module#register_execution_platforms),
or added via the [`--extra_execution_platforms`
flag](https://bazel.build/reference/command-line-reference#flag--extra_execution_platforms).
At that point they are [available for toolchain
resolution](https://bazel.build/extending/toolchains#toolchain-resolution) and
to be used for rules and actions.

In some cases, this is too restrictive: there are projects that are currently
using transitions to change the value of `--extra_execution_platforms` and add
new execution platforms mid-build. Also, because of the semantics of the flag,
[repeated use of `--extra_execution_platforms` overrides previous values, instead
of accumulating them](https://cs.opensource.google/bazel/bazel/+/c602cec7887470db3e8ed69600f5bd2f38e160d5).

The similar ways of registering toolchains has a way around this: the
`toolchain` rule has an attribute called
[`target_settings`](https://bazel.build/reference/be/platforms-and-toolchains#toolchain.target_settings),
which is a list of targets that provide
[`ConfigMatchingProvider`](https://cs.opensource.google/bazel/bazel/+/master:src/main/java/com/google/devtools/build/lib/analysis/config/ConfigMatchingProvider.java)
(such as
[`config_setting`](https://bazel.build/reference/be/general#config_setting)).
When a toolchain specifies `target_settings`, the toolchain is only available if
all of the settings are true for the current configuration.

# Proposal

To allow execution platforms to control whether they are active, a new attribute
will be added, named `enabled_settings`, which will take targets that provide
`ConfigMatchingProvider`. When the list of execution platforms is evaluated
suring toolchain resolution, each execution platform will check the enabled
settings and any that don't match the configuration will be skipped (and
reported if toolchain resolution debugging is active).

## Example of platform settings

```
config_setting(
    name = "extra_logging_enabled",
    flag_values = { "//flag:extra_logging": "True" },
)

CONSTRAINT_VALUES = [ ... ]

platform(
    name = "remote_extra_logging",
    constraint_values = CONSTRAINT_VALUES,
    exec_properties = ...,
    enabled_settings = [
        ":extra_logging_enabled",
    ],
)

platform(
    name = "remote",
    constraint_values = CONSTRAINT_VALUES,
    exec_properties = ...,
)
```

If both of these platforms are available (either via
`register_execution_platforms` or `--extra_toolchains`), and the default for the
`--//flag:extra_logging` flag is `False`, then the `remote` platform will be
used because the `enabled_settings` will not match. Once the flag is set, the
`enabled_settings` will match, and the `remote_extra_logging` platform will be available (and is
registered before the other).

In this case, to be very specific, an additional `config_setting` could be
defined as the opposite of `extra_logging_enabled` and used to guard `remote` to
ensure that only one of the two can ever be enabled at the same time.

# Implementation

The implementation will mirror the implementation for
`toolchain.target_settings`, by [accumulating the `ConfigMatchingProvider`
instances](https://cs.opensource.google/bazel/bazel/+/master:src/main/java/com/google/devtools/build/lib/rules/platform/Toolchain.java;drc=114c0c641d128429df32999012f7a1207c3bee02;l=55)
and storing them in the [`PlatformInfo`] provider. Then they will be checked
[during toolchain
resolution](https://cs.opensource.google/bazel/bazel/+/master:src/main/java/com/google/devtools/build/lib/skyframe/toolchains/SingleToolchainResolutionFunction.java;drc=114c0c641d128429df32999012f7a1207c3bee02;l=175)
to produce the list of usable execution platforms.
