---
created: 2018-11-27
last updated: 2018-11-27
status: draft
title: Repository rules with multiple return values
authors:
  - aehlig
---

# Repository rules with multiple return values

Third-party dependencies are typically packaged in some form or another, often
by a language-specific package manager. In this case, computing transitive
dependencies is best done by the native package manager for these kind of
packages (as opposed to bazel-bazed projects where [recursive
workspaces](2018-11-07-design-recursive-workspaces.md) would help). Still, bazel
needs to know about those dependencies to keep a correct dependency graph.

This document describes the existing approaches and describes a design based
on repository rules expanding to multiple repositories.

## Existing approaches

As the need to build against packaged dependencies existed since the
beginning of bazel, quite a few approaches have been developed that
work with the current bazel with out any changes.

### External generation

One approach to deal with external package management is to no longer
treat the `WORKSPACE` file as source of truth, but generate it from
a description mentioning the packages bazel directly depends on. This
generated `WORKSPACE` file might even be committed, so that, when not
updating the dependencies, they can be fetched independently of the
package manger (e.g., as `http_archive`) and all of bazel's caching
mechanisms for downloaded files apply.

As soon as dependencies from more than one package manager is used, this
approach, however, requires the various generation mechanisms to interact
cleanly with each other and only update "their" part of the `WORKSPACE`.
This requirement can be achieved by having one generated `.bzl` file for
each package manager from which an import function is loaded and called
in the `WORKSPACE` file.

Another drawback of this approach is that after updating the description
file for the external packages, the step of regenerating the `WORKSPACE`
file has to be triggered manually.

### One big external repository for the transitive dependencies

A simple approach based on external repositories is to have one repository
rule that takes the description of the third-party dependencies. This
rule would fetch all packages together with their transitive dependencies
and make them available in one big external repository for this package
manager, together with generated `BUILD` files describing the dependencies.

While this approach is simple, tracks the dependency on the description file,
and does not cause any issues when dependencies
from multiple package managers are used, all resolving, prefetching, and caching
rely on the ability of the package manager's ability to do so. All these
operations are outside the knowledge of bazel and hence bazel cannot
assist in any way.

### External repository exporting an import function

Another possible approach is to have one repository rule per package
manager. This rule, however, would not fetch the dependencies, but only compute
the transitive closure and serialize this information in a generated `.bzl`
file. From that `.bzl` file, the `WORKSPACE` would load a designated function
importing the needed packages in a convenient way (e.g., as `http_archives`).

In this way, dependencies on the description file are tracked correctly, and
all of bazel caching mechanism for file downloads apply. The approach also works
well if multiple package mangers are used.

The approach also works with using [resolved
files](https://docs.google.com/document/d/1kVNXcw3nLlfFQRR_87SGOka9DJ8nnawlYHUIK4m3s0I/edit)
for freezing dependencies, even if the package description contains floating
references, e.g., to the latest release of a particular package. What is lost in
this situation, however, is the information that the individual package
repositories where actually created by the package-manger repository rule. While
this is not a problem for normal operation, it will not work with partial syncs
(e.g., "update all maven dependencies, but leave the rest of the resolved file
fixed"), once we will implement them.

## True expansion to multiple repositories

Implicit in the design of [resolved
files](https://docs.google.com/document/d/1kVNXcw3nLlfFQRR_87SGOka9DJ8nnawlYHUIK4m3s0I/edit)
is another approach to rules for package managers: workspace rules expanding to
multiple repositories. The purpose of this document is to work out the details
of this design.

## Rules returning values of type list

Repository rules are allowed to return a description of the modified arguments
they can be called with in order to reproduce the same directory. This is to
allow the transition from a floating branch to a frozen version.

This design adds a new kind of repository rules: those that always return a
list with descriptions of other repositories. The intended use case are the
third-party package managers mentioned earlier. Such a rule would take as
argument some form of description of the list of packages directly needed
and return a list of instructions for fetching the all the packages needed,
including the transitive dependencies. This operation will need access to the
system, and potentially the network, to obtain the needed meta data from the
package manager's data base, but it is typically a cheap operation, as the
actual packages themselves are not fetched.

## Unconditional execution of rules returning multiple values

One fundamental difference between normal repository rules and the newly
introduced kind is that the latter introduce new names. For a normal repository
rule, the name of the repository can already be determined without executing the
rule; this allows for the current evaluated-when-needed semantics of external
repositories. If, on the other hand, a rule is allowed to expand to multiple
repositories, it has to be executed unconditionally, just to know which names
are currently available and how they are bound (an information that is needed
at the next `load` statement, as well as at the end of the `WORKSPACE` file,
then a `BUILD` file refers to an external repository).

### Opt-in by introducing a new rule class

As the requirement of unconditionally evaluating repository rules is different
from the existing semantics of repository rules, we make the new behavior opt-in
by restricting the possibility of expanding to multiple repositories to a newly
introduced kind of repository rules. So, when evaluating a workspace, bazel will
be able to tell if a rule has the ability to expand without having to execute
it.

More precisely, the `repository_rule` function will get an additional named
argument `meta_rule` defaulting to `False`. If this argument is true, the
`implementation` function will be required to return a list of repository
descriptions and it will be executed unconditionally.

### Precise output format of the new rule class

The return value of those new rules will follow that of resolved file, i.e.,
it will be a list of dictionaries with (initially) two entries. For the key
`rule_class` a string will be specified containing a label of a `.bzl` files and
an exported symbol, separated by the `%` character. The exported symbol must be
a repository rule; it must not be a meta-repository rule.
For the key `attributes` a dictionary with the arguments to that rule will be
specified; as the rule is a repository rule, all arguments will necessarily be
key-word arguments.

This list returned will become the `repositories` entry for executing the
meta rule. Also, when evaluating the `WORKSPACE` file, the returned repositories
will be treated as defined immediately after the meta rule. As none of the
returned repositories is allowed to be a meta-rule, no recursive expansion is
necessary.

