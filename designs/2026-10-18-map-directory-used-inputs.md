# Used input discovery for map_directory actions

Author: Tamir Duberstein, with delegated implementation and drafting assistance from OpenAI Codex

Last major change: 2026-10-18

Status: Draft; implementation is a proof of concept, not an approved API

Lead reviewer: Not assigned

Other reviewers: Not assigned

Discussion: Not announced

Tracking issue: [bazelbuild/bazel#31849](https://github.com/bazelbuild/bazel/issues/31849)

## Problem

An action created by `ctx.actions.map_directory` may need to scan a source namespace whose files are known only when the template expands. For example, a native compiler can determine which headers an individual source file reads. The current `template_ctx.run` creates a plain `SpawnAction`: every declared input remains an invalidating dependency, including files the compiler did not read. Dividing a batch into per-object actions does not fix that behavior if each action still declares the whole namespace.

Ordinary `ctx.actions.run` already supports post-execution pruning with `unused_inputs_list`. Extending that interface literally would require each scan to write the complement of its dependencies. The motivating Linux rule has roughly 100,000 candidate source files and thousands of compilation jobs. A short used set avoids writing almost that entire namespace once per job. This observation concerns list size, not a measured total memory improvement.

## Proposed API

Add two optional, paired parameters to `template_ctx.run`:

```starlark
template_ctx.run(
    executable = compiler_scanner,
    inputs = [source, scan_spec, namespace_manifest],
    tools = [declared_tool],
    discovery_inputs = source_and_generated_files,
    outputs = [scan_result, used_inputs],
    used_inputs_list = used_inputs,
    arguments = scan_arguments,
)
```

`inputs`, the executable, and the tool/runfiles closure are mandatory. `discovery_inputs` is a list or depset of candidate Files. Both sets are available when the action executes. After successful execution, `used_inputs_list` selects the candidate Files to retain as invalidating dependencies.

The list contains one execution-root-relative path per line, using the action's path mapping when enabled. It must be a declared regular output and must exist even when empty. Empty retains the mandatory and tool inputs. Exact paths of mandatory Files are accepted harmlessly. Missing output and undeclared path are separate errors.

Candidates must be individual Files, including children supplied by expanded input directories. A whole directory artifact cannot be selectively pruned and is rejected as a candidate. A caller may keep a directory artifact mandatory, in which case it remains fully retained and its child paths must be omitted from the used list. This avoids guessing artifact ownership from path prefixes.

## Correctness and cache behavior

The action must report every candidate that can affect its outputs. This is the same trust boundary as an ordinary action reporting unused inputs. It does not infer C or C++ include semantics.

Mandatory and candidate membership are separately included in the action key. Adding or removing a candidate must invalidate the action, including adding a file that would shadow a previously selected header. Adding a mandatory input must not be hidden by restoration of a previously pruned cache entry. The caller must also retain the ordered search paths, namespace mapping, flags, and other lookup semantics in mandatory inputs or arguments. A filename membership set alone does not describe search order.

The implementation reuses `EnhancedStarlarkAction` and the existing post-execution input update, cache restoration, and remote inline-output handling. It retains the original candidate universe for rediscovery and generated-artifact resolution after an evaluator reset. Reported paths are matched against the actual Artifact namespace; mapped paths retain all candidate identities with the same mapped path. No separate cache or cache-format change is proposed.

Existing `SkyframeActionExecutor` behavior frees a pruned action's discovered input set after writing its cache entry. This proposal preserves that memory policy. A later action-graph dump can consequently show the original candidate inputs, even with `--noinclude_pruned_inputs`; execution records, the declared used-list output, and mutation behavior are needed to establish pruning. Expanded action-graph records still identify concrete producers and their owning configurations.

This is a warm action-cache optimization. A fresh client or cache miss can require the complete candidate set before execution and discovery. The proposal does not promise cross-configuration scan hits on cold clients or full native C++ input discovery beyond a declared finite candidate universe.

## Cost and evidence

The existing motivating rule's logs count batch scans, so there is no authenticated old per-object unrelated-edit time or execution count. A real Linux remote-execution consumer fixture has completed successfully with three separate scanner children, followed by its compiler, backend, and export actions, using top-level-only output downloads. The entire invocation took 444.716 seconds with a 58.87-second critical path, including establishment of the declared tool closure. This is integration evidence, not a scan performance comparison.

An unchanged build recorded zero spawns and identical output bytes. An unrelated source edit recorded zero scanner or compiler spawns; editing one C source executed only its scanner, frontend, backend, and product actions while preserving the other C and assembly products. Editing the other source or its header selected only that source's scan and compiler chain. Changing an embedded binary selected only its assembly scan and compiler action; the assembly source remained byte-identical and the object changed. These are bounded fixture results. Full-kernel measurements remain pending.

The proof of concept's integration tests exercise unchanged builds, edits to used and unused candidates, source and expanded generated Files, mandatory/tool changes, candidate and mandatory membership changes, evaluator reset, mapped and unmapped paths, empty lists, and missing/invalid reports. Serialization and the existing shadowed-action suite are also covered.

Candidate validation and existing output-directory validation still flatten and traverse the input sets per child. The nested-set representation shares the candidate DAG and only weakly caches flattened lists, but that does not eliminate transient allocation or O(candidate-count) validation. Action-key fingerprints reuse the existing path-aware nested-set fingerprint cache. Full-scale CPU and peak-memory costs remain unmeasured and are a review question, not an established benefit.

## Alternatives and API consistency

* Expose `unused_inputs_list` unchanged: it can be sound if the caller computes the complement correctly and preserves every mandatory dependency. Each scan would emit O(candidate-count minus used-count) paths instead of O(used-count). The current template API does not expose this mode, and exposing it would still require the same action-owner bridge. Output size is the concrete distinction; a runtime advantage has not yet been measured.
* Split the current batch without discovery: improves accounting but still invalidates scans on unrelated content changes.
* Use analysis-time `ctx.actions.run`: cannot declare the motivating jobs before the execution-time manifest and expanded generated Files exist.
* Add a separate dependency cache or infer includes: duplicates Bazel's cache responsibilities or fails to preserve native compiler semantics.

The new mode shares the ordinary Starlark action implementation. The initial public API is confined to the demonstrated template consumer. Whether the same pair should also be exposed on ordinary `ctx.actions.run` is an API-consistency question for maintainers; it does not require a second implementation owner.

PR [#29193](https://github.com/bazelbuild/bazel/pull/29193) discusses reading an unused list before execution for ordinary actions. It does not currently expose template discovery. Applying that approach to the motivating scanner would require a preceding dependency-producing action or a separately available list, while the scanner itself is the operation that learns the dependencies. A preparation action with the same broad invalidation would leave that preprocessing cost in place. That PR remains open and is not approval of this post-execution used-list proposal. The concerns about redundant mechanisms and per-action cost apply here too.

## Compatibility and rollout

Both parameters default to absent. Existing template actions keep their current action type and behavior. Ordinary unused-input and shadowed-action semantics are preserved. Callers opting in require the Bazel version that introduces the API; no fallback to an unsound scan is proposed.

There is no consumer version change or public binary distribution in this proposal. A locally built canonical Bazel binary is being used to validate the proof of concept. Adoption depends on maintainer review and an available supported version.

## Review questions

1. Is a used set plus an explicit candidate set the appropriate extension of the existing pruning owner?
2. Should the same API be exposed on ordinary actions at introduction, or only after a concrete caller requires it?
3. Which full-scale candidate traversal measurements are required before acceptance?

The implementation will not be submitted for integration before the required design review is complete.
