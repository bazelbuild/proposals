---
created: 2018-10-22
last updated: 2019-06-06
status: Approved
reviewers:
  - gregce@google.com
  - schmitt@google.com
title: Auto-configured Host Platform
authors:
  - katre
discussion thread: https://groups.google.com/forum/#!msg/bazel-dev/s-YHxjyu4IE/CJ5nMejnBQAJ
---

# Abstract

Bazel's current system for determining the host platform is insufficiently robust and incorrectly couples the platform rule definition to the configuration. See the documentation on [Bazel platforms](https://docs.bazel.build/versions/master/platforms.html) for background details on platforms.

# Background

Currently, the host and target platforms are hard-coded (as `@bazel_tools//platforms:host_platform` and `@bazel_tools//platforms:target_platform` respectively), with special logic in the [platform rule definition](https://source.bazel.build/bazel/+/master:src/main/java/com/google/devtools/build/lib/rules/platform/Platform.java) to set the CPU and OS constraints based on the value of the `--cpu` and `--host_cpu` flags.

This is undesirable for a few reasons:
- Philosophically, the platform rule shouldn't need to read the configuration: it exists to help define the configuration
- Practically, this is a source of errors: after a host transition the meaning of `@bazel_tools//platforms:target_platform` changes, leading to errors in analysis.

The correct solution is for Bazel to auto-configure the host platform at startup, and then use that platform in the configuration. The host platform should then be immutable in the face of configuration changes. In addition, the `--platforms` flag should be changed to default to the host platform, if not set.

# Proposal

Bazel needs a place to generate the auto-configured platform for the host system. To handle this, a new external repository can be created, which is responsible for detection and autoconfiguration. This repository will write a BUILD file with a single platform target, referred to as `@local_config_platform//:host`.

## Implementation 1

In this version, the autoconfiguration and external repository are implemented in Starlark and shipped as part of the existing `@bazel_tools` repository (similar to how the [C++ toolchain is autoconfigured](https://source.bazel.build/bazel/+/master:tools/cpp/cc_configure.bzl)). This repository will check the system CPU and OS and use the standard constraint values defined in `@bazel_tools//platforms` to write the auto-detected host platform.

## Implementation 2

Instead of a Starlark repository, a new type of external repository rule would be written in native code. This will allow the repository to access the existing [CPU](https://source.bazel.build/bazel/+/master:src/main/java/com/google/devtools/build/lib/util/CPU.java) and [OS](https://source.bazel.build/bazel/+/master:src/main/java/com/google/devtools/build/lib/util/OS.java) autodetection. This system would also write a BUILD file with a single platform definition, using the standard set of constraint values available.

# Backward-compatibility

This change removes some functionality: the current behavior uses the `--cpu` and `--host_cpu` flags to set the target and host platform, whereas the new behavior will ignore those and only use the actual host CPU and OS for the auto-detected platform. This feature will be replaced by the ability to set the platform flags based on other flag values, see `Old Flags to Platform Migration`. As a stopgap if the flag migration scheme is not ready in time, the new autoconfiguration can also check the value of the existing flags.

This change will be released in several steps, with several pauses between releases in accordance with the Incompatible Change Policy:
1. Add the new repository, but do not directly use the new label anywhere.
  - This is finished.
2. Add an incompatibility flag, `--incompatible_auto_configure_host_platform`, defaulting to `false`. When this flag is enabled, there will be the following effects:
  - The default values of `--host_platform` and `--platforms` will be `@local_config_platforms//:host`.
  - The `platform` rule will ignore the current `host_platform` and `target_platform` attributes, thus stopping the use of `--cpu` and `--host_cpu` to configure these.
3. These changes will then be available for a release, in order for users to test and verify.
4. Next, the incompatible flag will be flipped to `true` by default.
5. Wait a release to be sure there are no issues.
6. At this point, the incompatible flag, the legacy platforms in `@bazel_tools//platforms`, and the legacy cpu detection code in the Platform rule implementation can all be removed.

In addition, this only affects the default values of the existing `--host_platform` and `--platforms` flags. Any users who wish to opt out of platform auto-detection can instead write platform targets representing their host and target and then set the relevant flag.

If any users are currently expecting `@bazel_tools//:target_platform` or
`@bazel_tools//:host_platform` to change in response the the `--cpu` and
`--host_cpu` flags, they can be advised to set up platform mappings, or to use
statically defined platforms, to accomplish the same behavior.
