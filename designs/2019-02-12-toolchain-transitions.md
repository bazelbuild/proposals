---
created: 2019-02-12
last updated: 2019-02-13
status: In Review
reviewers:
  - gregce
title: Toolchain Transitions
authors:
  - katre
discussion thread: https://groups.google.com/forum/#!topic/bazel-dev/5osWxhoF0Fk
---

# Abstract

Toolchains need access to two separate platforms: the target platform of the
original target that requires a toolchain, and the execution platform of the
original target. This is because a toolchain needs to select the proper tools to
use in the build (and which will be run on the execution platform), but it also
may want to build additional code that can be linked into the target (and which,
then, needs to be built for the eventual target platform).

See the related design for [Execution Transitions](2019-02-12-execution-transitions.md).

# Proposal

Consider this simple build graph, with non-essential details elided:


```py
foo_library(name = "example") # foo_library requires //foo:toolchain_type
toolchain_type(name = "toolchain_type")
toolchain(
    name = "foo_toolchain",
    toolchain_type = "//foo:toolchain_type",
    toolchain = "//foo:foo_toolchain_impl",
)
foo_toolchain(
    name = "foo_toolchain_impl",
    standard_library = "//foo:standard_library",
    compiler = "//foo:compiler",
)
cc_library(name = "standard_library", ...)
cc_binary(name = "compiler", ...)
```

In this build graph, we have several targets which require different configurations:

1.  `example`: The top-level target has the initial configuration (C1)
1.  `foo_toolchain`: This is loaded indirectly via toolchain resolution. The
    target `example` never has a direct dependency on this target, and it should
    not change its behavior based on the configuration.
1.  `foo_toolchain_impl`: This target is a direct dependency of `example` (via
    toolchain resolution). The configuration of this target is currently not
    well defined.
1.  `standard_library`: Because the `foo_library` rule intends to directly link
    this library into the output of `example`, it also needs to be in the
    initial configuration (C1).
1.  `compiler` - Because the `foo_library` rule intends to use this as the
    executable for an action while building the output of `example`, it needs to
    be in a different configuration (C2). C2 is the result of applying the
    [Execution Transition](2019-02-12-execution-transitions.md) to C1 (which is
    the original configuration used by `example`).

The simplest implementation is to allow the toolchain implementation (the target
`foo_toolchain_impl` in this example) to use almost the same configuration as
the original target (`example`). The differences are in a few internal-only
flags used to store required data.

First, the original execution platform's label must be stored so that it is not
cleared during toolchain resolution for the toolchain itself. This may happen
using a marker to prevent the clearing, or by using a second internal-only flag.
Then, when the toolchain-&gt;tool dependency (`foo_toolchain_impl` -&gt; `compiler`)
is analyzed, the execution transition can change the target platform to be the
correct execution platform, using the previously defined execution transition.

Secondly, the toolchain-&gt;library dependency (`foo_toolchain_impl` -&gt;
`standard_library`) can use the identity transition, to inherit the original
target's configuration. This avoids the need to transition to a special
configuration, and then back to the original target configuration.

The new transition will not be exposed to Starlark rules (or to native rules,
outside of the toolchain resolution system).

# Migration

After the new transition, existing toolchains will need to be updated to use the
execution transition on dependencies which are expected to work on the execution
platform. Native rules can be updated immediately, but Starlark rules will
require a migration period.

1.  Add a new incompatible flag, `--incompatible_enable_toolchain_transition`.
1.  Add `exec` as a synonym for `host` for the `cfg` parameter to
    [attribute declaration](https://source.bazel.build/bazel/+/master:src/main/java/com/google/devtools/build/lib/analysis/skylark/SkylarkAttr.java;drc=a65ce6d547ee36f7d1fbeb81105d22849193371f;bpt=1;l=269).
1.  Existing toolchains add `cfg = "exec"` to all dependencies. This is a no-op
    change.
1.  If the incompatible flag is set, enable the new toolchain transition for
    target-&gt;toolchain dependencies.
1.  Update [Starlark's attribute machinery](https://source.bazel.build/bazel/+/master:src/main/java/com/google/devtools/build/lib/analysis/skylark/SkylarkAttr.java;drc=a65ce6d547ee36f7d1fbeb81105d22849193371f;bpt=1;l=269)
    to use the execution transition for `exec` instead of the host transition.

There would need to be pauses for Bazel releases between steps 2 and 3, and 3
and 4. Steps 4 and 5 can happen in the same release.

Even with the incompatible flag, a project which depends on multiple sets of
rules may have compatibility problems if rule set A has updated and is ready for
the flag, but rule set B has not. In this case, projects should be very careful
about updating rule dependencies in tandem, and the Bazel Configurability team
needs to be very proactive about updating rules in the smallest timeframe
possible.

