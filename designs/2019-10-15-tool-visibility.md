---
created: 2019-10-15
last updated: 2018-10-23
status: To be reviewed
reviewers:
  - laurentlb
title: Changing visibility for implicit arguments
authors:
  - aehlig
---


# Abstract

We propose that for implicit arguments of a rule, visibility is
checked with respect to the rule definition rather than the rule
usage.


# Background

Build rules are parameterized by attributes. Some attributes (those where
the name starts with `_`) cannot be changed on using the rule. Those [private
attributes](https://docs.bazel.build/versions/master/skylark/rules.html#private-attributes-and-implicit-dependencies)
will always have their default value as specified in the rule
definition. Typically, they are used to provide tools needed for
the rule, e.g., the [dependency of the `java_grpc_library` on
`protoc`](https://github.com/bazelbuild/bazel/blob/ebb77e41973cb6b9f963159f4ef4a17e524ce062/third_party/grpc/build_defs.bzl#L63),
but also for tools specific for that rule and coming from the
same package, like the [`make_rpm` script in the `pkg_rpm`
rule](https://github.com/bazelbuild/rules_pkg/blob/b8d6ea0a5465973ce0970f6e063dfebea473732c/pkg/rpm.bzl#L163).

For every target, it is checked that its dependencies
are legitimate. Targets can specify in the [common
attribute](https://docs.bazel.build/versions/master/be/common-definitions.html#common-attributes)
`visibility` which other targets can depend on them. For each
target, all label attributes of the defining rule must be visible
by that target.

Currently, all attributes are treated equally. In particular,
private attributes still must be visible from usage of that rule.
In practice that means that everything referred to by a private
attribute must be at least as fusible as the rule itself, which,
for lack of visibility rules for rules, is `//visibility:public`.
While public visibility is intended anyway for generic tools
like `protoc`, it can be a problem for tools or scripts specific
for a rule; in those cases the command-line arguments options,
etc, should be considered an internal interface between the
tool and the rule. An example of such an internal tool is the
[`codedesigntool`](https://github.com/bazelbuild/rules_apple/blob/7f8a25a57ab9a4e406025eaa2c4394a20f793f47/tools/codesigningtool/BUILD#L7-L10)
that [belongs to the
`_COMMON_PRIVATE_TOOL_ATTRS`](https://github.com/bazelbuild/rules_apple/blob/7f8a25a57ab9a4e406025eaa2c4394a20f793f47/apple/internal/rule_factory.bzl#L105)
in [`rules_apple`](https://github.com/bazelbuild/rules_apple).
In fact, that tool motivated the [first
request](https://github.com/bazelbuild/bazel/issues/7377) for the
feature addressed in this proposal.

## Example

The general scheme of [depending on a
tool](https://docs.bazel.build/versions/1.0.0/skylark/rules.html#private-attributes-and-implicit-dependencies)
is to have a rule definition in  `//foo/rule:rule.bzl` as follows.

```
def _impl(ctx):
  ...

foo_binary = rule(
  implementation = _impl,
  attrs = {
    "srcs" : ...,
    ...
    "_tool" : attr.label(default=Label("//foo/tool:foobuilder"),
                         executable=True, cfg="host"),
  }
```

Now, if that rule is used, say in `//bar`, with a `BUILD` file like the
following.

```
load("//foo/rule:rule.bzl", "foo_binary")

foo_binary(
  name = "myapp",
  srcs = ...
)
```

Then `//bar:myapp` implicitly depends on `//foo/tool:foobuilder`, so
`//foo/tool:foobuilder` must be visible by `//bar:myapp`. This visibility
allows targets in `//bar` to also use `foobuilder` directly, even if it
is intended to be an implementation detail of `foo_binary`. This unrestricted
use makes it hard for the `//foo` package to evolve the command-line interface
of `foobuilder`.


# Proposal

We propose to change the visibility requirements for private
attributes. For private attributes, instead of checking that the
attribute is visible from the target using the rule, we propose to
verify that the private attribute is visible from the definition of
the rule (i.e., the label corresponding to the file containing the
rule definition). In this way, it will be possible to restrict the
visibility of a tool to the package containing the rule definition,
while still allowing the rule to be used everywhere.


# Backward-compatibility

The change is not backwards compatible; it might be that a rule uses
a tool that is not visible to definition of the rule, but to all
uses of the rule. Therefore, this change will be guarded by the new
flag `--incompatible_visibility_private_attributes_at_definition`.

Also, changing the way the visibility is verified might block potential
use cases of the current way.

## Private attribute as a replacement for rule visibility

Currently, bazel has no concept of visibility that can be used to restrict
usage of a rule or `bzl` file. It is possible to abuse the visibility checks
for private attributes at the place a rule is used to restrict usage of that
rule, simply by adding a private dummy attribute and restricting its visibility
appropriately. This would no longer work with the proposed change.

