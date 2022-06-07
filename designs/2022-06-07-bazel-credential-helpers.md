---
created: 2022-06.07
last updated: 2022-06.07
status: To be reviewed
reviewers:
  - TBD
title: Credential Helpers for Bazel
authors:
  - Yannic
---


# Abstract

Many, if not most, Bazel workspaces have a dependency on connecting to some
online service. This can either be for fetching external repositories (either
through `http_archive`, `repository_ctx.download()`, or `bzlmod`), or for
connecting to a server for `Remote Caching`, `Remote Execution` or uploading
a `Build Event Stream`. In this proposal, we propose a generic mechanism for
Bazel to fetch credentials for authenticating against such services.


# Background

Feature request in the issue tracker:
    https://github.com/bazelbuild/bazel/issues/15013

As of June 2022, Bazel's support for authentication against remote services is
limited to a few hard-coded options:
- `GCP IAM`, which, as the name suggests, is only suitable for authenticating
  against GCP,
- `mTLS`, which is not widely used,
- `.netrc`, which is generic but depends on a plaintext file for reading
  credentials, which is not very secure, and
- `command-line flags`, which is generic but also requires either always passing
  them manually or putting credentials in a plaintext file (in the form of a
  `.bazelrc` file). Additionally, there are several flags depending on the
  remote service (e.g., there's `--remote_header`, but also `--bes_header`)
  which makes this way of providing credentials not very ergonomic.

However, Bazel is not the only tool with a need to authenticate against remote
services. Many other applications have solved similar problems in the past.


# Proposal

We propose to allow Bazel to retrieve credentials from an (external) helper
binary. This standard approach is used by, e.g., `git` or `docker`, both of
which face similar problems for connecting to remote services.

A credential helper can be any program that can read values from standard input.
We use the first argument in the command line to differentiate the kind of
command to execute. For now, there will only be one valid value:
- `get`: Retrieves credentials. The payload provided as standard input to the
  credential helper will be a `json` object with a single key `url` containing
  the url (including the protocol) to get credentials for (`{"url": "..."}`).

  Upon success, the credential helper must provide a `json` object adhering to
  the following spec in standard output:

  ```json
  {
    "headers": {
      "name1": ["value1", "value2", "..."],
      "name2": ["value3", "..."]
    }
  }
  ```

  Upon failure, the credential helper may provide a human-readable error
  message in its standard error that Bazel will print.

Whenever connecting to a remote server, Bazel will execute the credential helper
to retrieve credentials. Since there are usually many connections to the same
service (e.g., in the form of RPCs against a `Remote Cache`), Bazel may
(optionally) cache credentials to avoid running the credential helper binary too
often.

Since Bazel wouldn't know which credential helper to call, users must
explicitly configure Bazel by passing `--credential_helper`, which can either
be an absolute path, or a string. In case the value is a string (`foo`), Bazel
will look for an executable with the name `foo-bazel-credential-helper` on the
path. Since the credential helper must be available very early in the build
(i.e., during the loading phase for fetching external repositories), it cannot
be a label to a target.


# Backward-compatibility

This feature is fully backward compatible. Users need to opt-in to using a
credentials helper by passing `--credential_helper` to Bazel.


# Misc

This proposal was inspired by Docker's protocol for credential helpers,
available at https://github.com/docker/docker-credential-helpers.

There's a prototype implementing a very similar protocol available at
https://github.com/EngFlow/bazel/pull/2.
