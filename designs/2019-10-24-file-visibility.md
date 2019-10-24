---
created: 2019-10-24
last updated: 2019-10-24
status: To be reviewed
reviewers:
  - laurentlb
title: Visibility for source files
authors:
  - aehlig
---


# Abstract

We propose that source files used in a BUILD file do not implicitly inherit the
public visibility of the package.

# Background

Bazel uses labels as a uniform name space for all dependencies,
regardless of their type. Therefore, labels must also exist for
source files being depended upon. To simplify writing `BUILD` files,
these are added implicitly. More precisely, after evaluating a
`BUILD` file, targets are created for all not-yet defined labels in
the same package, assuming they are source files. These implicitly
defined targets inherit the `default_visibility` of the package.

## Example and description of the problem

Consider the following `BUILD` file.

```
package(default_visibility=["//visibility:public"])

exports_files(["app.1"])

cc_binary(
  name = "app",
  srcs = ["main.c", "util.c", "util.h"],
)
```

After evaluating the `BUILD` file, the only declared targets are
`:app` and `:app.1`, both inheriting the `default_visibility`.
Moreover, `:app` depends on on the targets `:main.c`, `:util.c`, and
`:util.h` in the same package. As these targets are not declared, it
is assumed that they are source files, and they are added implicitly
as targets, as if there were an additional `exports_files(["main.c",
"util.c", "util.h"])`; those implicitly added targets also inherit
the `default_visibility`. So, just from the fact that those files
are used, they become publicly visible, even though the
intention might have been to only expose the application and its
manual page. As a result of this, every package can depend on,
e.g., the file `util.c` directly, making it hard for the package
to evolve this internal interface without breaking other packages.

# Proposal

We propose that implicitly added targets are added with visibility
`//visibility:private` regardless of the `default_visibility` of
the package. If a source file should be exposed more broadly, an
explict `exports_files` would have to be added.

# Backward-compatibility

The change is not backwards compatible. Therefore, it will be
guarded by a flag `--incompatible_no_implicit_file_export`.
