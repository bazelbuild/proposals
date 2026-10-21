---
created: 2026-10-20
last updated: 2026-10-20
status: Draft
reviewers:
  - meteorcloudy
title: IDE provider
authors:
  - aehlig
---

# Overview

Add a new module to BCR that defines a provider by which rules
optionally can communicate to IDEs the structure of the target, in
particular with respect to how it is built.

# Background

One of the advantages of a generic build tool like `bazel` is
that users can bring rules for new languages, as well as rules for
custom ways of building code in existing languages. In `bazel`,
interoperability of all these rules (including with non-public ones)
is obtained through providers: rules can provide consuming targets
with additional information on how to use that target correctly.

The providers currently in use focus on how to _use_ a target (e.g.,
visible header files, link order). IDEs, on the other hand, need
information on how a target _was built_: what the source files
were, with which flags were they compiled, etc. Therefore, IDEs
currently have to resort to inspecting the rule attributes; while
this works well for known rules, it breaks down for custom rules,
even though they might be for a language known to the IDE.

Therefore, we propose to define a standard IDE provider allowing
rules to provide information on the way a the target was build. It
is optional for rules to opt into also providing this provider. But
if they do, IDEs can properly index that target, even if the rule is
not known to them; this is particular useful for non-public rules.

# Detailed Proposal

# Adding a new provider

A new module is published
to [BCR](https://github.com/bazelbuild/bazel-central-registry) only
defining a provider. Having the provider definition in a separate
module allows rules to opt in without transitively pulling in too
many dependencies.

This new provider supports the following fields, where `extra_info`
serves as an escape hatch for not-yet thought-of use cases. As with
all bazel providers, each field is optional. Some might not make
sense for some rules.

- `language`: a string to identify the programming language the source
   files of this target are written in. To obtain a unique naming,
   programming languages are identified by the canonical BCR
   rule set for that language; in doubt, the rule set existing
   on BCR first is considered canonical for that language.
   So typical values would be `"rules_cc"`, `"rules_java"`,
   `"rules_kotlin"`, `"rules_python"`, etc.

- `sources_with_flags`: a list of `struct`s containing the
  following fields.

   - `sources` a `depset` of `File`s, describing source files
     compiled by actions defined by this target.

   - `flags`: a list of strings, describing those compile flags
     influencing how the source fill is to be interpreted (like
     language standards, defines); flags only specifying location
     of input or output file (like the `-o` of the `C`-compiler)
     should not be included.

  For many rules, this list will have only a single entry as
  all source files of the target are compiled in the same way.
  However, some rules may accept different kind of source files
  and handle them differently (e.g., `cc_library` accepts `.cc`
  files, headers, objects, etc). Non-compiled sources of that
  target should still be listed with empty `flags`. It is expected
  that the IDE has enough knowledge of the language to recognize
  the different source groups by file names.

- `resources`: a depset of `File`s that end up in the artifacts
  produced by this target but are not treated as regular sources
  for the respective language. An example are the resources in a
  `java_library`.

- `interpreter_flags`: a list of strings describing any
  semantically-relevant flags given to the interpreter. This is
  relevant for non-compiled languages where options given to the
  interpreter might affect how the source files are to be interpreted.

- `dependencies`: a `dict` mapping strings to `depset`s of labels.
  To allow consistent use of keys, the module will also define
  string constants for `COMPILE_TIME`, `RUN_TIME`, and `TOOL`
  dependencies.

- `extra_info`: a `dict` mapping strings to arbitrary values; can
  be used to communicate additional values via a pre-agreed key to
  the IDE. Unless those values are very specific to the project-IDE
  pair, after finding out that those additional values are useful,
  it should be aimed for standardization by adding a (pull) request
  to extend the standard fields.

## Evolution of the provider

By the way BCR resolves dependencies, a rule can obtain a version
of a dependency that is newer than the one requested. Therefore,
it is important that the provider remain backwards compatible.
Hence fields that might no longer be needed will be deprecated and
newer versions of rules will stop filling them, but they will not
be removed from the provider to avoid breaking old rules versions
still in use.

If, on the other hand, future evolution of `bazel` or IDEs require
additional information to be passed, a patch should be submitted to
the repository defining the provider. It will be reviewed, merged,
and a new version of the module will be published to BCR. If rules
started using `extra_info` to give that information before the
update of the BCR module, they should continue to do so for a while
till IDEs have caught up and use the official field.

## Adoption by rules

Rules can opt in to use this new provider to improve support of
their targets by IDEs. This is particular useful for the lesser-known
rules. In particular, it is recommended for non-public rules (e.g.,
company-internal rules) as IDEs otherwise have no way of knowing
about the way they work.

In order to opt into supporting this provider, rules add a dependency
on this new BCR module and make the implementation function
additionally return this new provider is all relevant fields filled.

## Support by IDEs

IDEs may start querying target for the new provider, typically
in form of running an aspect over certain user-selected targets.
If the new provider is given, information contained in its value
should be given precedence over all other means of obtaining the
same information. In particular, IDEs and their aspects should
refrain from inspecting attributes of targets providing this new
IDE provider. Information on using the target (as is needed for
depending targets) should still be obtained from the standard
language provider.

# Alternatives considered

Certain design choices could have been made differently.

- We could have opted for a provider without enforced fields.
  This would have solved the problem of backwards compatibility
  naturally. However, this would have made it hard to have a central
  reference on the fields that IDEs might care about. Ensuring that
  the fields go through the process of code review and publishing
  a new version to BCR helps ensuring a central point of reference.

# Backwards Compatibility

Adding a new module to BCR is backwards compatible. Rules returning
an additional provider is backwards compatible; more over, using
the additional provider is opt-in for rules.
