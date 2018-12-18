---
created: 2018-11-07
last updated: 2018-11-19
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
after being introduced. So the following is perfectly legitimate, assuming
that the function `foo()` introduces a repository `@com_example_indirect`.

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
load("@com_example_indirect//:rules.bzl", "bar_repo")
bar_repo(name = "bar", ...)
```

Now, the the promise that recursive workspaces are trying to provide
is that an external repository (like `@com_example_foo`) can just be added
and used without having to think about its dependencies.

Now assume that `@com_example_foo//:foo.bzl` originally looked as follows
(which would work in the non-recursive case).

```
load("@bazel_tools//tools/build_defs/repo:http.bzl", "http_archive")

def foo():
  http_archive(
    name = "com_example_indirect",
    urls = ["https://example.com/1.1/indirect.tar.gz"],
    sha256 = "6a6abb37d3dec18048ea33e176da7552ac45b96ac9eef4098dfdd282d23ec177",
  )
```

But the maintainers of `@com_example_foo` decided that, instead of hardcoding
versions and hashes, it is better to take them indirectly form `@bar` which they
added to their `WORKSPACE` file. Hence `@com_example_foo//:foo.bzl` now looks
as follows.

```
load("@bazel_tools//tools/build_defs/repo:http.bzl", "http_archive")

load("@bar//:goodsources.bzl", "goodsources")

def foo():
  (urls, sha256) = goodsources["indirect"]
  http_archive(
    name = "com_example_indirect",
    urls = urls,
    sha256 = sha256,
  )
```

The common expectation of recursive worksapces is that `foo()` can still be
evaluated in the top-level `WORKSPACE` file as the recursion should take care of
of the dependencies that `@com_example_foo` needs in the definition of `foo`.
This, however, is only possible, if we explore the dependencies of
`@com_example_foo` before we try to evaluate the
`load("@com_example_foo//:foo.bzl", "foo")` statement.

This example also shows that the original design of recursive workspaces does
not work. It would require that the definition of `@bar` in the top-level
`WORKSPACE` file take precedence. That defintion, however, depends on knowledge
of `@bar` (as `bar_repo` is loaded from `@com_example_indirect`, where the URL
is taken from `@bar//:goodsources.bzl`), a circular definition that cannot be
resolved.

To ensure that the call to `foo()` can still be be evaluated in a recursive
sync, we propose a depth-first traversal of the transitive dependencies.
To simplify the semantics, we
also propose the depth-first traversal as precedence order: a repository
found earlier in the depth-first traversal takes precedence over a later
repository with the same name; in fact, the latter will not even be
read (and hence the dependencies in its `WORKSPACE` file ignored).

As bazel eventually generates a global namespace, repositories are distinguished
by name (and only name). That is, if, while exploring a `WORKSPACE` file of a
repository, we find a repository with a name not yet in the global name space,
that repository is loaded and its `WORKSPACE` file read recursively. If, on the
other hand, a repository with this name exists already, that entry is ignored
and the next entry is read. This ignoring of an entry is done without any
form of warning, as we consider it normal that two direct dependencies have a
shared dependency.

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

We do not make any promises about which topological order bazel chooses apart
from it being reproducible. One possible implementation is to examine the first
repository in the `WORKSPACE` file and go through its arguments one by one in
a deterministic order; if a reference to a not yet fetched repository is found,
the fetch is aborted, the not-yet fetched repository fetched recursively, and
the original fetch restarted. As opposed to normal Skyframe operations, the
non-parallel inspection of the arguments is important here to give a
well-defined order. In the following example, we do not promise whether `@a`
or `@b` is explored first, but we do promise that the order will always be
the same, indepenently of which of `@a` or `@b` fetches faster.

```
load("@bazel_tools//tools/build_defs/repo:http.bzl", "http_archive")

http_archive(
  name = "com_example_foo",
  urls = ["https://example.com/foo/foo.tar.gz"],
  sha256 = "0a6717e765818538f153ac27ca6cf97c4290cb25dd374017e1dc2bd0d6f6bf5c",
  build_file = "@a//:foo.BUILD",
  workspace_file = "@b//:foo.WORKSPACE",
)

http_archive(name = "a", ...)
http_archive(name = "b", ...)

```

As circular dependencies are not allowed, projects can always avoid this
confusingly-looking semantics of a recursive bazel sync by presenting the
entries in their `WORKSPACE` file in topological order.

### Recursive `sync` is necessarily sequential

Another consequence of having a  precedence order on definitions of external
repositories is that the fetch of a repository may only be started once all
places that might find defintions take precedence have been fully discovered
(as no observable behaviour, including network access and errors may come
from a defintion that is overridden by another one). For depth-first traversal,
this implies strictly sequential fetches. This is costly (in terms of wall-clock
time), but probably acceptable, given that in the intended
use case, recursive syncs to update the resolved information, are rare.

### Interaction with repository renaming

The [design document on diamond
splitting](https://docs.google.com/document/d/1254CQ8T4Rmeasg4NO1NPail2kLPC50VJ7Ok6JsoSe-c/)
proposes that for recursive workspaces, the renamings assigned for a repository
are also applied to the the repositories pulled in inderectly from the
`WORKSPACE` file of that repository. We follow this proposal.

Moreover, we suggest that the renaming be applied everywhere in the external
repository.
- If the `WORKSPACE` file introduces an external repository, its name is mapped
  before deciding whether it is already present or has to be added to the
  global name space (this was already part of the quoted design document).
- If we pull in a repository from the workspace, the renamings are composed
  (in the sense of function composition, where a dict is seen as a function
  by taking the identity on all arguments that are not keys in the dict).

For example, assum the `WORKSPACE` of `@ext`, where `@ext` is an external
repository with `assignment = {"a":"b", "u":"v", "x" : "y"}`, contains the
following entry.

```
load("@bazel_tools//tools/build_defs/repo:http.bzl", "http_archive")

http_archive(
  name = "a",
  urls = ["https://downloads.example.com/a-1.0.tar.gz"],
  assignments = {"z" : "x", "s" : "t"},
)
```

Then, it will first be checked, if a repository `@b` is already in the global
context. If this is the case, then nothing will be added. Otherwise, a new
repository `@b` will be added, as if there was the following declaration in the
global workspace.

```
load("@bazel_tools//tools/build_defs/repo:http.bzl", "http_archive")

http_archive(
  name = "b",
  urls = ["https://downloads.example.com/a-1.0.tar.gz"],
  assignments = {"z" : "y", "s" : "t", "a" : "b", "u" : "v", "x" : "y"},
)
```


### "Declare what you use" might lead to dropping of top-level definitions

It is generally good practise to declare all the dependencies of a project
that are used directly, including those that are also a dependency of another
directly used dependency. The reason for this approach (sometimes also called
"strict deps") is that direct dependencies might change their dependencies and,
unless we declare all direct dependencies, some might become dangling once they
no longer get pulled in indirectly.

We encourage declaring all direct dependencies on external repositories in the
world of recursive workspaces as well. This might imply that a definition in
the top-level workspace is ignored if the same repository is pulled in earlier
as an indirect dependency. As long as the dependency graph at a repository level
is cycle free, the order can be chosen in such a way, that the definition in the
top-level `WORKSPACE` file is discovered first. However, such an order might
still become invalid if the declared (branch-following) direct dependencies
change their dependency structure. Hence ease of declaring intent in the
`WORKSPACE` file (i.e., not having to care about the structure of indirect
dependencies) is conflicting with the desire of not having ingored repository
definitions in the top-level `WORKSPACE` file. This proposal favours simplicity
of writing `WORKSPACE` files over ease of reading off the version of a
dependency from the `WORKSPACE` file.

### Opt-out of recursion to break cyclic dependencies

While the definitions of repositories have to be acyclic, and the dependency
graph of the targets has to be acyclic as well, the latter need not induce
a cycle-free graph on the repositories the targets come from. In other words,
there might well be repositories `@A` and `@B` such that a target from `@A`
depends on some target in `@B`, and some target in `@B` depends on a target of
`@A`. In this case, the `WORKSPACE` files of `@A` and `@B` contain each other,
making it a priori impossible to specify both `@A` and `@B` in a recursive
workspace.

As an escape-hatch for these situations, i.e., cyclic dependencies between
repositories in conjunction with the need to chose specific versions for
both, we allow another magic parameter `recursive` to every repository rule
(magic in the sense as `assignments` is a magic parameter that has a special
meaning regardless of how the rule otherwise choses to interpret its
parameters). If set to `False`, it will prevent recursive expansion of that
repository, even in recursive `sync`s. For commands other than a recursive
`sync`, this parameter is always ignored. If a rule is not recursive, it is
the responsibility of the author of the global `WORKSPACE` file to ensure that
all dependencies of the targets in that repository are added to the global
workspace in some way.

### Support for resolved files in direct dependencies

This design basically proposes using resolved files as a checked-in cache for
the depth-first traversal of the dependency graph. Now, if a direct dependency
has a resolved file already, it might be desirable to use that instead of
recursively traversing its `WORKSPACE` file. First of all, this would
be faster, as fewer repositories would have to bloaded. Second, and more
importantly, the commited resolved file might contain a well tested snapshot of
the needed dependencies that works well together. So, to allow supporting
reusing resolved files in dependencies, we allow the `recursive` parameter to
be a string as well. If it is, and a file with that name exists relative to the
top of the external repository, then this file is read as a resolved file
instead of recursively interpreting the `WORKSPACE` in this repository.
