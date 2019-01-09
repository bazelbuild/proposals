---
created: 2019-01-14
last updated: 2019-01-14
status: draft
title: Delaying of `load` statements
authors:
  - aehlig
---

# `load` statements in Starlark

In Starlark, `load` statements are used to import values
defined in other files into the current context. They do so by
introducing a new symbol and the loading is performed immediately
upon executing the `load` statement. There is a push for `BUILD`
files to [have `load` statements only at the beginning of a
file](https://github.com/bazelbuild/bazel/issues/5815) to make it
easier to find out where a symbol is defined.

## Splitting of the top-level `WORKSPACE` file

For `BUILD` files, as well as files transitively loaded from `BUILD`
files, eagerly loading all needed symbols at the beginning of the
file works fine. All files from which to load symbols are already
defined before the first `BUILD` file is read.

For the `WORKSPACE`, however, the situation is different. New
external repositories are still being defined here, thus providing
new locations where files could be loaded from. To deal with this
special situation, the top-level `WORKSPACE` file is split at each
block of top-level `load` statements. Each of the resulting chunks
is interpreted separately with its own environment. This splitting,
however, only happens in the `WORKSPACE` file and on top level,
thus allowing for definitions directly in the `WORKSPACE` file that
cannot be moved to a macro.

This non-composible behavior is most obvious
when dependencies are added using the [`*_deps`
pattern](2018-11-07-design-recursive-workspaces.md#the-dependencies-pattern).
There, each repository exports a Starlark function `deps` adding
its dependencies. These `deps` functions can be loaded and called
from a `WORKSPACE` file adding these repositories as dependencies.
However, these `deps` function cannot load (and hence not call)
the `deps` functions provided by the dependencies they add.

## Suggested changes

To support, in a `WORKSPACE` context, loading files from repositories
that still have to be defined, we suggest adding a new boolean
parameter `delay_load` to the `load` statement, defaulting to
`False`. (As the value cannot be a string, there is no overlap
with with aliased imports; but we can still decide to disallow an
import to be aliased to `delay_load`.) It will be an error if a
value different from `False` is used if the file containing the
`load` statement is not included (directly or indirectly) from the
(top-level) `WOKRSPACE` file.

If a `load` statement with `delay_load` set to `True` is executed,
it binds the corresponding variables not to a value, but to a thunk
that is evaluated on first use. On that first use, the repository
with the file to be loaded has to be already defined, otherwise
bazel will error out. At the same time, the definition of the
external repository the file load from belongs to (if was taken
from an external repository) will be made immutable; any further
attempts to redefine that external repository will be ignored.

### Use case: recursive `*_deps` pattern

With delayed loading added, a recursive `*_deps` pattern can be
coded directly in Starlark, providing an alternative to [recursive
workspaces](2018-11-07-design-recursive-workspaces.md). For example,
the `deps` function of a repository `@foo` depending on a single
repository `@bar` could be defined in a file `@foo//:deps.bzl` that
looks as follows.

```
load("@bazel_tools//tools/build_defs/repo:http.bzl", "http_archive")
load("@bar//:deps.bzl", "bar_deps", delay_load=True)

def foo_deps()
  if "bar" not in native.existing_rules():
    http_archive(name="bar", ...)
    bar_deps()
```

Note that we can even keep the requirement that all `load`
statements appear in the file before any other statements. Moreover,
the recursive `*_deps` pattern is not tied to a fixed traversal
strategy; `*_deps` functions may choose to first define all or some
repositories before calling their `deps` functions.
