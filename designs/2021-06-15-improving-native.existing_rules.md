---
created: 2021-06-15
last updated: 2021-06-23
status: To be reviewed
reviewers:
  - haxorz
  - comius
  - kkress
title: "Improving native.existing_rules"
authors:
  - tetromino
  - brandjon
---

# Abstract

This document proposes an API change to `native.existing_rule()` and
`native.existing_rules()` (collectively referred to as “`existing_rule/s`”),
which allow macros to introspect on the targets already defined in the current
package. The aim is to significantly improve performance while essentially
preserving the existing API.

A comprehensive redesign of the API, e.g. to reassess what use cases should be
supported and whether its expressivity should be changed, is out-of-scope.
Likewise, this doc does not propose any changes to the representation of reified
attribute values, such as using lists rather than tuples as the type of value
corresponding to label list attributes.

`existing_rule/s` can also be called from within WORKSPACE-loaded .bzl files to
introspect on the previously defined repositories. Since repositories are
internally represented as targets, the changes suggested in this doc should
apply seamlessly to that usage as well.

# Use cases

BUILD files and Starlark macros sometimes need to query for other targets
(typically, selected by rule kind, name, and/or tags) that exist in the current
package. Some reasons include:

*   Validate a package’s targets by failing at package load time if any foo
    targets have (or don’t have) certain dependencies, tags, visibility, or
    names matching a particular pattern. (Note that this use case would be
    addressed by the
    [package validation proposal](https://docs.google.com/document/d/1FfRXzW5RAB7tixbd1Lac51GNqNGQN932ZuTRdg8oUWU).)
*   Programmatically create a foo target per each bar target, sometimes with a
    check that a foo target with an expected name was not already initialized.
    Example:
    [apollo](https://github.com/ApolloAuto/apollo/blob/master/tools/cpplint.bzl).
*   Programmatically create a foo meta-target depending upon or aggregating all
    bar targets so far. Examples:
    [tensorflow](https://github.com/tensorflow/tensorflow/blob/master/tensorflow/lite/build_def.bzl),
    [tink](https://github.com/google/tink/blob/master/java_src/tools/build_defs/tink_java_rules.bzl).
*   From WORKSPACE files: verify whether a rule set was imported under a
    specific name. Example:
    [rules_rust](https://github.com/bazelbuild/rules_rust/blob/main/rust/repositories.bzl).

# Current situation

`existing_rule/s` return the values as mutable dictionaries containing copies of
all target attribute values. This results in unnecessary memory bloat,
especially when not all targets are needed, or when only the targets’ names are
needed. In the worst case, it allocates a large object just to test whether a
single target exists; for these cases `existing_rule()` would be much more
efficient. The memory bloat is quadratic when `existing_rules()` is called in a
loop that creates targets, or in a double loop, or in a macro that is called
many times within a package.

There has been a feature request inside Google to improve `existing_rules()`
performance since 2019, when it was discovered that it accounted for 1.74% of
Bazel CPU use in the main CI pipeline (although many inefficient uses of
`existing_rules()` in Google’s monorepo have been fixed since then).

# Proposed solution

We will drastically improve the efficiency of `existing_rule/s` by 1) avoiding
unnecessary copying and 2) making the enumeration of targets lazy and the
initial call constant-time.

Instead of a mutable dict, `existing_rule()` would return an **attribute view**,
an immutable dict-like object that provides access to the target’s attributes.
This view type would also replace the inner dicts seen in the result of
`existing_rules()`. There are two warts: one is that when retrieving an
attribute whose value is a `select()`, we have to keep the
[existing behavior](https://github.com/bazelbuild/bazel/commit/19fe9f087f1122c986a779b29b1c7832507868ea)
of `starlarkifyValue` by making a copy converting from `BuildType.SelectorList`
to `packages.SelectorList`; we will be able to eliminate the conversion only
once the two `SelectorList` types
[are merged](https://github.com/bazelbuild/bazel/issues/13586). The second wart
is list-valued attributes, which, if we want to preserve existing behavior, we
would need to always copy into a tuple.

Instead of a mutable dict of dicts, `existing_rules()` would return a **target
view** wrapping the state of `Package.Builder#targets` at the time of the call;
instantiating new rule targets would *not* alter an existing target view object.
This would be implemented by having the view remember the number of targets,
`n`, in existence when it was created. Since the `targets` field is a
`HashBiMap` (which uses insertion order iteration) and targets are never
deleted, we can simply iterate `n` many times. We would also need a map from a
target to its insertion order in `Package.Builder`, to ensure that subscripting
the view by target name cannot access targets that were not in existence when
the view was created. (If this map turns out to measurably increase memory
usage, we can create and grow it lazily so that packages that never call
`existing_rules()` don’t pay this cost.)

Note that under this proposal, `existing_rules()[name]` would be about as
efficient as `existing_rule(name)` (modulo the cost of the target creation order
map if it’s created lazily), meaning that the former, commonly-seen usage would
no longer be a performance antipattern. Calling `existing_rules()` by itself
would not incur quadratic complexity in either time or space. That could only
happen if the user looped over the result or made an explicit copy,
respectively.

# Backward compatibility

There are two user-visible changes to the return value of existing_rule/s.

*   The type (as seen by `type()`) and stringification (as seen by `str()` and
    `repr()`) would change to indicate that they are views rather than dicts.
*   Mutation of these values would be prohibited. (If that causes significant
    migration pain, we may want to add copy-on-modify behavior emulating a
    mutable dict.)

We propose to enable the change via a single incompatible flag.

Migration would entail not relying on type/stringification info. Based on
`existing_rules()`, `type()`, and `repr()` usage observed in Google’s monorepo,
we expect that very few .bzl files would require any migration.

If a .bzl or BUILD file requires mutability, it would need to be changed to copy
the views into a dict, e.g: `for t in dict(native.existing_rules().items())`.

Past experience shows that changes to `existing_rule/s` may break some users,
but a sampling of several hundred current uses in Google’s monorepo did not show
any where that mutability of the return value is required, so we expect the
number of breakages to be low and that the flag would be easy to flip.

The kind pseudo-attribute that is inserted by `existing_rule/s` remains
unaffected by this proposal.

# Rejected approaches

## Replace with a broader API

In the past, it was suggested that `native.existing_rules` should be completely
redesigned or eliminated altogether: it makes BUILD files brittle and is one of
the factors which prevents multithreaded loading within a package. However,
there seems to be a real recurring need to programmatically define extra rule
targets based on other targets in a package. Any API for doing so would either
depend on where in a BUILD file it is invoked from (causing the same brittleness
as `existing_rules`) or would have to be invoked through post-loading-phase
callbacks (whose order and mutual interaction would be hard to reason about).
Furthermore, a complete redesign of the `existing_rule/s` API would cause
migration pain for users which we would prefer to avoid.

## Filtering the returned targets

We considered several approaches for reducing the size of the dict returned by
`existing_rules()` in cases where users don’t want all targets. Ultimately, we
decided that returning a target view was much simpler and still delivers most of
the performance benefit, in the sense that retrieving information about specific
targets by name is O(1) (or amortized O(1) if the target insertion order map is
generated lazily). The only time built-in filtering is more efficient than using
target views is when the user wants to iterate over some subset of the targets,
e.g. all targets of a given rule kind.

Suppose that the built-in filtering is implemented by internally iterating over
all targets and skipping over ones that don’t match the criteria. This avoids
the need for user-written Starlark filter logic. It also avoids allocating
attribute dicts for skipped targets, though this point is made moot by switching
to attribute views. Overall, the performance advantage is not asymptotic but
rather a possibly significant constant factor (native logic vs Starlark).

On the other hand, we could make the built-in filtering asymptotically better by
avoiding iteration altogether. This could be done by materializing the filtered
result in the target view, and incrementally updating it as new targets are
defined. In the common case there would be a small number of unique queries
executed repeatedly in a given package – for instance, “get me all `cc_library`
targets” is a single query that could be run hundreds of times – so the number
of target views to maintain would be small. But this is a lot of extra
complexity for an optimization that might not be worthwhile. And in packages
where an `existing_rules()` filter is called only once, continuing to update the
list of filtered results until the package is fully loaded would be wasted work.

Therefore, we have decided to forego filtering for now. We will reconsider it in
the future if `existing_rule/s` still proves to be a bottleneck for package
loading after implementing the accepted design.

### Filtering syntax

We considered a few paradigms for how the API to `existing_rules()` would change
if we were to use the above filtering approach.

1.  Add a generic query ability to existing_rules(), a la genquery, to return
    all reified targets matching an arbitrary expression (and restricted to the
    current package). Ex:

    `result = native.existing_rules(query="kind(cc_library, this_pkg:*)")`

2.  Add a dict argument specifying the attributes to match. Ex:

    `result = native.existing_rules(matches={"kind": "cc_library", ...})`

3.  Add keyword parameters for a few supported attributes, ad hoc. Ex:

    `result = native.existing_rules(kind = "cc_library", ...)`

The first approach seems to work too hard to find a common thread between
`existing_rules()` and `genquery`. For instance, operators like `somepath` seem
irrelevant to `existing_rules()`’ use case. The underlying query machinery also
can’t be reused to implement this since `existing_rules()` operates on an
incomplete package rather than the completed result of loading all packages.

The second approach seems general enough on the surface, but it’s unclear how to
handle queries that ask for anything besides exact matches on an attribute
value. For instance, a user may want to get targets that have *all* the tags
from a given list, and/or targets whose kind is *one* of a given list of rule
kinds, and/or targets whose names *start with* a particular string. This kind of
subset/superset, disjunction/conjunction, exact/substring match logic might be
easier to implement ad-hoc with the third approach, e.g. by allowing arguments
like `kind = ["cc_library", "java_library", ...]`, but a design that can satisfy
most current use cases would not be simple.

## has_existing_rules()

We considered adding a new function that simply returns a boolean indicating
whether any targets satisfying a filter expression (described above) exist. This
corresponds to a common use case that today is served very inefficiently by
`existing_rules()`, or (for exact name matches) somewhat less inefficiently by
`existing_rule()`. However, the proposal in this doc makes those performance
concerns obsolete.

## Lints and limits

There has been a feature request inside Google to limit the number of
`existing_rules()` calls per package, weighted by package size. In light of the
expected performance improvements from switching to target views instead of a
dict of dicts, we think that new limits (beyond the existing limits on the
number of Starlark computation steps) aren’t needed.

Likewise, we considered adding a buildifier lint to suggest using
`existing_rule()` in place of `existing_rules()` where possible. Such a lint
would still be worthwhile, but would now bring only a readability and constant
factor performance benefit.

## Memoization

@haxorz proposed caching (and making immutable) the return value of
`existing_rules()` and invalidating the cached value when a new rule target is
instantiated. This approach, relative to status quo, would improve performance
for the “validating existing targets” and “aggregating existing targets into a
meta-target” use cases; for the “create a new target per each existing target”
use case, the cache would thrash and some sort of snapshotting of the memoized
dict would probably be required. In any case, returning a lightweight view
instead of a dict of dicts and avoiding copying all target attributes would make
memoization unnecessary for most users.

## Stateless view

We considered that `existing_rules()` might return a *stateless view* - a view
that doesn’t remember the state of the package at the time it was created, but
that merely always returns all targets currently in existence. This would be
simpler to implement since we don’t have to track a sequence number or a copy of
the insertion order of `Package.Builder#targets`. It was also supposed that the
semantic change would be unlikely to be noticed by users, since most callers of
`existing_rules()` use the result immediately rather than save it for later.
However, a naive implementation would allow infinite loops in macros and violate
the existing restriction that a loop doesn’t modify what is being iterated over:

```
    for r in native.existing_rules().values():
        if r["kind"] == "cc_library" and r["name"].startswith("foo_"):
            add_another_cc_library(name = r["name"] + "_wrapper", ...)
```

The infinite loop may be avoided by snapshotting the result at the point where
iteration begins, but that requires implicit copying or else extra
implementation machinery like what we already proposed.

