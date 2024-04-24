---
created: 2024-04-16
last updated: 2024-04-24
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
already be a composed transition), and has fairly basic semantics.

1. The first transition is applied to the configuration.
   1. This will produce either a single result configuation (for a 1:1
      transition), or several (for a 1:2+ transition).
2. The second transition is then applied to each result configuration.
   1. If the second transition has any `input` flags, those are read from the
      first result. The second transition may then decide based on its logic
      what the `output` flags are.
   2. If the first transition was a 1:2+ transition, the second transition must
      be a 1:1 transition.
   3. If the first transition was a 1:1 transition, the second transition can be
      a 1:1 transition or a 1:2+ transition.
3. All result configurations from both transition applications are then
   collected and used as normal. Rules see a single (composed) transition, and
   cannot tell how many transitions were actually applied.
4. Composed transitions can themselves be composed, with the same limitation
   that there is only a single 1:2+ transition allowed anywhere in the chain.

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
