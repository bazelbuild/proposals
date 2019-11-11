---
created: 2019-11-11
last updated: 2019-11-11
status: Draft
reviewers:
  - aragos
  - laurentlb
title: Exposing Target Platform Constraints
authors:
  - katre
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

One option is to expose a method (again, on either `SkylarkRuleContextApi` or
`ToolchainContextApi`) to check whether a given constraint value is present.
Showing only the rule context version:

```py
if ctx.target_platform_has_constraint('@bazel_platforms//os:windows'):
  platform_separator = '\\'
```

This version is simpler for rule authors to use, but still allows rule authors
to inspect constraints used by different rule sets. The implementation can
create a similar method in
[`ConstraintCollection`](https://source.bazel.build/bazel/+/master:src/main/java/com/google/devtools/build/lib/analysis/platform/ConstraintCollection.java?q=ConstraintCollection)
to perform the actual check.

## Alternative 1: A list of all constraint values

The simplest option to enable this feature would be to simply to expose all
[`ConstraintValueInfoApi`](https://source.bazel.build/bazel/+/master:src/main/java/com/google/devtools/build/lib/skylarkbuildapi/platform/ConstraintValueInfoApi.java)
values from the current target platform's `PlatformInfo`. This would be fairly
easy to add as a new attribute on either `SkylarkRuleContextApi' or
`ToolchainContextApi`, depending on which form of rule code is preferred:

```py
# Via the rule context directly:
toolchain = ctx.toolchains['//tools/cpp:toolchain_type']
constraint_values = ctx.target_platform_constraints

# Via the toolchain context:
toolchain = ctx.toolchains['//tools/cpp:toolchain_type']
constraint_values = ctx.toolchains.target_platform_constraints
```

One disadvantage of this option is that rule authors will then need to iterate
this list checking for constraint settings and values they are interested in.
While a utility method can be added to make this superficially similar to
performing the check in `ConstraintCollection`, this will always have more
overhead due to being implemented in Starlark.

## Alternative 2: A method that returns all constraints for a given setting

Another natural way to consider this problem is to request the constraint values
for a given constraint setting:

```py
values = ctx.target_platform_constraints['@bazel_platforms//os']
```

Note that in the near future the [uniqueness of constraint values may be
relaxed](https://github.com/bazelbuild/bazel/issues/8763), allowing for multiple
constraint values for the same setting in a platform. Due to this, the
`target_platform_constraints` lookup should return a set, not a single
`ConstraintValueInfo`.

## Further Options

Instead of using labels to identify constraint settings and values, it would be
useful to use the actual `ConstraintSettingInfo` and `ConstraintValueInfo`
providers. These would need to be determined via an implicit dependency, but
would allow for checking of constraint visibility and guarantee correct
resolution of cross-repository labels:

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

This would ensure that rules can only access constraints that they are allowed
to establish dependencies on, at the cost of requiring more complexity for rule
authors.

# Backward-compatibility

This is a new feature being added to `SkylarkRuleContextApi`, so there are no
backwards compatibility concerns.
