---
created: 2018-10-29
last updated: 2018-10-29
status: Implemented
reviewers:
  - dslomov
  - dannark
title: Progress reporting for external repositories
authors:
  - aehlig
---


# Abstract

The repository context will be extended by a method `report_progress`
allowing repository rules to report about their progress.

# Background

The standard mode for bazel is to build everything from source,
including third-party libraries the project depends on. To build
such a unified view of the transitive sources, third-party sources
are usually added as external repositories, i.e., as descriptions
on how to fetch, unpack, and modify the corresponding code.

Depending on network connectivity, number of archives to be fetched,
and their respective sizes, setting up an external repository can
take quite some time. Therefore, it is desirable that the user be
informed about this activity, to be able to understand the timing
of the build. Moreover, setting up an external repository is more
structured than the typical bazel actions like compiling or linking,
so the simple [information about which repositories are being
fetched](https://github.com/bazelbuild/bazel/commit/30cd6ab8cdcefef92b9af6d9f9f0cee06ce2f2fe)
is not always enough. It is therefore a long standing feature
request to [report progress from a Starlark external repository
rule](https://github.com/bazelbuild/bazel/issues/1289).

# Proposal

A new function `report_progress` is added to the repository context; the
function accepts a single argument which is a string. Starlark repository rules
may use this function to set their one-line progress status. Setting a new
progress status overrides any status set previously. So, if a rule wants
to report on achievements already made, that has to be repeated in subsequent
status updates. Typically, however, we expect status messages like
"Fetching source archive (step 1/3)",  "Fetching patches (step 2/3)", and
"Patching soruce tree (step 3/3)".

It is up to the implementing UI to decide how this information is presented.
Generally, we expect the UI to present the status of an external repository
rule in a similar way as (non-workspace) actions, e.g., showing the total number
of fetches going on and a sample of the earliest started fetches still going
on; instead of the `progress_message` of a regular action, the repository name
together with the argument of the last call to `report_progress` will be used
for an ongoing fetch.

This proposal reflects the idea, that execution of a repository rule is
basically a structured action. For actions, a one-line description is
passed as `progress_message` argument to `ctx.run`. For the execution
of a repository rule, the rule can update the progress message before
each step to inform the user what the next step will be.

As for the `progress_message` paramter of a regular action, rule owners
should be aware that the UI might not be able to show more than the first
line shortened to the terminal width. For example, in the [sample
implementation](https://bazel-review.googlesource.com/c/bazel/+/79731), a
typical screen shot could look as follows.

```
(12:01:26) Building: no action
    Fetching @rule0; Final steps...
    Fetching @rule1; Performing step #0/4
    Fetching @rule2; Performing step #2/4
    Fetching @rule4; Performing step #1/4
    Fetching @rule3; Performing step #0/4
    Fetching @rule5; Initial set up
    Fetching @remotejdk10_linux_aarch64; fetching
    Fetching @remotejdk10_macos; fetching
    Fetching @remotejdk10_win; fetching
    Fetching @remotejdk10_linux; fetching
    Fetching @remotejdk_macos; fetching
    Fetching @remotejdk_linux; fetching ... (14 fetches)
```


# Backward-compatibility

This proposal only adds a new function to the repository context; exisiting
functions are not modified. The change is fully backwards compatible.
