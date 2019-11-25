---
created: 2019-11-11
last updated: 2019-11-13
status: In Review
reviewers:
  - aragos
  - laurentlb
title: Exposing Target Platform Constraints
authors:
  - katre
discussion thread: https://groups.google.com/d/msg/bazel-dev/WcysNkM7OLg/W2JCqBZKAgAJ
---


# Abstract

Rule authors occasionally need information about the target platform in a rule
implementation. This can be achieved currently using either custom toolchains or
by using implicit attributes with a `select()` call. This proposal aims to allow
rule authors the information they need in a simpler manner.

# Background

Every configured target has a target platform that the outputs are compatible
with. This target platform may be set (either by default or by a command-line
flag) when the build begins, or may be changed as a result of a configuration
transition.

A current workaround to determine the target platform is to check
[`ctx.configuration.host_path_separator`](https://github.com/bazelbuild/bazel/issues/9209),
or to add [an implicit attribute that selects on a relevant platform
constraint](https://github.com/bazelbuild/bazel/issues/9209#issuecomment-539239447).
However, both of these are inexact and add additional overhead for rule authors.

# Current Status

The target platform information, as a [PlatformInfo
provider](https://source.bazel.build/bazel/+/master:src/main/java/com/google/devtools/build/lib/analysis/platform/PlatformInfo.java),
is currently available during rule implementation (and can be accessed by native
rules), via the
[`ToolchainContext`](https://source.bazel.build/bazel/+/master:src/main/java/com/google/devtools/build/lib/analysis/ToolchainContext.java).
However, the
[`ToolchainContextApi`](https://source.bazel.build/bazel/+/master:src/main/java/com/google/devtools/build/lib/skylarkbuildapi/ToolchainContextApi.java)
itself only exposes toolchain info providers via a map-like API, which allows
code such as:

```py
toolchain = ctx.toolchains['//tools/cpp:toolchain_type']
```

# Proposal: A check method that takes a constraint value label

One option is to expose a method (on `SkylarkRuleContextApi`) to check whether a
given constraint value is present. The value passed would need to be a valid
`ConstraintValueInfo` provider, obtained by an implicit depdency on the
constraint value definition that is to be checked. This ensures both that the
constraint value is valid, and that the rule author has the correct visibility
available to use the constraint value.

```py
def _impl(ctx):
  windows_constraint = ctx.attr._windows_constraint[platform_common.ConstraintSettingInfo]
  if ctx.target_platform_has_constraint(windows_constraint):
    platform_separator = '\\'


my_rule = rule(
  implementation = _impl,
  attrs = {
    ...
    '_windows_constraint': attr.label(default = '@bazel_platforms//os:windows'),
  },
)
```

This version requires an additional implicit attribute from rule authors to use,
but the overhead is low and the correctness guarantees are worthwhile.

The implementation can check the type and then call a similar method in
[`ConstraintCollection`](https://source.bazel.build/bazel/+/master:src/main/java/com/google/devtools/build/lib/analysis/platform/ConstraintCollection.java?q=ConstraintCollection)
to perform the actual check.

# Backward-compatibility

This is a new feature being added to `SkylarkRuleContextApi`, so there are no
backwards compatibility concerns.
