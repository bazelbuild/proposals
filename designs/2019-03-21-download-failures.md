---
created: 2019-03-21
last updated: 2019-03-21
status: Implemented
title: Handling download failures
authors:
  - aehlig
---


# Abstract

We propose to add a new named argument `allow_fail` to the methods
`download` and `download_and_extract` of the repository context,
defaulting to `False`. If set to `True`, errors in the download will
not be fatal but only reported to the caller to decide on further
action.

# Background

## Repository Rules

While Bazel was originally designed with a mono repository in mind,
it became clear that not everybody organizes their code in this
way, and additional sources of source code had to be considered.
Also, occassionally, information about the host system are needed,
e.g., the location of the host `C` compiler.

Bazel's approach to all those "non-hermetic" operations is the
concept of external repositories. An external repository is a
collection of source files that are "fetched" once and then treated
in the same hermetic way like any other source file in the main
repository, under the `@nameofexternalrepo//` name space. During
the fetch, however, arbitrary operations, including calling out to
the network or reading not a-priori declared files from the host
system, can be performed.

## Repository Rules

To obtain flexibility (as Bazel cannot anticipate all version
control systems that might be used, or other needs for non-hermetic
actions), instructions for fetching a class of external repositories
(e.g., using a particular version-control system) are described by
a piece of Starlark code. Similar to regular rules, the Starlark
function of those so-called _repository rules_ expects a single
argument, in this case a _repository context_. This context provides
various functions needed to "fetch" an external repository. These
include basic write operations, e.g., functions to create a file
or a symbolic link. Then, it provides a `download` function to
download a file from a list of provided URLs and (optionally) verify
the file agains a known good hash. Finally, it provides with the
`execute` function a generic way to interact with the operating
system by allowing to execute arbitrary commands and inspect
`stdout`, `stderr`, and the exit code. Currently, arbitrary exit
codes of `execute`d commands can be handled by the rule, whereas
an unsucceful attempt to `download` a file results in the whole
external repository being considered failed.

## Repository Cache

The most common way of fetching external dependencies is to fetch
an archive, unpack it, and maybe patch it or add a `BUILD` file. As
Bazel needs to be aware of all dependencies, it is not uncommon, that
several workspaces have dependencies in common. To avoid refetching
the respective archives for every workspace, bazel has a cache,
indexed by the hash of the downloaded file. The hash specified
when downloading a file not only serves as means of verifying the
integrity of the download, it will also be used to avoid unnecessary
downloads, if the file is already in cache.

# Proposal

## Proposed Change

We propose to add an additional, named, parameter `allow_fail` to
`ctx.download` and `ctx.download_and_extract`. It will be boolean
and default to `False`. If set to `True`, failures to download the
specified file will not be fatal, i.e., not mark the "fetch" of the
external repository as failed. To allow the repository rule to act
depending on success or failure of the download, the returned struct
will contain an additional entry `success` indicating the success
of the download. Moreover, if (and only if) `allow_fail` is set to
`True`, the list of URLs provided may also be empty (resulting in
a pure cache probe).

## Use Case: Avoid uncessary credential requests

A recent [feature request](https://github.com/bazelbuild/bazel/issues/7635)
asked support cache probes. The reason for this feature request
is that files are used where some form of credential is needed to
download&mdash;and obtaining that may take time/user interaction/network
access. Now, if the file is already in cache, it would not be necessary
to obtain the credential, but this could not be benefitted from,
as it would require an ability to probe the cache. This proposal
obviously solves this problem, and, in fact, is the third option
in said feature request.

## Use Case: `http_archive` at head

With the introduction of [resolved
files](https://blog.bazel.build/2018/09/28/first-class-resolved-file.html),
it has become desirable to no only write reproducible rules fetching
sources, but also ones "following head". For version control systems,
this requires checking out the `master` branch and recording the
currently latest commit in this branch; both are tasks that can
easily be accomplished by `ctx.execute`. When following release
archives, however, the task of updating basically consists in probing
the (predictable) URL of the next release(s). To allow rules to
do this, an attempt to download is necessary, where a failure in
the download can be handled by the rule. It seems cleaner to add
this option to the `ctx.download` functionality, than relying on
constructs like `ctx.execute(["wget", ...])`; on the one hand, it
is more portable (which of `wget`, `curl`, `fetch`, etc, should be
assumed to be present) and on the other hand, upon success, the file
directly ends up in cache, without the need of a second download.

# Backward-compatibility

As long as the new `allow_fail` option is not explicitly set, the
behaviour of `ctx.download` and `ctx.download_and_extract` does not
change. The struct returned will have an additional entry `success`,
but a struct was choosen precisely as an extension interface to
allow returning additional information in the future.
