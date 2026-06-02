---
created: 2026-04-14
last updated: 2026-06-02
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

I propose we add an `index.json` at the root of the registry, which will list the modules that exist in that registry.

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

## Generating the Index

To ensure that we don't add any extra churn when adding modules to the registry during the normal PR process, the `index.json` should be created as a pre-deployment step rather than living inside of the Git repository itself.

A separate script will be made that will walk the `modules/` directory of the repo and generate the index file from the information contained in there.

In the future, if it is deemed that additional information about the modules themselves is needed then the script can be adapted to extract that information from the `metadata.json` file of each of the modules.

In terms of enabling this for the BCR, the generation of this file could be performed in [bcr_postsubmit.py](https://github.com/bazelbuild/continuous-integration/blob/f64e8b84498cfde2fb1845c5ca7e4761ca30c129/buildkite/bazel-central-registry/bcr_postsubmit.py#L153) before being uploaded to GCS.


# Backward-compatibility

There needs to be an upgrade path available to allow registries that have already been created to add an `index.json` and have it populated with modules that are already in their own registry. This should be fairly simple as the file will be generated rather than living in the git repository itself. Existing registries can download the script and run it from their own registry repo, and then upload that to where their registry is hosted.

For registries which are not backed by a git repository, generating the file in the JSON format shown above should be trivial.

There should also be the assumption that not all registries will have an `index.json` and any future tooling around this functionality will need to also assume this.

# Open questions

- What information should be added to the `index.json`?
  - Current design is to start with just the module names and expand as needed.
