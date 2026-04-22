---
created: 2026-04-14
last updated: 2026-04-14
status: Draft
reviewers:
  - meteorcloudy
title: BCR Module indexing
authors:
  - jordan-bonser
---

# Abstract

Currently the only way to discover bazel modules in the Bazel Central Registry is to use the website(https://registry.bazel.build/).

For privately hosted registries there doesn't appear to be any way to do this, and especially not from the Bazel CLI.

This proposal aims to add indexing at the registry level, so all registries can have their modules be discoverable/enumerable.

# Background

The Bazel Central Registry is currently implemented as a static HTTP-hosted repository where each module is stored under a directory corresponding to its name. While this design keeps the hosting model simple, it lacks a registry index describing the set of available modules.

This means unless you know a module exists in a registry and what its exact name is, it is difficult to know what modules are available.

# Proposal

I propose we add an `index.json` under the `modules` directory of the registry, which will list the modules that exist in the registry.

The format of the file would be a simple JSON object, which lists out the modules in the registry:

```
{
  "modules": [
    "abc",
    "abseil-cpp",
    "abseil-py",
    "ada-py",
    "aexml",
    ...
    "toolchains_llvm"
  ]
}
```

## Motivation

By implementing this proposal, it opens the door to more useful follow on proposals.

One example is the ability to add a module search functionality to the bazel CLI. This functionality could iterate over all the registries that you've registered in your `.bazelrc`, and find modules related to your search term.

Example `.bazelrc`:
```
common --registry=https://bcr.bazel.build
common --registry=https://my_private_registry.com
```

Example future functionality that is made capable:
```
$ bazel mod search "llvm"

"@toolchains_llvm" (https://bcr.bazel.build)
"@llvm" (https://bcr.bazel.build)
"@private_llvm" (https://my_private_registry.com)
```

This functionality would make use of the `index.json` to perform the lookup.

## Adding a Module

As part of adding a module to the BCR, there needs to be a mechanism to ensure that a PR to add a module also has the addition of the module to the `index.json`. One approach to this would be to add logic to a script ran as part of the PR Pipeline, that would ensure the new module exists in the index. The `bcr_validate.py` may be a good candidate for this logic.

## Handling Merge Conflicts

With there being a single file that needs to be modified on each module addition, it could become an issue in terms of merge conflicts. To combat this I suggest we do a number of things:
- Ensure the `index.json` is in alphabetical order.
- Create a custom Git merge driver to resolve the conflicts in alphabetical order.

### Merge Driver
Creating a custom merge driver is fairly simple. The approach that is usually taken is to add a new merge driver to the git config:

For example:

```
[merge "alphabetical"]
    name = merge using alphabetical diff
    driver = git merge-file -L %A -L %O -L %B -p --diff-algorithm=<chosen-diff-algorithm> > %A %O %B > %A
```

Enabling this for a specific file can be done by adding to the `.gitattributes` like so:

```
modules/index.json merge=alphabetical
```

### Diff Algorithm

The simplest approach may be to use a different git `diff-algorithm` for the merge. This completely depends on how effective the various diff algorithms deal with the alphabetically sorted module list. If there is a clear outlier which resolves the conflicts correctly then this would be as simple as changing the driver to the above git merge-file example `git merge-file -L %A -L %O -L %B -p --diff-algorithm=<chosen-diff-algorithm> > %A %O %B > %A`.

### JSON Merge Driver Script

Alternatively using a JSON friendly programming language, you could create a script which is JSON aware and performs the merge via sorting the JSON alphabetically:

```
[merge "alphabetical"]
    name = merge using alphabetical diff
    driver = python scripts/merge_json_sorted.py %O %A %B
```

# Backward-compatibility

There needs to be an upgrade path available to allow registries that have already been created to add an `index.json` and have it populated with modules that are already in their own registry.

There should also be the assumption that not all registries will have an `index.json` and any future tooling around this functionality will need to also assume this.

# Open questions

- What information should be added to the `index.json`?
  - Current design is to start with just the module names and expand as needed.
