---
created: 2023-06-08
last updated: 2023-06-09
status: In Review
reviewers: gregestren
title: Standard Platform Transitions
authors: katre
discussion thread: https://github.com/bazelbuild/bazel/discussions/18628
---

# Overview

Many rule authors and users want to create targets that can easily change the
target platform: rule authors may want to add convenient rule attributes for
this, whereas users frequently need to bundle together executables for multiple
platforms in a single high-level artifact.

A [previous proposal was
submitted](https://github.com/bazelbuild/proposals/blob/main/designs/2022-08-03-platforms-on-targets.md),
but the functionality was too narrowly scoped to fit most use cases, and the
changes to Bazel were too high to be worth it. This proposal instead suggests
several transitions and rules which can be added to the existing
[bazelbuild/platforms](https://github.com/bazelbuild/platforms) repository, and
which rule authors or users can then import and use. Because these are pure
Starlark implementations, with no Bazel changes, rule authors can feel free to
extend or ignore these as needed.

# New Transitions

## Single Platform Transition

Rules that want to change the target platform can use the `change_platform`
transition, which can be parameterized in a few different ways. These work with
incoming (rule) transitions and outgoing (attribute) transitions.

### Change to a static platform

Some rules want to change to a single, statically determined platform, either
for the entire rule or for a specific attribute. They can use the `platform`
parameter to `change_platform`, passing a Label to use for the new value of the
`--platforms` flag.

```py
my_rule = rule(
  cfg = change_platform(platform = Label("//new/target:platform")),
)
```

### Change based on the `platform` attribute

Some rules want to change to a single platform based on a standard attribute
named “platform”, either for the entire rule or for a specific attribute. The
rule is responsible for defining the `platform` attribute.

**Open Question**: Should we define a function to help define the attribute?

```py
my_rule = rule(
  cfg = change_platform(),
  attrs = {
    "platform": attr.label(providers = [platform_common.PlatformInfo]),
  },
)
```

### Change based on a different attribute

Some rules want to change to a single platform based on a custom attribute,
either for the entire rule or for a specific attribute. The rule is responsible
for defining the attribute.

```py
my_rule = rule(
  cfg = change_platform(attribute = "my_platform"),
  attrs = {
    "my_platform": attr.label(providers = [platform_common.PlatformInfo]),
  },
)
```

## Split Platform Transitions

Rules that want to perform split transitions can use the `split_platforms`
transition, with similar parameters. These only work with outgoing (attribute)
transitions.

### Change to a set of static platforms

Some rules want to change to a set of statically determined platforms for a
specific attribute. They can use the `platforms` parameter to
`split_platforms`, passing a list of Labels to use for the new value of the
`--platforms` flag.

```py
my_rule = rule(
  attrs = {
    "deps": attr.label_list(
      cfg = split_platforms(platforms = [
        Label("//new/target:platform1"),
        Label("//new/target:platform1"),
      ]),
    ),
  }
)
```

### Change based on the `platforms` attribute

Some rules want to change to a set of platforms based on a standard attribute
named “platforms” for a specific attribute. The rule is responsible for
defining the `platforms` attribute.

```py
my_rule = rule(
  attrs = {
    "deps": attr.label_list(
      cfg = split_platforms(),
    ),
    "platforms": attr.label_list(providers = [platform_common.PlatformInfo]),
  }
)
```

### Change based on a different attribute

Some rules want to change to a set of platforms based on a custom attribute for
a specific attribute. The rule is responsible for defining the attribute.

```py
my_rule = rule(
  attrs = {
    "deps": attr.label_list(
      cfg = split_platforms(attribute = "my_platforms"),
    ),
    "my_platforms": attr.label_list(providers = [platform_common.PlatformInfo]),
  }
)
```

## Implementation notes

The different variants of these transitions are technically functions that
create transitions (in order to parameterize the new flag values correctly).
Each will be given a unique name based on the current package and be used only
for the rule it is attached to. Specifically, using `change_platform(platform = foo)`
 multiple times from different attributes or rules will generate multiple
transitions.

An alternate approach would be to name each based on a hash of the parameters,
and thus avoid duplication in the same package. This can be explored if the
overhead of multiple identical transitions seems to be too high.

# New Rules

## Depend on a target built for a different platform

The `platform_data` rule can be used to change the target platform of a target,
and then depend on that elsewhere in the build tree.

```py
cc_binary(name = "foo")

platform_data(
    name = "foo_embedded",
    target = ":foo",
    platform = "//my/new:platform",
)

py_binary(
    name = "flasher",
    srcs = ...,
    data = [
        ":foo_embedded",
    ],
)
```

Regardless of what platform the top-level `:flasher` binary is built for, the
`:foo_embedded` target will be built for `//my/new:platform`.

**Open Question**:

Should there be an inline macro variant, like this:

```py
cc_binary(name = "foo")

py_binary(
    name = "flasher",
    srcs = ...,
    data = [
        platform_data(
            target = ":foo",
            platform = "//my/new:platform",
        ),
    ],
)
```

This is easier for BUILD authors, but obscures the name of the new target,
which may be undesirable.

## Depend on a target built for multiple platforms

In theory, a similar `multiplatform_data` rule could be written which does a
split transition on a `platforms` attribute, but it is currently unclear what
the best way to pass that data back to the parent target would be, given that
we do not want to rewrite rules to understand a custom provider. This will not
be implemented until there is a clearer use case.

