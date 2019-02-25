---
created: 2019-02-12
last updated: 2019-02-13
status: In Review
reviewers:
  - gregce
title: Execution Transitions
authors:
  - katre
discussion thread: https://groups.google.com/forum/#!topic/bazel-dev/5osWxhoF0Fk
---

# Abstract

Historically, Bazel has had a single configuration used for targets that need to
be executed on the host. This is called the
[**host configuration**](https://docs.bazel.build/versions/master/guide.html#configurations),
and is created when Bazel begins analysis. Every dependency that is built for
the host platform shares the same host configuration.

This fails to work properly when actions can be executed on platforms different
from the host platform (for example, using remote execution). In these cases,
Bazel needs a transition to an **exec configuration**, and not every target built
to be used on an execution platform should have the same configuration.

See the related design for [Toolchain Transitions](2019-02-12-toolchain-transitions.md).

# Proposal

In order to properly transition to a configuration where the previous execution
platform is the current target platform, the execution platform must be
available to the transition. Note that there is no flag to directly set the
execution platform (the closest flag available is
[`--extra_execution_platforms`](https://source.bazel.build/bazel/+/master:src/main/java/com/google/devtools/build/lib/analysis/PlatformOptions.java;l=56?q=extra_execution_platforms),
which adds available execution platforms, but does not select one).

This new transition needs to be usable on any attribute, regardless of whether
the rule involved is providing a toolchain itself. As an example, the `genrule`
rule has an attribute named `tools`, which should be built in the context of the
execution platform, instead of the current host platform.

Because of this, the execution transition will be a single-instanced
implementation of
[`PatchTransition`](https://source.bazel.build/bazel/+/master:src/main/java/com/google/devtools/build/lib/analysis/config/transitions/PatchTransition.java),
similarly to how
[`HostTransition.INSTANCE`](https://source.bazel.build/bazel/+/master:src/main/java/com/google/devtools/build/lib/analysis/config/HostTransition.java)
is implemented. This will require special handling in
[`DependencyResolver`](https://source.bazel.build/bazel/+/master:src/main/java/com/google/devtools/build/lib/analysis/DependencyResolver.java)
to pass in the correct execution platform.

For native rules, accessing the new transition will involve changing transitions
from `.cfg(HostTransitions.INSTANCE)` to `.cfg(ExecTransition.INSTANCE)`.

For Starlark rules, a new argument to the existing `cfg` parameter will be
added, allowing rules to declare attributes with `cfg = "exec"` instead of `cfg
= "host"`.

## Implementation

To implement the new transition, during toolchain resolution, the selected
execution platform's label will be added to an internal-only flag on
[`PlatformOptions`](https://source.bazel.build/bazel/+/master:src/main/java/com/google/devtools/build/lib/analysis/PlatformOptions.java).
If no execution platform is selected, this internal-only flag will be cleared.
Then, when the new execution transition is used, the following steps will take
place:

1.  For all fragments, the
    [FragmentOptions.getHost](https://source.bazel.build/bazel/+/master:src/main/java/com/google/devtools/build/lib/analysis/config/FragmentOptions.java;l=61)
    method will be called.
1.  The previous execution platform's label will be copied from the
    internal-only flag to the `--platforms` flag. This will replace any value of
    `--platforms` set by `getHost()`.
1.  The internal-only flag will be cleared.
1.  After other transitions have been applied, the (in process)
    [flag migration](https://docs.google.com/document/d/1Vg_tPgiZbSrvXcJ403vZVAGlsWhH9BUDrAxMOYnO0Ls/edit)
    process will then update related legacy flags, such as `--cpu` and
    `--crosstool_top`.

The `getHost` method will be called to allow for flags to be set to reasonable
default values for non-target builds. For example, users expect that when they
set both
[`--compilation_mode`](https://source.bazel.build/bazel/+/master:src/main/java/com/google/devtools/build/lib/analysis/config/BuildConfiguration.java;l=477)
and
[`--host_compilation_mode`](https://source.bazel.build/bazel/+/master:src/main/java/com/google/devtools/build/lib/analysis/config/BuildConfiguration.java;l=488),
any tools (which are built for the execution platform) will use the latter flag.
This allows, for example, building output targets with debugging code, while
using optimized builds for tools that are invoked during the build.

Since the selected execution platform is strictly a function of the required
toolchains, the target platform, and the available execution platforms and
toolchains, there are no worries that this will add any non-determinism to the
build. However, the new data added (the selected execution platform label) will
cause the configuration key to change, as the configuration itself is different.
This could be avoided if there were an out-of-band way to pass information into
a transition, or a way to use transition factories instead of static transitions
when defining attribute configurations.

### Other implementation possibilities

Instead of using an internal-only flag which is set unconditionally during
toolchain resolution, `DependencyResolver` could be updated to detect that the
new `Executiontransition.INSTANCE` transition is being used, and directly inject
the execution platform at that point. This would simplify the configuration
changes, at the cost of making the implementation less flexible. In particular,
it would require more work to identify an execution transition that is part of a
composed transition.

# Backwards Compatibility

As this is a new transition, with a new access system, there are no backwards
compatibility worries. See the migration plan for
[Toolchain Transitions](2019-02-12-toolchain-transitions.md),
however, for some caveats for Starlark rules.

Over time, we should expect to replace existing uses of the host transition
(both native and in Starlark) with the exec transition. This migration can be
handled usecase-by-usecase, and will be addressed separately.

