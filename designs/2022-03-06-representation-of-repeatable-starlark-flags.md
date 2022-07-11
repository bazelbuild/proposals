---
created: 2022-03-06
last updated: 2022-03-07
status: To be reviewed
reviewers:
  - @gregestren
title: "Representing repeatable Starlark flags as config.string_list"
authors:
  - @fmeum
---

# Abstract

Repeatable Starlark flags should be represented as `config.string_list` rather than `config.string`.
This would both reduce the amount of special casing required across the codebase and allow any list
of strings as a default value, not just singleton lists.

# Background

In order to fully starlarkify native rules, Starlark build settings have to provide a way to
represent repeatable command-line flags such as the native `--copt`. This is currently possible via
the `allow_multiple` parameter of `config.string`, as seen in the following hypothetical example:

```starlark
# rules_cc/cc/internal/config.bzl
repeatable_string_flag = rule(
    implementation = _impl,
    build_setting = config.string(flag = True, allow_multiple = True)
)

# rules_cc/cc/BUILD.bazel
load("@rules_cc/cc/internal:config.bzl", "repeatable_string_flag")
repeatable_string_flag(
    name = "copt",
    build_setting_default = "", # Not possible to set a list as default value.
)
```

Uses of this flag on the command-line are accumulated, so that the value
of `ctx.build_setting_value` would be `["-O3", "-g"]` with this command-line invocation:

```shell
bazel build //some:target --@rules_cc//cc:copt=-O3 --@rules_cc//cc:copt=-g
```

This approach to repeatable flags has the following drawbacks:

1. Since the flag is represented by `config.string`, its `build_setting_default` takes a string
   rather than a list of string. The effective default is then taken to be the singleton list
   containing that string. This is unexpected behavior and prevents rule authors from specifying
   actual list-valued defaults, including the empty list (see [bazel#13817]).
2. Within Bazel and outside `StarlarkOptionsParser`, a `BuildSetting` instance backing
   a `config.string` created with `allows_multiple = True` has to behave just like one created
   with `config.string_list`, even though its type is still `String` and not `ListType<String>`.
   This requires special handling in a number of places outside the actual option parsing, such as
   in [StarlarkTransition], [StarlarkRuleContext], and [StarlarkOptionParser]. It is not clear
   whether all parts of the codebase handle this correctly (see [bazel#14894] for an example).

# Proposal

Repeatable string flags could instead be represented as `string_list` build settings that only
differ from the default one in the way usages of the flags on the command-line are parsed.

Specifically, a new named parameter `repeatable` can be added to `config.string_list` with the
following behavior:

* The type of the parameter is `boolean`.
* The default value of the parameter is `False` (i.e., the parameter is optional).
* It is an error to set `repeatable` to a value other than `False` if `flag` is not set to `True`.
* If `flag` is `True` and `repeatable` is set to `False`, the `string_list` setting behaves as it does now:

```
--copt=a,b,c        --> ["a", "b", "c"]
--copt=a --copt=b,c --> ["b", "c"]
```

* If `flag` is `True` and `repeatable` is set to `True`, the `string_list` setting behaves as it does
  now with the exception that the value of the setting set on the command line is a list containing
  the unmodified values of each individual occurrence of the flag in the order they appear. In
  particular, the values are *not* split on a comma. This parsing behavior agrees with that of a
  `config.string` flag with `allow_multiple = True`:
  
```
--copt=a,b,c        --> ["a,b,c"]
--copt=a --copt=b,c --> ["a", "b,c"]
```

With this new parameter, the example from the previous section could look as follows:

```starlark
# rules_cc/cc/internal/config.bzl
repeatable_string_flag = rule(
    implementation = _impl,
    build_setting = config.string_list(flag = True, repeatable = True)
)

# rules_cc/cc/BUILD.bazel
load("@rules_cc/cc/internal:config.bzl", "repeatable_string_flag")
repeatable_string_flag(
    name = "copt",
    build_setting_default = [], # Any list can be set as a default here.
)
```

Since a `BuildSetting` instance created in this way has the type `ListType<String>`, any list can be
specified as its default value and no further special handling is required in any part of Bazel
other than the logic that accumulates command-line flag values. This resolves both issues described
in the previous section.

## Additional steps

Since the new parameter makes `config.string_list` a full replacement for `config.string` with
`allow_multiple = True`, the `allow_multiple` parameter can be deprecated.

As the parts of the code listed in the [Background](#background) section could be simplified
substantially if `allow_multiple` were removed, an `--incompatible_config_string_allow_multiple`
Starlark semantics flag defaulting to `true` could be added and used specified in
`enableOnlyWithFlag` for the `allow_multiple` parameter.

## Alternatives considered

Instead of a `boolean` parameter `repeated`, the new parameter of `config.string_list` could be a
`String` argument that allows to specify the name of a "flag passing style". This would allow for
the introduction of flag styles other than comma-separated and repeated and potentially be more
self-documenting, but also complicates the API without a clear current need.

# Backward-compatibility

This proposal adds a new named parameter to an existing Starlark function, which is a
backwards-compatible change.

If the additional step of deprecating `allow_multiple` and introducing an incompatible flag for it
is taken, rulesets will have to migrate to `config.string_list` at some point. The migration is
straightforward: `config.string(flag = True, allow_multiple = True)` has to be replaced with
`config.string_list(flag = True, repeated = True)` and the `build_setting_default` value has
to be set as a list. This change does not impact users of the ruleset in any way.

[bazel#13817]: https://github.com/bazelbuild/bazel/issues/13817

[bazel#14894]: https://github.com/bazelbuild/bazel/issues/14894

[StarlarkOptionParser]: https://github.com/bazelbuild/bazel/blob/0dc078ab83458cc7752293a08c92a6d690f1ee58/src/main/java/com/google/devtools/build/lib/runtime/StarlarkOptionsParser.java#L207

[StarlarkRuleContext]: https://github.com/bazelbuild/bazel/blob/0dc078ab83458cc7752293a08c92a6d690f1ee58/src/main/java/com/google/devtools/build/lib/analysis/starlark/StarlarkRuleContext.java#L632

[StarlarkTransition]: https://github.com/bazelbuild/bazel/blob/0dc078ab83458cc7752293a08c92a6d690f1ee58/src/main/java/com/google/devtools/build/lib/analysis/starlark/StarlarkTransition.java#L329
