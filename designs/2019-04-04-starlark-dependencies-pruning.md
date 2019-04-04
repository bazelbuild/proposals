---
created: 2019-04-04
last updated: 2019-04-04
status: To be reviewed
reviewers:
  - ?
title: Dependencies pruning in starlark
authors:
  - emmanuel-p
---


# Abstract

The goal of this proposal is to provide a way to implement dependencies pruning
in starlark.


# Background

In order to speed up builds (and mostly re-builds) a common strategy is to prune
dependencies: after an action is run, it is possible to tell which inputs were
not used. That way, bazel can stop watching for any change in those files, as
they cannot affect the outcome of the action.

Bazel provides such a mechanism, but it is only implemented in the native rules
(at least C++).

The goal is to expose a similar mechanism to starlark.


# Proposal

## API changes

This proposal adds the following attribute to `action.run()` method:

*   **`unused_inputs_list`**: File containing list of inputs unused by the
    action.

    The content of this file (generally one of the outputs of the action)
    contains the list of inputs that were not used during the whole action
    execution. Any change in those files must not affect in any way the outputs
    of the action.

Note: the negative form was used (`unused_inputs_list` instead of
`used_inputs_list`) in order to avoid mistakenly forgetting some inputs. Being
exhaustive in the list of inputs is particularly hard and brittle: tool author
must not forget the action binary itself, random added resources, etc...

## Implementation

Implementation is straightforward:

1.  After the action has run, the file specified in `unused_inputs_list` is read
    and corresponding inputs are removed from the inputs of the action.
2.  If the action has to run again, inputs removed in 1. are restored (through
    the input-discovery mechanism) in order to make them available again to the
    action.


# Backward-compatibility

Non-issue: a new attribute is added.

