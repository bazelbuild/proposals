---
created: 2020-10-09
last updated: 2020-10-09
status: Draft
reviewers:
title: Adding http mirror to accelerate repos downloading
authors:
  - yancl
discussion threads:
---

# Abstract

Every time we build our projects by Bazel from a clean environment(ci for example),
it will cost a lot of time to download the external artifacts(sometimes it even breaks because of network jitters).
So maybe it will be better for Bazel to support something like mirror server so that
we can maintain our own mirrored external artifacts and speed up the building process.

# The problem

Currently we can add mirrored version urls like [this](https://github.com/bazelbuild/rules_go/blob/v0.24.3/go/private/repositories.bzl#L50)
to our `http_archive` rule to make it. but there are two problems:

* We need to change a lot of WORKSPACEs of our own codebase
* We can't change the starlark functions 3rd repos defined like [this](https://github.com/grpc/grpc/blob/v1.32.0/bazel/grpc_deps.bzl#L6) easily

# Proposal of the solution

We propose to add a new flag(`--http_mirror`) that auto prefix the `--http_mirror` to
the urls of the rule `http_archive` and then redirect the http(s) request to the mirrored server.

If the mirrored server failed, NOT FOUND or INTERNAL ERROR, for example, retry the original server
to make it more robust.


# Bazel

## Implementation plan

1. Add a new `--http_mirror` flag to `RepositoryOptions`

2. Refactor the existing `HttpDownloader` to auto convert urls to mirrored urls

## URL mirror rule
- http(s)://host[:port]/path --> {http_mirror}/host[:port]/path
