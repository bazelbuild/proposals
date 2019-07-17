---
created: 2019-06-25
last updated: 2019-06-25
status: In Review
title: Platforms and Strategies
authors:
  - katre
reviewers:
  - philwo (lead)
  - ishikhman
  - agoulti
discussion thread: https://groups.google.com/d/msg/bazel-dev/NjcyelfHvyA/n9Buk1mODQAJ
---

# Background

Bazel currently has two different systems that select where a build action is
executed: [execution
platforms](https://docs.bazel.build/versions/master/platforms.html) and
[execution
strategies](https://docs.bazel.build/versions/master/user-manual.html#strategy-options).
Historically, these have not communicated: this leads to the situation where
they can mismatch, for example by selecting a local-only toolchain, but using
the "remote" execution strategy, or the opposite, where remote execution
platforms and toolchains are selected, but the "remote" strategy is not enabled.
Both of these situations lead to failed builds and confused users.

With the recent addition of [automatic execution strategy selection in Bazel
0.27](https://blog.bazel.build/2019/06/19/list-strategy.html), we now have a
chance to update this selection, and make both systems aware of each other.

# Proposal

This proposal calls for changes to both toolchain resolution, and to execution
strategy selection, in order for both systems to be aware of each other.

To facilitate these changes, a new attribute will be added to the `platform`
rule.

## Platform Rule

A new attribute, `spawn_strategy`, will be added to the `platform` rule, which
will take a list of strings. The contents of the list will be compared to the
names of the existing execution strategies (i.e., the allowed values for the
[`--spawn_strategy`](https://docs.bazel.build/versions/master/user-manual.html#flag--spawn_strategy)
flag), but with no explicit checking for allowed values.

Example:
```py
platform(
    name = "local",
    spawn_strategies = [
      "local",
      "sandbox",
      "worker",
    ],
)

platform(
    name = "remote",
    spawn_strategies = [
      "remote",
    ],
)
```

In the [PlatformInfo
provider](https://docs.bazel.build/versions/master/skylark/lib/PlatformInfo.html),
these will be converted to a `Set<String>`, indicating that order of the
strategies does not matter.

Open questions:
-  Should the strategies be lower-cased? `--spawn_strategy` does not appear to
   do any special processing.
-  Is there a better name for the attribute? I prefer `execution_strategies`,
   but `spawn_strategies` more closely matches the existing `--spawn_strategy`
   flag, and so may be clearer.

Once the new attribute is available, the existing default host platforms will be
updated to add the following strategies:
-  worker
-  sandboxed
-  local

When a platform is not being used as an execution platform (for example, if a
platform is used as a target platform), the new attribute will be unused.

### Alternatives Considered

Instead of adding a new attribute which is only useful for execution platforms,
a new flag could be added: `--platform_spawn_strategy`. Any execution platform
that is not also the host platform would use these values for the following
stages. The host platform would use the hard-coded defaults from above.

## Toolchain Resolution

During [toolchain
resolution](https://docs.bazel.build/versions/master/toolchains.html#toolchain-resolution),
Bazel first determines the correct set of toolchain implementations to use for
each available execution platform. Then, it selects the highest priority
execution platform, determined by the order execution platforms are defined,
first via the `--extra_execution_platforms` flag and then by the order of calls
to `register_execution_platforms` in WORKSPACE.

To consider execution strategy, instead of using ordering to determine priority,
toolchain resolution will instead use the order of available execution
strategies (as given by the `--spawn_strategy` flag). The selection of execution
platform will then follow this logic:

1. For each execution strategy in order:
   1. Find the first execution strategy that claims support for the strategy, as
      indicated by the new `spawn_strategies` attribute.
   2. Verify is this strategy and execution platform are compatible with the
      target under consideration by checking the (non-configurable) `tags`
      attribute (as the execution strategy selection does now).
2. If all execution strategies have been tried and no available execution
   platform exists, fail with an error describing the situation.

In order for legacy platforms to continue working, a platform that does not set
the `spawn_strategies` attribute will be considered to support every execution
strategy.

At this point target configuration can continue as it currently does.

## Execution Strategy Selection

When an action is executed, each available strategy is checked in order to see
if it supports the action (based on execution requirements from the tags and
other sources, such as the toolchain).

When considering the execution platform, the selection of spawn strategy only
needs to add a check that the selected execution platform (which is currently
available as
[Spawn.getExecutionPlatform](https://source.bazel.build/bazel/+/master:src/main/java/com/google/devtools/build/lib/actions/BaseSpawn.java;l=155?q=BaseSpawn)),
so no new interfaces need to be added.

It will still be possible to end up in a situation where the execution strategy
and the execution platform do not match: toolchain resolution will not be able
to verify toolchain-based execution requirements, and the existence of the
[`--strategy
mnemonic=strategy`](https://docs.bazel.build/versions/master/user-manual.html#flag--strategy)
and
[`--strategy_regexp`](https://docs.bazel.build/versions/master/user-manual.html#flag--strategy_regexp)
flags mean that the strategy selection can still be overridden. In these cases,
we need to make sure the error messages from action execution indicate both the
selected execution strategy and the selected execution platform, so that users
can diagnose and fix these types of mis-configuration.

# Backwards Compatibility

The changes in this proposal will not take effect unless the new attribute on
`platform` is specified. Therefore, any existing users will see no changes until
the platforms they use are updated.

# Documentation

As part of this work, documentation and tutorials also need to be added, with
special attention to examples that cause errors or misconfiguration, and how to
recover.
