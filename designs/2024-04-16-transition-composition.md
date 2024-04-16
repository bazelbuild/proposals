---
created: 2024-04-16
last updated: 2024-04-16
status: review
reviewers:
  - gregestren
title: Starlark Transition Composition
authors:
  - katre
discussion thread: https://github.com/bazelbuild/bazel/discussions/22019
---

# Overview

Bazel's facility for [configuration
transitions](https://bazel.build/extending/config#user-defined-transitions) are
widely used in rules today when rules or their dependencies need to change the
configuration. Bazel has functionality to compose multiple native transitions
into one (see
[`ComposingTransition`](https://cs.opensource.google/bazel/bazel/+/master:src/main/java/com/google/devtools/build/lib/analysis/config/transitions/ComposingTransition.java),
but that hasn't yet been exposed to Starlark for custom transitions.

# Composition Semantics

The existing native composition takes two transitions (either of which may
already be a composed transition), and has fairly basic semantics:

1. If both transitions are 1:1 ("patch") transitions, then the composed
   transition is also 1:1.
2. If either transition is a 1:2+ ("split") transition, then the composed
   transition is also 1:2+.
   1. Composing two split transitions is not allowed, due to the large number of
      generated configurations.
3. The actual configuration change applies the first transition, and then
   applies the second transition to each split of the result.

These semantics will be kept in the Starlark version of transition composition.

# Proposal

The existing Starlark transitions will be extended to allow composition using
the `+` operator (using the existing
[`HasBinary`](https://cs.opensource.google/bazel/bazel/+/master:src/main/java/net/starlark/java/eval/HasBinary.java)
interface):

```py
transition1 = transition(...)
transition2 = transition(...)
composed = transition1 + transition2
```

In all cases, the first operand must be a transition (either a Starlark
transition, or a native transition exposed to Starlark).

The second transition may be:

1. Another Starlark or native transition
   1. Including the [`config.exec`](https://bazel.build/rules/lib/toplevel/config#exec) form of the execution transition.
2. The string "exec", for the execution transition on the default exec group.
3. The string "target", for a target transition.
   1. This is effectively a no-op but is allowed for symmetry with the [`cfg`
      parameter of `attr.label`](https://bazel.build/rules/lib/toplevel/attr#label.cfg)

The composed transitions can be used like other Starlark transitions, including
on rules and attributes, although the usual restriction that a 1:2+ transition
is not allowed as an incoming rule transition will be checked.
