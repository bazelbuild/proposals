---
created: 2019-01-14
last updated: 2019-02-06
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
import to be aliased to `delay_load`.) Delayed loading is only available
in a `WORKSPACE` context. It will be an error if a
`delay_load` is set to `True` in a file
not included (directly or indirectly) from the
(top-level) `WOKRSPACE` file.

If a `load` statement with `delay_load` set to `True` is executed,
it binds the corresponding variables not to a value, but to a thunk
that is evaluated on first use (in particular, the thunk is never
executed if the symbol is never used). On that first use, the repository
with the file to be loaded has to be already defined, otherwise
bazel will error out. At the same time,
the external repository that the file was loaded from (if it was taken
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

# Possible implementation

Unfortunately, the evaluation of a `WORKSPACE` file is not that
imperative, as it is evaluated in the Skyframe framework. If other
information is needed, e.g., a file from a different repository,
then it has to be requested and, if not yet available, the current
computation has to return `null` and gets restarted once the value
is fetched. The dependency graph of computations depending on other
values has to be cycle free.

## Extending of `SkyKey`s for fragments of the `WORKSPACE` file

The `SkyKey` for a fragment of the `WORKSPACE` file is currently
given by the path and the index of the chunk; here, the `WORKSPACE`
file is divided into chunks at each block of `load` statements. The
only way, these keys are obtained is by starting with the initial
key, asking for the corresponding value and get the next key as
the `next()` part of the value, and so on, following this chain.

Now, if we would keep this structure of the `SkyKey`, upon evaluating
a delayed `load`, the `compute` function for such a key would have
to ask for that repository to be fetched and wait to be restarted.
Doing this directly, the newly discovered definition of the repository
would never be made known to Skyframe.

To overcome this problem, we propose to extend the `SkyKey` for
the `WORKSPACE` by a "subindex" which is the set of the names of
the newly discovered repository definitions in earlier evaluations.
Then upon a thunk being forced with the repository not yet loaded,
the `compute` function can return a value with the repositories
discovered so far (in particular, the newly discovered definition
of the repository that was loaded lazily). That value will have as
a `next` value the same key, with the subindex extended by the name
of the repository the thunk referred to (and where the definition
was just discovered). When the `compute` function for that extended
key is evaluated, the subindex tells for which repositories the
definition is already available from previous iterations; for those,
the function will ask for the respective files to be fetched, even
if the loading is delayed.

A [proof-of-concept implementation of delayed `load`
statements](https://bazel-review.googlesource.com/c/bazel/+/88830)
based on this extension of the `WorkspaceFileKey`s is available.

## Handling of `native.existing_rules()`

Special care has to be taken in the implementation of the function
`native.existing_rules()`. Under the proposed change of the
`SkyKey` structure, a `WORKSPACE` chunk is evaluated several times
and on later iterations with repository definitions known from
previous iterations. Doing so without changing the implementation
of `native.existing_rules()` can lead to such a call anticipating
definitions happening only later in that chunk; in particular, it
could lead to a different execution path being taken the second
time this Starlark code is evaluated.

Fortunately, the proposed subindex contains the names of all the
repositories which should be considered not known from previous
chunks, but, instead, only be mentioned in `native.existing_rules()`
if freshly defined in the current evaluation. To do so, the list
of anticipated definitions, i.e., the subindex, has to be passed
through the whole Starlark evaluation mechanism. (This has not been
done in the mentioned prototype implementation.)
