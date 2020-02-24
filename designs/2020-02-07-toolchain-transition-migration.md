---
created: 2020-02-07
last updated: 2020-02-10
status: Under Review
reviewers:
  - gregce
  - cparsons (for Starlark API implications)
title: Toolchain Transition Migration
authors:
  - katre
discussion thread: https://groups.google.com/d/msg/bazel-dev/g23uuZ7zSTM/X0qoXh8XBAAJ
---

# Migrating to Toolchain Transitions

For several years, rule authors have been asking for more nuance in how targets
depend on their toolchains via toolchain resolution:
-  Some authors want their dependencies in the execution configuration (of the
   parent target)
-  Some want the target configuration (again, of the parent target)
-  Some want both, depending on what part of the toolchain is being built:
   https://github.com/bazelbuild/bazel/issues/4067

This is addressed in the proposal for [Toolchain
Transitions](https://github.com/bazelbuild/proposals/blob/master/designs/2019-02-12-toolchain-transitions.md),
which is currently [being
implemented](https://github.com/bazelbuild/bazel/issues/10523). However, the
migration plan in that proposal is out of date and won't easily work: too many
different rule sets are using toolchains, and there's no way to coordinate a
"flag day" flip of all of them at once. Instead, it is possible to have a new
migration plan for the toolchain transition, that will allow rule authors to
independently migrate their rules (with some additional work). The plan is to
add a new attribute during rule creation, so that individual rules can declare
whether or not they are ready for the toolchain transition. There will also be
an incompatible flag that overrides that attribute, so that we can assess the
overall progress towards migration and begin removing the attribute.

This migration will take at least one, and probably two, major Bazel releases
(so, realistically, three to six months), plus an additional release for
cleanup.

# Migration Steps

Here is the proposed timeline:
1.  Finish implementing the toolchain transition, along with sufficient tests to
    be comfortable with it.
1.  Add a new attribute, `incompatible_use_toolchain_transition`, on both
    Starlark's `rule` function and the native `RuleClass.Builder`.
    1.  If the rule attribute is true, the rule will depend on toolchains via
        the toolchain transition.
1.  Add a new incompatible flag, `--incompatible_override_toolchain_transition`.
    1.  If the flag is set to true, the rule attribute is ignored and all rules
        use the toolchain transition.
1.  Wait for the above to be released.
1.  Rule authors migrate their rules to use toolchain transitions.
    1.  See details below.
    1.  Create tracking issues against rule sets to alert them of the work
        needed.
1.  When all rules covered in Bazel's CI are ready, flip the
    `--incompatible_override_toolchain_transition` flag to true.
1.  Wait for release.
1.  Remove flag and attribute functionality, leave the
    `incompatible_use_toolchain_transition` rule attribute as a no-op.
1.  Rule authors remove use of `incompatible_use_toolchain_transition`.
1.  Wait for release.
1.  Remove the rule attribute entirely.

## Migrating Starlark Rules

Starlark rules can migrate with the following steps:
1.  Add `cfg = "exec"` or `cfg = "target"` to all label attributes of your
    **toolchain** rule. (ie, to `go_toolchain` or `scala_toolchain`)
    1.  Use the execution transition for dependencies that will execute as part
        of the target build.
    1.  Use the target transition for dependencies that should be in the same
        configuration as the target build.
        1.  Technically `cfg = "target"` is not needed, as it is the default,
            but it is useful to document this clearly.
    1.  These changes are permanent.
    1.  One possible migration path is the following:
        1.  Add `cfg = "exec"` to **every** dependency. This is a no-op since
            currently all dependencies are in the `host` configuration.
        2.  Enable the `--incompatible_override_toolcgain_transition` flag and
            test builds. The change should be minor, given the similarities
            between `exec` and `host` dependencies, but this will reveal any
            cases where this isn't true. Typically, this has been seen where the
            depenedency itself has further dependencies, which are now free to
            transition to unforseen configurations.
        3.  Add `cfg = "target"` to dependencies where this makes sense. This
            change can be done individually for each dependency and
            tested/released separately, if desired, to ensure build correctness.
1.  Add `incompatible_use_toolchain_transition = True` to **every** rule that
    uses a toolchain.
    1.  That is, every rule that sets `toolchains = ["//your/rule:toolchain_type"]`.
    1.  This change will be reverted when the global migration is complete.
1.  Test and release your rules as normal.
1.  When the global toolchain transition migration is complete, remove the
    `incompatible_use_toolchain_transition` attribute from your rules.

## Migrating Native Rules

Native rules migrate similarly to Starlark rules:
1.  Add `cfg(ExecutionTransitionFactory.create())` to the correct attributes
    of your **toolchain** rule. (ie, to `cc_toolchain` or `java_toolchain`)
    1.  Use the execution transition for dependencies that will execute as part
        of the target build.
    1.  Dependencies that should be in the same configuration as the target
        build do not need anything.
    1.  These changes are permanent.
1.  Add `builder.useToolchainTransition(true)` to **every** rule that uses a
    toolchain.
    1.  That is, every rule that calls `builder.addRequiredToolchains()`.
       1.  This is inherited by child rules and so can be set at a high level.
    1.  This change will be reverted when the global migration is complete.
1.  Test your rules as normal, and wait for a Bazel release.
1.  As part of the global toolchain transition migration, your
    `useToolchainTransition` calls will be removed.

