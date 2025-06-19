---
created: 2025-06-18
last updated: 2025-06-18
status: To be reviewed
reviewers:
  - fmeum
title: Template for proposals
authors:
  - jacky8hyf
---


# Abstract

`run_shell` actions should respect registered `sh_toolchain()`.

A shell binary, as well as all tools available in `PATH`, should be declared in
the `sh_toolchain()`. Then, all actions created by `ctx.actions.run_shell()`
should use the given shell and `PATH` to resolve commands.

# Background

As of Bazel 8.2.1, `ctx.actions.run_shell` on a Linux machine does not respect
the `sh_toolchain()` registered. In addition, even with
`--incompatible_strict_action_env`, `PATH` is only frozen to a restrictive list
like `/usr/local/bin:/usr/local/sbin:/usr/bin:/usr/sbin:/bin:/sbin:.`. The
actions can still access tools from the host in `/usr/bin`, for example.

This means if rules from a dependent Bazel module uses `ctx.actions.run_shell`,
the user can't control the list of tools to ensure hermeticity. For example,
a target of the `bazel-skylib`'s `copy_file()` rule always uses `cp` from
the host. The user is unable to provide a hermetic one because of this code:

https://github.com/bazelbuild/bazel-skylib/blob/223e4e945801dfbc0bfa31d0900196f5bb54b0fc/rules/private/copy_file_private.bzl#L56

```
    ctx.actions.run_shell(
        command = "cp -f \"$1\" \"$2\"",
    )
```

Same goes for `genrule()`s; the following `genrule()` always uses `cp` from
host:

```
genrule(
    name = "copied",
    srcs = ["src"],
    outs = ["dst"],
    cmd = "cp -fL $< $@",
)
```

This is particularly an issue when the user cannot directly modify the action or
the `genrule()` because they are from a dependent module. The user cannot ensure
hermeticity in their build just because they used `copy_file()` from
`bazel_skylib` without checking its implementation.

# Proposal

## sh_toolchain() should allow specifying a list of tools

This feature request should really go into
https://github.com/bazelbuild/rules_shell, but it is related here as well.

```
sh_toolchain(
    tools = [
        "//label/to:cp",
        "//label/to:ls",
        "//label/to:filegroup_of_other_tools",
    ],
    # ...
)
```

When `tools` is specified, any relevant shell will start with `PATH` evaluated
to directories containing the aforementioned labels to tools. `PATH` should NOT
contain host `PATH`s, e.g. `/usr/bin`.

### Why labels, not paths?

This allows some tools to be built from sources. In the above example,
`//label/to:cp` could be a `native_binary()` prebuilt, or a `cc_binary()` built
from sources.

### Allowlist of host tools

Sometimes, the user may want to use hermetic tools for some, and host tools for
other tools. This is useful when the sources / prebuilts of these host tools
can't yet be imported to the source tree, possibly due to licensing or
infrastructure issues.

To achieve this in the above setup, the user can create a new
`repository_rule()` that relies on
[`repository_ctx.which()`](https://bazel.build/rules/lib/builtins/repository_ctx#which)
and
[`repository_ctx.symlink()`](https://bazel.build/rules/lib/builtins/repository_ctx#symlink).
Then, for example:

```
# path/to/BUILD.bazel
sh_toolchain(
    name = "sh_toolchain",
    tools = [
        "//label/to:cp",
        "@my_host_tools//:rsync", # symlink to /usr/bin/rsync
    ],
)
```

Alternatively, to make user's life easier, `sh_toolchain()` could add a
`host_tools` attribute.

## ctx.actions.run_shell() should respect `sh_toolchain()`

When a flag, `--incompatible_shell_actions_use_sh_toolchain`, is set, all
`run_shell()` actions in the current dependency graph of requested targets must:

-   Use the shell binary declared in the resolved `sh_toolchain()`, if any
-   Before running the shell command, `PATH` must be set to a contained list
    of directories containing `sh_toolchain(tools=)` only.

Example:

```
ctx.actions.run_shell(
    # ...
    command = """
        echo $PATH
        # ...
    """
)
```

Under `--incompatible_shell_actions_use_sh_toolchain`, with the aforementioned
`sh_toolchain()` registered, the following might be printed in the terminal:

```
/home/$USER/.cache/bazel/_bazel_$USER/30b14dec2b1e1f8876279f4a1457d4a6/execroot/_main/path/to/sh_toolchain/path
```

### No changes to run_shell() or rule() API

Ideally, there should not be any changes to the `run_shell()` or `rule()` API,
or any API, to achieve this, other than the command line flag
`--incompatible_shell_actions_use_sh_toolchain`. This is to ensure that users
of old Bazel module versions benefit from this, and we don't need to update the
entire universe of Bazel modules to roll this feature out.

For example, a user of `bazel-skylib`'s `copy_file()` only needs to
add `--incompatible_shell_actions_use_sh_toolchain` to ensure that it uses
the `//label/to:cp` the user provides. They don't need to wait until
`bazel-skylib` is updated to adopt this feature.

# Backward-compatibility

If `sh_toolchain(tools=)` is not specified, it should fallback to its original
behavior of not touching `PATH` for any relevant shell instantiated. This
ensures backwards compatibility.

If `--incompatible_shell_actions_use_sh_toolchain`, `ctx.actions.run_shell()`
and `genrule()` should keep its original value of `PATH`, controlled by
`--incompatible_strict_action_env`.

# Security considerations

Today (as of Bazel 8.2.1), an action can still access host
tools if it explicitly requests so:

```
    ctx.actions.run_shell(
        command = "/usr/bin/cp -f \"$1\" \"$2\"",
    )
```

```
genrule(
    cmd = "/usr/bin/cp -fL $< $@",
)
```

The `--incompatible_shell_actions_use_sh_toolchain` flag is not intended to
guard intentional breakages of hermeticity when the user or Bazel module
maintainer does the above. In other words, `/usr/bin/cp` would still resolve
to the host `cp`, just as before.

