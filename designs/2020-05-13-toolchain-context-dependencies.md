---
created: 2020-05-13
last updated: 2020-05-19
status: Approved
reviewers:
  - gregce
  - janakdr
title: Passing Toolchain Context across Dependencies
authors:
  - katre
discussion thread: https://groups.google.com/d/msg/bazel-dev/5rVPa23XuOY/vLPFmHkNCQAJ
---

# Abstract

Consider a configured target (CT1) which depends on another target (CT2). We
have identified two use cases so far where a configured target would need to
know details of the toolchain context of the parent target that depends on it.

1. If CT2 is a toolchain implementation (`cc_toolchain`, `java_toolchain`, etc),
   it needs to know the specific execution platform of CT1, in order to
   guarantee that it can generate tools that will execute on CT1’s execution
   platform. See [Toolchain Transitions](2019-02-12-toolchain-transitions.md)
   for details.
    1. In the case of execution groups, CT2 needs to know the execution platform
       of the specific execution group that required that toolchain type: if
       multiple execution groups require the same toolchain type and have
       different execution platforms, this could require multiple instances of
       CT2.
1. If CT2 is a `config_setting` target, it would be useful to be able to match
   on details of the toolchain in use by CT1. See [Incompatible Target
   Skipping](https://docs.google.com/document/d/12n5QNHmFSkuh5yAbdEex64ot4hRgR-moL1zRimU7wHQ/edit?ts=5dfbe2fe#heading=h.5mcn15i0e1ch)
   for details.
    1. Although the `config_setting` will have the same configuration as the
       parent target, because it will not have the same required toolchain
       types, toolchain resolution may happen entirely differently.
    1. That proposal also needs to be updated to handle execution groups, but
       the direction should be straightforward.

For the toolchain transitions case, the [previous
proposal](2019-02-12-toolchain-transitions.md#implementation) was to have a
toolchain-specific configuration transition which included the execution
platform. This would then require the toolchain implementation to change the new
configuration back to the original target configuration (by removing the extra
information), or change the new configuration to an execution configuration (by
replacing the target platform with the saved execution platform). However, this
approach requires extra overhead, more configurations in memory, and complicated
cleanup afterward. A similar approach might be used to enable the
`config_setting` case, but would also be ad-hoc and require complicated
configuration machinery.


# Proposal

Instead of inflating the configuration to use as a data channel, we can instead
create a subclass of the existing
[ConfiguredTargetKey](https://cs.opensource.google/bazel/bazel/+/master:src/main/java/com/google/devtools/build/lib/skyframe/ConfiguredTargetKey.java).
Currently, ConfiguredTargetKey contains a Label and a
BuildConfigurationValue.Key. By adding a subclass with a
[ToolchainContextKey](https://cs.opensource.google/bazel/bazel/+/master:src/main/java/com/google/devtools/build/lib/skyframe/ToolchainContextKey.java),
the toolchain implementation configured targets (or `config_setting`
implementation) will have direct access to the parent target’s toolchain
context. Because the toolchain context will already have been loaded from CT1,
when CT2 requests it there will be no need for a Skyframe restart, as the value
will already be present.

The ToolchainContextKey currently contains:
*   A BuildConfigurationValue.Key pointing to the configuration
*   The set of required toolchain types, as Labels
*   The set of execution constraints added by the target, also as Labels
*   A boolean indicating whether the configuration fragments need to be checked,
    as part of configuration trimming.

The ConfiguredTargetFunction would be responsible for checking that this exists
and then making it available to be used. In the case of
[ConfigSetting](https://cs.opensource.google/bazel/bazel/+/master:src/main/java/com/google/devtools/build/lib/rules/config/ConfigSetting.java),
we would need to make this available to the implementation via the
[RuleContext](https://cs.opensource.google/bazel/bazel/+/master:src/main/java/com/google/devtools/build/lib/analysis/RuleContext.java).
We would probably want to allow rules to indicate in the RuleClass.Builder that
they want this, and vet it to ensure that only allowed core rules (like
`config_setting`) can reach the information. We would not want to expose this to
Starlark rule implementations without further thought and design.

For toolchain implementations, instead of making the parent toolchain context
available to the rule implementation, we would just use the execution platform
data to set the configuration for dependencies of the toolchain:
*   For `cfg = "target"` dependencies, we would re-use the existing target
    configuration (which is the same for CT1 and CT2).
*   For `cfg = "exec"` dependencies in the default execution group, we would use
    the execution platform from the toolchain collection to create the
    ExecutionTransitionFactory.
*   Since toolchain implementations aren’t expected to create actions, there is
    no reason to expect toolchain implementations to use non-default execution
    groups. If any did, they would use the standard toolchain resolution system
    and ignore the parent toolchain context information.


## Skyframe Impact

Re-visiting the example from above, imagine a configured target (CT1)
that requires one toolchain type (TT), this will be resolved to a single
toolchain implementation (CT2). With the toolchain context added to the
ConfiguredTargetKey, there will still be only a single ConfiguredTargetKey for
each toolchain implementation: the toolchain context depends on the required
toolchain types and the configuration.

Since most rules require only a single toolchain type, and very few targets
specify execution constraints, we will have a single ConfiguredTargetKey for
each configuration and toolchain implementation, which is the same situation we
have today.

It is important to note that there are rarely direct dependencies on toolchain
implementations: this may happen in some implementation tests but is not
expected during normal builds.

Without this feature, we will be creating new configurations to convey the same
information, which will increase the number of BuildConfigurationValue.Key
instances as well as the number of ConfiguredTargetKey instances.


# Compatibility

This proposal doesn’t directly affect any functionality which would require
backwards compatibility guarantees. Individual features which it unlocks, such
as the improved toolchain transitions, would still need their own backwards
compatibility checks. See [Toolchain Transitions
Migration](2020-02-07-toolchain-transition-migration.md) for details.

