---
created: 2018-11-07
last updated: 2018-11-07
status: To be reviewed
reviewers:
  - dslomov
  - dannark
title: Recursive Workspaces
authors:
  - aehlig
---

# Recursive workspaces

As the default mode of operation of bazel is to build everything
completely from source, bazel needs to be aware of all the code that
transitively affects the project under consideration. As long as
all code is organized in one big mono-repository, this is easy. However,
once one deals with code organised in several repositories, the
problem arises of keeping track of repositories that are not used directly
by the project, but only indirectly as transitive dependencies.

## Existing approaches

### Ignore the problem

The historic approach of bazel with respect to transitive dependencies
was to ingore the problem and expect the user to (manually) add all transitive
dependencies to the `WORKSPACE` file. This approach is followed by quite some
projects and therefore should continued to be supported, i.e., any form of
recursive workspace handling should be sufficiently opt-in to not make the
workflow any harder for projects that already manage all their transitive
dependencies explicitly in their `WORKSPACE` file (maybe with the help of
some specialized tools).

### The first design of recursive `workspaces`

The [first design for recursive workspaces](https://bazel.build/designs/2016/09/19/recursive-ws-parsing.html)
approaches the problem of transitive dependencies in the following way.

- A definition of a repository in a `WORKSPACE` file takes precedence over
  definitions of the same repository in the `WORKSPACE` files included as
  transitive dependencies of said `WORKSPACE` file.

- For definitions that are not in a direct line, it is considered an error
  if they define an external repository with the same name, but with a different
  defintion. However, those errors are only reported, once discovered; so it
  may well be that for two targets, building each individually on a clean
  workspace works, but building both together will error out due to the
  definition conflict.

The laziness in error checking was specified to avoid that the whole `WORKPACE`
has to be evaluated recursively (which includes fetching all transitive
dependencies, even if not needed for a specific target) before being able to
start the build.

This proposal was never implemented.

### The `dependencies()` pattern

Another approach for indirect dependencies that emerged over time is that
repositories that intend to be used as third-party dependencies export a
Starlark function that adds their dependencies, if not added already. So,
the top-level `WORKSPACE` file would have entries like the following.

```
load("@bazel_tools//tools/build_defs/repo:http.bzl", "http_archive")

http_archive(
  name = "com_example_foo",
  urls = ["https://example.com/foo/foo-1.2.3.tar.gz"],
  strip_prefix = "foo-1.2.3",
  sha256 = "0a6717e765818538f153ac27ca6cf97c4290cb25dd374017e1dc2bd0d6f6bf5c",
)

load("@com_example_foo//:deps.bzl", "foo_deps")
foo_deps()
```

The exported function `foo_deps()` would add the repositories `com_example_foo`
depends on, if not already present. To do so, it checks if the repository
name is already in `native.existing_rules()` and if this is not the case,
adds the respective dependency.

```
load("@bazel_tools//tools/build_defs/repo:http.bzl", "http_archive")

def foo_deps():
  if "com_example_bar" not in native.existing_rules():
    http_archive(name="com_example_bar", ...)
  ...

```

This approach works well for one level of dependencies. However, the
`*_depenencies()` function need to be aware of all their transitive
dependencies; an exported Skylark function can add more repositories,
but cannot load functions from those repositories.

Moreover, using the `dependencies()` pattern implies that all repositories
exporting `*_dependencies()` are loaded unconditionally.

## Requirements

We consider manually keeping a list of all transitive dependencies a legitimate
usse case; this use case also allows most control of which versions of the
transitive dependencies are used. As a consequence, recursive workspaces must
be added in a truely opt-in fasion, not causing additional fetches to existing
projects.

Bazel values reproducibility of builds. In particular, the result of a target
(including whether an error in the workspace are reported or not)
should only depend on the repository itself, not on any previously built
targets (or the order in which they were built). This implies that, when using
an external repository, all possible conflicts or definitions taking priority
have to be explored first, even if this implies fetching a large numebr of
repositories that might bring an alternative definition in their indirect
dependencies.

Finally, we are aiming for a general mechanism of handling transitive
dependencies, without any limits on the length of dependency chains.

## Suggested implementation

### Recursion only on `bazel sync`

To reconceile the requirement of the outcome being independent of previous
builds, and that of avoiding unnecessary fetches, we propose using the
concept of [resolved
files](https://blog.bazel.build/2018/09/28/first-class-resolved-file.html).
So, for normal builds, the full list of all external repositories involved
will already be cached in a local Starlark file, without the need of fetching
anything not needed. To update that list, an new option `--recursive` to the
`bazel sync` command will be used. This option will tell bazel to also inspect
the `WORKSPACE` files of external repositories. Typically, bazel would be
instructed to record the result of this recursive traversal in a resolved file;
in this way, the expensive operation of fetchting and inspecting all transitive
repositories need to be done only once per dependency update and can be shared
between different developers of the same project.

In this way, any form of recursive workspaces is also completely opt-in:
recursive sync is controlled by a newly added option, and telling bazel to
write or read a resolved file also has to be enabled explictly by appropriate
flags.

### Traversal order and precedence: depth-first

For direct dependencies in a `WORKSPACE` file, they may be used, immediately
after being introduced. So the following is perfectly legitimate.

```
load("@bazel_tools//tools/build_defs/repo:http.bzl", "http_archive")

http_archive(
  name = "com_example_foo",
  urls = ["https://example.com/foo/foo-1.2.3.tar.gz"],
  strip_prefix = "foo-1.2.3",
  sha256 = "0a6717e765818538f153ac27ca6cf97c4290cb25dd374017e1dc2bd0d6f6bf5c",
)

load("@com_example_foo//:foo.bzl", "foo")
foo()
```

Now, the idea of recursive workspaces is that an external repository, like
`@com_example_foo` by just be added without having to think about its
dependencies. The file `@com_example_foo//:foo.bzl`, on the other hand,
as it comes from `@com_example_foo`, may freely load functions defined
in external repositories of the workspace of `@com_example_foo`.

To best accomodate these dependencies, i.e., to still allow the call
to `foo()` be evaluated in a recursive sync, we propose a depth-first
traversal of the transitive dependencies. So simplify the semantics we
also propose the depth-first traversal as precedence order: a repository
found earlier in the depth-first traversal takes precedence over a later
repository with the same name; in fact, the latter will not even be
read (and hence its dependencies are ignored).

It should be noted that here the proposal differs from the earlier one
that argued that repositories defined in the top-level `WORKSPACE` should
take precedence over transitive dependencies. Our argument is that it
is more desirable to have the meaning of `foo()`, and hence the set of
repositories it might introduce to the global name space, be independent
of declarations made later in the top-level `WORKSPACE` file. Moreover,
this requirement is even necessary, if we want to stay with the current
semantics of a `WORKSPACE` file that cuts it into chunks at each load
statement and assumes that each chunk can be evaluated knowing only
the previous chunks.

### Order to be understood logically

Bazel currently allows contructions like the following in the `WORKSPACE`.

```
load("@bazel_tools//tools/build_defs/repo:http.bzl", "http_archive")

http_archive(
  name = "com_example_foo",
  urls = ["https://example.com/foo/foo.tar.gz"],
  sha256 = "0a6717e765818538f153ac27ca6cf97c4290cb25dd374017e1dc2bd0d6f6bf5c",
  build_file="@com_example_bar//:foo.BUILD",
)

http_archive(
 name = "com_example_bar",
 urls = ["https://example.com/bar/bar.tar.gz"]
 sha256 = "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855",
)
```

Here, the repository `@com_example_foo` cannot be fetched before
`@com_example_bar` is available, as the fetch itself requires a file from
`@com_example_bar`. Therefore, despite the location in the `WORKSPACE` file, we
consider `@com_example_bar` logically before `@com_example_foo`. In particular,
if `@com_example_bar` pulls in, maybe indirectly, a repository called
`@com_example_foo` through its `WORKSPACE`, the defintion of `@com_example_foo`
in the top-level `WORKSPACE` file will be ingored.

This topological sorting also applies over indirect edges. So in the following
snippet, the repository `@com_example_baz` will be fetched first and might
override the definitions of `@com_example_foo` and `@com_example_bar` in the
top-level `WORKSPACE` file.

```
load("@bazel_tools//tools/build_defs/repo:http.bzl", "http_archive")

http_archive(
  name = "com_example_foo",
  urls = ["https://example.com/foo/foo.tar.gz"],
  sha256 = "0a6717e765818538f153ac27ca6cf97c4290cb25dd374017e1dc2bd0d6f6bf5c",
  build_file="@com_example_bar//:foo.BUILD",
)

http_archive(
 name = "com_example_bar",
 urls = ["https://example.com/bar/bar.tar.gz"]
 sha256 = "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855",
  build_file="@com_example_baz//:bar.BUILD",
)

http_archive(
 name = "com_example_baz",
 urls = ["https://example.com/bar/baz.tar.gz"]
 sha256 = "01ba4719c80b6fe911b091a7c05124b64eeece964e09c058ef8f9805daca546b",
)
```

As circular dependencies are not allowed, projects can always avoid this
confusingly-looking behaviour by presenting the entries in their `WORKSPACE`
file in topological order.

### Recursive `sync` is necessarily sequential

Another consequence of having a  precedence order on definitions of external
repositories is that the fetch of a repository may only be started once all
places that might find defintions take precedence have been fully discovered
(as no observable behaviour, including network access and errors may come
from a defintion that is overridden by another one). For depth-first traversal,
this implies strictly sequential fetches. This is costly (in terms of wall-clock
time), but probably acceptable, given that in the intended
use case, recursive syncs to update the resolved information, are rare.
