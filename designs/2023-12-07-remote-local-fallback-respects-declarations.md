---
created: 2023-12-07
last updated: 2023-12-08
status: Draft
reviewers:
  - TBD
title: `--remote_local_fallback` Respects Strategy Declarations
authors:
  - Silic0nS0ldier
---

# Abstract

The flag `--remote_local_fallback` makes it possible for remotely executed action spawns that fail to fallback to the `local` spawn strategy.
This proposal seeks to treat spawn retries identically to actions tagged with `no-remote`.

# Background

Falling back to the local environment is not always safe, and may be contrary to spawn strategy configuration.

e.g.

```ini
# Denotes actions with the mnemonic `FOO` must always be spawn with the `remote` strategy
build --strategy=Foo=remote
```

Requiring that an action be only spawned remotely may reflect a technical limitation.
Such as the remote offering a Windows environment while Bazel itself is running on a macOS machine.
The Windows binaries cannot be reasonably expected to work on macOS, and compatibility layers to support such a scenario are unlikely to provide identical behaviour.

Even in a context where remote and local environments are identical, `local` may not represent the optimal spawn strategy.
* `worker` may offer better performance, especially if several remotable actions fail (e.g. connectivity loss).
* `sandboxed` may provide an environment closer to that of the remote.

## Related Issues

* [remote: remove local fallback for remote execution](https://github.com/bazelbuild/bazel/issues/7202)
* [Make --remote_local_fallback honor --spawn_strategy](https://github.com/bazelbuild/bazel/issues/15519)

# Proposal

A new flag `--incompatible_strict_remote_local_fallback` will be introduced which when flipped makes `--remote_local_fallback` rerun spawn strategy selection with `remote` ignored.

This means;
* All registered strategies along with their filters are considered (`--spawn_strategy`, `--strategy`, `--strategy_regexp`, etc).
* If a `remote` spawned action has no local fallback, no attempt to spawn locally is made and the build fails.

To avoid confusion, the `--remote_local_fallback_strategy` flag (currently marked as a deprecated no-op, ([#7480](https://github.com/bazelbuild/bazel/issues/7480))) will be removed.

# Backward-compatibility

Current behaviour is counterproductive, so the expectation is that flag guarded behaviour would become the default in the next major version of Bazel.
