---
created: 2024-08-14
last updated: 2026-07-22
status: Draft
reviewers: []
title: Copy Action
authors:
  - Silic0nS0ldier
---

# Abstract

Copying files and directories is a common need that is underserved in Bazel. This proposal seeks to make this a builtin capability to improve performance and simplify rule development.

# Background

## Status Quo

The following actions (`ctx.actions.*`) already exist;
- `args` - An abstraction for memory-efficient argument management. May produce a file.
- `declare_directory` - Declares a directory output.
- `declare_file` - Declares a file output.
- `declare_symlink` - Declares a symlink output.
- `do_nothing` - Does nothing, only serving as an insertion point for the deprecated 'extra actions' feature.
- `expand_template` - Creates a file using a template.
- `map_directory` - An abstraction for creating actions (via `template_ctx`) based on the files within input director(y/ies).
  - `args` - Same as `ctx.actions.args`.
  - `declare_file` - Same as `ctx.actions.declare_file`.
  - `declare_subdirectory` - Declares a directory output within one of `map_directory`'s outputs.
  - `run` - Same as `ctx.actions.run`.
- `run` and `run_shell` - Runs an executable/shell script. May produce file(s), director(y/ies) and/or symlink(s).
- `symlink` - Creates a symlink.
- `template_dict` - Abstraction for `expand_template` that allows deferring evaluation of values.
- `write` - Creates a file.

Among these actions, the closest builtin analogue to copying a file is using `ctx.actions.symlink`, which depending on the scenario can lead to different behavior at runtime (e.g. NodeJS import resolution is affected, and package managers like [pnpm](https://pnpm.io/) with their integrations like [Rules JS](https://github.com/aspect-build/rules_js) rely on these observable differences). This makes it a poor subsitutite.

A true copy today requires a spawn, via `ctx.actions.run`, `ctx.actions.run_shell`, or `genrule`. Utility rulesets wrap these:

- [`@bazel_skylib`](https://github.com/bazelbuild/bazel-skylib)
  - [`copy_directory`](https://github.com/bazelbuild/bazel-skylib/blob/bac104bc6065308a043489757f2a7ffd159c7fd1/docs/copy_directory_doc.md)
  - [`copy_file`](https://github.com/bazelbuild/bazel-skylib/blob/bac104bc6065308a043489757f2a7ffd159c7fd1/docs/copy_file_doc.md)
- [`bazel_lib`](https://github.com/bazel-contrib/bazel-lib)
  - [`copy_directory` + `copy_directory_bin_action`](https://github.com/bazel-contrib/bazel-lib/blob/222a5bf32e8b6a546059cfff85fe01af7164e596/lib/copy_directory.bzl)
  - [`copy_file` + `copy_file_action`](https://github.com/bazel-contrib/bazel-lib/blob/222a5bf32e8b6a546059cfff85fe01af7164e596/lib/copy_file.bzl)
  - [`copy_to_bin` + `copy_file_to_bin_action` + `copy_files_to_bin_actions`](https://github.com/bazel-contrib/bazel-lib/blob/222a5bf32e8b6a546059cfff85fe01af7164e596/lib/copy_to_bin.bzl)
  - [`copy_to_directory` + `copy_to_directory_bin_action`](https://github.com/bazel-contrib/bazel-lib/blob/222a5bf32e8b6a546059cfff85fe01af7164e596/lib/copy_to_directory.bzl)

Because these are all spawns, they inherit spawn costs:

- **Per-copy overhead disproportionate to the task.**
  Process launch (or worker round trip), sandbox setup, action-cache lookups, and with remote execution an `Execute` call, an `ActionResult`, and CAS round trips — all to produce bytes the build already has.
- **Merkle tree and memory growth.**
  Each copy spawn contributes its tool and inputs to merkle tree construction, increasing CPU and memory costs.
- **Workarounds that punish remote builds.**
  Forcing copies to run locally with [`no-remote`](https://github.com/bazelbuild/bazel-skylib/blob/bac104bc6065308a043489757f2a7ffd159c7fd1/rules/private/copy_common.bzl#L43) avoids remote round trips but forces remote-produced inputs to be downloaded under `--remote_download_minimal` (defeating Build without the Bytes). It also breaks down when using a remote execution service that forbids clients from uploading `ActionResult`s (a common security measure since they can reference arbitrary bytes).
- **Workarounds that punish cache hit rates.**
  Opting copies out of caching with [`no-cache`](https://github.com/bazelbuild/bazel-skylib/blob/bac104bc6065308a043489757f2a7ffd159c7fd1/rules/private/copy_common.bzl#L44) reduces storage demands by the disk cache, but causes the spawns to run more often.
- **Cache-key fragility.**
  The copy is keyed on its implementation (tool digest, command line), so ruleset upgrades invalidate every copy in the graph. If the spawn does not support path mapping (`--experimental_output_paths=strip` flag set and `supports-path-mapping` present in `execution_requirements`) configuration differences (e.g. OS) also contribute to invalidations.
- **Batching trades one problem for another.**
  Batching copies into one spawn amortises overhead but destroys incrementality. In most incremental builds only one input of the batch changed, yet the whole batch re-runs (and re-uploads).

## Artifact Type vs. Materialised Filesystem Type

Declared artifact types do not always map to the actual materialised filesystem types, at least on the surface level.
- Sandboxing
  - Under Bazel, the sandbox spawn strategy populates a directory with symlinks.
  - Under certain RBE services (e.g. EngFlow), sandboxing is implicit. No symlinks are necessary, hardlinks may be used.
- Runfiles Directory
  - Under Bazel (Linux default), runfiles are represented as a tree of symlinks.
  - Under certain RBE services (e.g. EngFlow), runfiles content is materialised as hardlinks.

This proposal does not touch these materialisation strategies, just the resources (files, directories, symlinks) they refer to. e.g.
```
With ctx.actions.symlink
original.txt (file) -> copied.txt (symlink) -> __.runfiles/copied.txt (symlink)
                                   ^^^^^^^
With ctx.actions.copy
original.txt (file) -> copied.txt (file) -> __.runfiles/copied.txt (symlink)
                                   ^^^^
```

Another noteworthy callout is `--remote_download_symlink_template`. Any workloads which are problematic with this flag today (e.g. NodeJS canonicalising file paths before import resolution) will remain problematic with this proposal.

## Source Artifact Assumptions

Bazel assumes all source inputs (files not generated by actions) are ordinary files. That is, [`File.is_directory`](https://bazel.build/rules/lib/builtins/File#is_directory) and [`File.is_symlink`](https://bazel.build/rules/lib/builtins/File#is_symlink) will always be `False` and `File.is_source == True`.

Additionally remote execution has historically had [issues supporting source directories](https://github.com/bazelbuild/bazel-skylib/blob/bac104bc6065308a043489757f2a7ffd159c7fd1/rules/private/copy_common.bzl#L30-L32).

## Build Without The Bytes

Since Bazel 7 BwoB (Build without the Bytes) has been [enabled by default](https://blog.bazel.build/2023/10/06/bwob-in-bazel-7.html). This capability greatly reduces data transfers between Bazel and remote caching/execution services, as well as reducing local storage requirements.

BwoB has come along way since it's [introduction in Bazel 0.25](https://blog.bazel.build/2019/05/07/builds-without-bytes.html), however it has limits. Namely certain builtin action types (`symlink` and `write`) eagerly materialise their outputs. This does not contradict `--remote_download_outputs` as the outputs are _not technically_ produced by a remote, however a naive copy implementation can have much higher IO and storage demands when eagerly materialised.

## Bazel Remote Output Service

The [Bazel Remote Output Service](https://docs.google.com/document/d/1W6Tqq8cndssnDI0yzFSoj95oezRKcIhU57nwLHaN1qk/edit) proposal saw the introduction of `--experimental_remote_output_service` which delegates management of Bazel's output tree to a separate service.

# Proposal

Introduce a new built-in `copy` action:

```starlark
ctx.actions.copy(
    # File: a file (including source tree), tree or symlink
    input,
    # File: a declared file, tree or symlink
    output,
    # string|None: Optional string specifying a single file or tree artifact to extract from input
    path = None,
    # string|None: Optional progress message string.
    # Defaults to "Copying %{input} to %{output}"
    progress_message = None,
)
```

This `copy` action has the following behaviours:

1. **Preserves artifact type.**
   A file copies to a file, a directory to a directory, symlink to a symlink. Mismatched input/output types are an analysis-time error with 2 exceptions:
   1. Directory contents (directory or file) can be copied out by specifying `path`. Type mismatch and not found errors are surfaced when contents are known at execution-time.
   2. Source artifact types are checked as execution-time (see [Source Artifact Assumptions](#source-artifact-assumptions)).
2. **Identical content and executable bit.**
   - File content is identical and executable permission is propagated (`is_executable = True` not required).
   - Files within a directory have identical content.
   - File or directory fulfilled with a symlink (e.g. `symlink(target_file = some_file, output = declare_file(...))`) references the same file or directory as the input and executable permission is propagated.
   - Symlink (e.g. `symlink(target_path = "../some_path", output = declare_symlink(...))`) use the same target string.
3. **No spawns.**
   Like symlink actions, it has no execution strategy, no execution platform, no sandbox, and never executes remotely. Bazel performs the work in-process without affecting spawn specific queues.
4. **Realisation of the output is deferred where possible.** <span id="deferred-realisation"></span>
   When the input's content is remote-backed under Build without the Bytes, the copy completes as a metadata-only operation and the output is materialised on demand, exactly like any other remote-backed output.
   > [!NOTE]
   > Lazy realisation of source artifacts is out of scope for this proposal as Bazel's rewinding machinery recovers a lost artifact by re-executing its generating action. Source artifacts have no generating action, and [`ActionRewindStrategy`](https://github.com/bazelbuild/bazel/blob/5cce7794834a1531ee5fad131f11a9d4d8e66781/src/main/java/com/google/devtools/build/lib/skyframe/rewinding/ActionRewindStrategy.java) requires all lost artifacts to be derived.
   >
   > Related work:
   > - [Lazily create runfiles symlink trees with BwoB](https://github.com/bazelbuild/bazel/pull/26971)
   > - [Lazily create local symlinks with BwoB](https://github.com/bazelbuild/bazel/pull/26045)
   > - [Add `--file_write_strategy` to Bazel](https://github.com/bazelbuild/bazel/pull/24921)
5. **Copy-on-Write and hardlinking.**
   Local copies (e.g. copying source files under local execution) will be performed by [`java.nio.file.Files#copy`](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/nio/file/Files.html#copy(java.nio.file.Path,java.nio.file.Path,java.nio.file.CopyOption...)). Optimisations such as Copy-on-Write (CoW, cloning, reflinking) and hardlinking are not a part of this proposal, but can be implemented later.
   > [!NOTE]
   > Hardlinking is more widly supported, but not always the desired behaviour.
   > - Modifications to hardlinked files can produce unexpected results (edits affecting more files than intended), especially if a file is hardlinked multiple times.
   > - On macOS hardlinking may cause issues with the Gatekeeper if [an executable shares an `inode` with a previously quarantined path](https://developer.apple.com/forums/thread/663456).
   >
   > If hardlinking optimisations are implemented, a way to opt-out (e.g. flag) should be included.

## Supported Input/Output Combinations

| Input               | Output              | `path`   | Outcome                                     |
|---------------------|---------------------|----------|---------------------------------------------|
| file                | `declare_file`      | —        | copied file                                 |
| file (symlink)      | `declare_file`      | —        | copied file (no symlink)                    |
| directory           | `declare_directory` | —        | copied directory                            |
| directory           | `declare_directory` | required | copied subdirectory                         |
| directory           | `declare_file`      | required | copied file in directory                    |
| directory (symlink) | `declare_directory` | —        | copied directory (no symlink)               |
| directory (symlink) | `declare_directory` | required | copied subdirectory in referenced directory |
| directory (symlink) | `declare_file`      | required | copied file in referenced directory         |
| symlink             | `declare_symlink`   | —        | new symlink with the identical target path  |

Copying a symlink out of a directory is not supported as Bazel already dereferences encountered symlinks during metadata collection (raising an error on encountering a dangling symlink).

## Caching

Compared to spawns (`run` and `run_shell` action types) a `copy` action needs no dedicated result type for caching. Everything can be cheaply computed locally from input metadata. In certain circumstances caches aren't even touched (e.g. copied file not used by any remote spawns).

> [!NOTE]
> Like with [_4. Realisation of the output is deferred where possible_](#deferred-realisation), Bazel's rewinding machinery relies on `ActionResult`s. Lazy uploading of source artifacts to remote caches is out of scope for this proposal.

## Introspection

Like with the `symlink` action, little can be inferred from BEP and disk/remote cache records alone. Identification of the input requires `bazel aquery`.

**File Copy Example**

<details open>
<summary>Copy</summary>

```
action 'Copying inputs/u000001.bin to copy/u000001.bin'
  Mnemonic: Copy
  Target: //:copy
  Configuration: k8-fastbuild
  Execution platform: @@platforms//host:host
  ActionKey: 70b26b7b8da2f7e12e89fba77c8dd50d5cef58c5f285fe93520a000041cba820
  Inputs: [bazel-out/k8-fastbuild/bin/inputs/u000001.bin]
  Outputs: [bazel-out/k8-fastbuild/bin/copy/u000001.bin]
```
</details>

<details>
<summary>Run Shell</summary>

```
action 'Copying (spawn) u000000.bin'
  Mnemonic: CopySpawn
  Target: //:spawn
  Configuration: k8-fastbuild
  Execution platform: @@platforms//host:host
  ActionKey: 0adda2c484cd8d4afd7a3e0a9de660b6c659393b8a0419c30f9b79e43cfef12e
  Inputs: [bazel-out/k8-fastbuild/bin/inputs/u000000.bin]
  Outputs: [bazel-out/k8-fastbuild/bin/spawn/u000000.bin]
  Command Line: (exec /bin/bash \
    -c \
    'cp -L "$1" "$2"' \
    '' \
    bazel-out/k8-fastbuild/bin/inputs/u000000.bin \
    bazel-out/k8-fastbuild/bin/spawn/u000000.bin)
# Configuration: 52c85d7b35d3f6598ca5991e2fcb16a4e5cd92789e362244901c18381d8f7241
# Execution platform: @@platforms//host:host
```
</details>

<details>
<summary>Symlink</summary>

```
action 'Symlinking symlink/u000001.bin'
  Mnemonic: Symlink
  Target: //:symlink
  Configuration: k8-fastbuild
  Execution platform: @@platforms//host:host
  ActionKey: c90914cc1e7ea06f84518cf4f179e63dbd20dad9358a550644494828e4a4ed28
  Inputs: [bazel-out/k8-fastbuild/bin/inputs/u000001.bin]
  Outputs: [bazel-out/k8-fastbuild/bin/symlink/u000001.bin]
```
</details>

**File Within Directory Copy Example**

<details open>
<summary>Copy</summary>

```
action 'Copying pkg/tree.dir to pkg/extracted.txt'
  Mnemonic: Copy
  Target: //:extract
  Configuration: k8-fastbuild
  Execution platform: @@platforms//host:host
  ActionKey: 70b26b7b8da2f7e12e89fba77c8dd50d5cef58c5f285fe93520a000041cba820
  Inputs: [bazel-out/k8-fastbuild/bin/inputs/u000.tree]
  Outputs: [bazel-out/k8-fastbuild/bin/extract/u000001.bin]
  CopyPath: u000001.bin
```
</details>

## Build Event Protocol

When `--build_event_publish_all_actions` is specified, BEP will include events for builtin `copy` actions. Content is comparable to what `symlink` actions produce.

**File Copy Example**

<details open>
<summary>Copy</summary>

```json
{
    "id": {
        "actionCompleted": {
            "primaryOutput": "bazel-out/k8-fastbuild/bin/copy/u000001.bin",
            "label": "//:copy",
            "configuration": {
                "id": "52c85d7b35d3f6598ca5991e2fcb16a4e5cd92789e362244901c18381d8f7241"
            }
        }
    },
    "action": {
        "success": true,
        "label": "//:copy",
        "primaryOutput": {
            "uri": "bytestream://127.0.0.1:50051/blobs/cbedcbc892db1449a64cfb8f89a77f692446bf8d63a375d67e9f741069d32e08/4194304"
        },
        "configuration": {
            "id": "52c85d7b35d3f6598ca5991e2fcb16a4e5cd92789e362244901c18381d8f7241"
        },
        "type": "Copy"
    }
}
```
</details>

<details>
<summary>Run Shell</summary>

```json
{
    "id": {
        "actionCompleted": {
            "primaryOutput": "bazel-out/k8-fastbuild/bin/run_shell/u000001.bin",
            "label": "//:spawn",
            "configuration": {
                "id": "52c85d7b35d3f6598ca5991e2fcb16a4e5cd92789e362244901c18381d8f7241"
            }
        }
    },
    "action": {
        "success": true,
        "label": "//:spawn",
        "primaryOutput": {
            "uri": "bytestream://127.0.0.1:50051/blobs/bd145647b14b30aede9085bbaebf0a08fb5ea1b313d062076e86addb9ad257ee/4194304"
        },
        "configuration": {
            "id": "52c85d7b35d3f6598ca5991e2fcb16a4e5cd92789e362244901c18381d8f7241"
        },
        "type": "CopySpawn",
        "commandLine": [
            "/bin/bash",
            "-c",
            "cp -L \"$1\" \"$2\"",
            "",
            "inputs/u000001.bin",
            "bazel-out/k8-fastbuild/bin/run_shell/u000001.bin"
        ],
        "startTime": "2026-07-16T08:40:10.248274260Z",
        "endTime": "2026-07-16T08:40:10.261274260Z"
    }
}
```
</details>

<details>
<summary>Symlink</summary>

```json
{
    "id": {
        "actionCompleted": {
            "primaryOutput": "bazel-out/k8-fastbuild/bin/symlink/u000001.bin",
            "label": "//:symlink",
            "configuration": {
                "id": "52c85d7b35d3f6598ca5991e2fcb16a4e5cd92789e362244901c18381d8f7241"
            }
        }
    },
    "action": {
        "success": true,
        "label": "//:symlink",
        "primaryOutput": {
            "uri": "bytestream://127.0.0.1:50051/blobs/09118149ed6648a30d0f00a48fd3e57e5e3c59696fbfd14832b196d714e96c61/4194304"
        },
        "configuration": {
            "id": "52c85d7b35d3f6598ca5991e2fcb16a4e5cd92789e362244901c18381d8f7241"
        },
        "type": "Symlink"
    }
}
```
</details>

**Directory Copy Example**

<details open>
<summary>Copy</summary>

```json
{
    "id": {
        "actionCompleted": {
            "primaryOutput": "bazel-out/k8-fastbuild/bin/copy/u000.tree",
            "label": "//:copy",
            "configuration": {
                "id": "52c85d7b35d3f6598ca5991e2fcb16a4e5cd92789e362244901c18381d8f7241"
            }
        }
    },
    "action": {
        "success": true,
        "label": "//:copy",
        "configuration": {
            "id": "52c85d7b35d3f6598ca5991e2fcb16a4e5cd92789e362244901c18381d8f7241"
        },
        "type": "Copy"
    }
}
```
</details>

<details>
<summary>Run Shell</summary>

```json
{
    "id": {
        "actionCompleted": {
            "primaryOutput": "bazel-out/k8-fastbuild/bin/run_shell/u000.tree",
            "label": "//:spawn",
            "configuration": {
                "id": "52c85d7b35d3f6598ca5991e2fcb16a4e5cd92789e362244901c18381d8f7241"
            }
        }
    },
    "action": {
        "success": true,
        "label": "//:spawn",
        "configuration": {
            "id": "52c85d7b35d3f6598ca5991e2fcb16a4e5cd92789e362244901c18381d8f7241"
        },
        "type": "CopySpawn",
        "commandLine": [
            "/bin/bash",
            "-c",
            "cp -RL \"$1\"/. \"$2\"/",
            "",
            "bazel-out/k8-fastbuild/bin/u000.tree",
            "bazel-out/k8-fastbuild/bin/run_shell/u000.tree"
        ],
        "startTime": "2026-07-22T04:43:46.260465564Z",
        "endTime": "2026-07-22T04:43:46.276465564Z"
    }
}
```
</details>

<details>
<summary>Symlink</summary>

```json
{
    "id": {
        "actionCompleted": {
            "primaryOutput": "bazel-out/k8-fastbuild/bin/copy/u000.tree",
            "label": "//:symlink",
            "configuration": {
                "id": "52c85d7b35d3f6598ca5991e2fcb16a4e5cd92789e362244901c18381d8f7241"
            }
        }
    },
    "action": {
        "success": true,
        "label": "//:symlink",
        "configuration": {
            "id": "52c85d7b35d3f6598ca5991e2fcb16a4e5cd92789e362244901c18381d8f7241"
        },
        "type": "Symlink"
    }
}
```
</details>

## Example usage

### `@bazel_skylib`'s [`copy_directory`](https://github.com/bazelbuild/bazel-skylib/blob/bac104bc6065308a043489757f2a7ffd159c7fd1/rules/private/copy_directory_private.bzl)

```starlark
def _copy_directory_impl(ctx):
    dst = ctx.actions.declare_directory(ctx.attr.out)
    # Analysis-time error if `src` is not a tree artifact
    ctx.actions.copy(ctx.file.src, dst)

    files = depset(direct = [dst])
    runfiles = ctx.runfiles(files = [dst])

    return [DefaultInfo(files = files, runfiles = runfiles)]

copy_directory = rule(
    implementation = _copy_directory_impl,
    provides = [DefaultInfo],
    attrs = {
        "src": attr.label(mandatory = True, allow_single_file = True),
        # Cannot declare out as an output here, because there's no API for
        # declaring TreeArtifact outputs.
        "out": attr.string(mandatory = True),
    },
)
```

Compared to the canonical implementation:

- The implementation is significantly smaller.
- Both spawn branches (POSIX shell and `cmd.exe`) are replaced by one `copy` call.
- The `_exec_is_windows` attribute and `:is_windows` target are gone.
- The `no-remote`/`no-cache` execution-requirement workaround is gone, along with its Build-without-the-Bytes download penalty.

### `bazel_lib`'s [`copy_to_bin`](https://github.com/bazel-contrib/bazel-lib/blob/222a5bf32e8b6a546059cfff85fe01af7164e596/lib/private/copy_to_bin.bzl)

```starlark
def copy_file_to_bin_action(ctx, file):
    if not file.is_source:
        return file
    if ctx.label.workspace_name != file.owner.workspace_name:
        fail(_file_in_external_repo_error_msg(file))
    if ctx.label.package != file.owner.package:
        fail(_file_in_different_package_error_msg(file, ctx.label))

    if file.path.startswith("bazel-"):
        first = file.path.split("/")[0]
        suffix = first[len("bazel-"):]
        if suffix in ["testlogs", "bin", "out"]:
            print(_probably_should_git_ignore(first))

    dst = ctx.actions.declare_file(file.basename, sibling = file)
    ctx.actions.copy(file, dst)
    return dst

def copy_files_to_bin_actions(ctx, files):
    return [copy_file_to_bin_action(ctx, file) for file in files]

def _copy_to_bin_impl(ctx):
    files = copy_files_to_bin_actions(ctx, ctx.files.srcs)
    return DefaultInfo(
        files = depset(files),
        runfiles = ctx.runfiles(files = files),
    )

copy_to_bin = rule(
    implementation = _copy_to_bin_impl,
    provides = [DefaultInfo],
    attrs = {
        "srcs": attr.label_list(mandatory = True, allow_files = True),
    },
)
```

Compared to the canonical implementation:

- Core validation logic remains unchanged.
- No helper required to copy file(s).
- No toolchain resolution is necessary.
- N/A for the macro since the the canonical implementation just fowards to the rule.

# Backward-compatibility

This proposal won't impact backward compatibility, although feature detection should be considered so that existing utility rulesets can optionally use the newer (and more efficent) API without needing to wait for supported Bazel versions to age out of the support matrix.

# Alternatives Considered

## `ctx.actions.symlink`

Not a copy. Symlinks are runtime-observable and ecosystems (NodeJS resolution, pnpm layouts) assign them meaning. Substituting symlinks where copies are required changes behaviour.

## Server-side short-circuiting of copy spawns

A remote execution service can recognise known copy actions and synthesise the `ActionResult` without scheduling execution. Brittle (copy spawn structure can change easily, breaking heuristics), more network activity vs. proposal, does not benefit local execution.

## Persistent worker / batched copy spawns

Workers amortise process launch but keep every other spawn cost (cache round trips, merkle trees, BES events) and add worker lifecycle complexity. Batching amortises overhead at the cost of incrementality (a batch re-runs and re-uploads when any one input changes). Both are optimisations of the wrong primitive.

### Benchmarks

For this proposal to achieve it's goal, there needs to be a demonstratable improvement over the status quo. To that end a [prototype](https://github.com/Silic0nS0ldier/bazel/pull/18) was created and benchmarked across various scenarios on a 16-core Linux host with btrfs (copy-on-write capable).

Some notes for the results:
- All results are the average of 10 runs.
- Remote builds are run with `nativelink` (same machine) and `--remote_download_outputs=minimal`.
- Peak RSS (resident set size) is collected with polling, true peaks may be higher.
- AC and CAS reflect `--disk_cache` for local and `--remote_cache` for remote.
- OUT refers to disk space used for Bazel's output tree. Reflinks are specially handled to avoid double counting.
- BEP refers to total build event protocol events produced with `--build_event_publish_all_actions`, size is with JSON output.

**Per-Artifact Spawns**

| Content                            | Strategy | Cache    | Cache Misses | Wall   | CPU Time | Peak RSS   | AC        | CAS       | OUT       | BEP | BEP Size  |
|------------------------------------|----------|----------|--------------|--------|----------|------------|-----------|-----------|-----------|-----|-----------|
| 200 × 4 MiB source files           | local    | Cold     |          200 | 1.44 s | 7.96 s   | 1000.2 MiB | 800.0 KiB | 801.6 MiB | 800.0 MiB | 429 | 346.0 KiB |
|                                    |          | 10% miss |           20 | 196 ms | 862 ms   | 863.9 MiB  | 880.0 KiB | 881.6 MiB | 800.0 MiB |  69 | 162.6 KiB |
|                                    |          | 1 miss   |            1 | 110 ms | 149 ms   | 976.0 MiB  | 804.0 KiB | 805.6 MiB | 800.0 MiB |  31 | 143.6 KiB |
|                                    |          | Warm     |            0 | 79 ms  | 114 ms   | 1.1 GiB    | 800.0 KiB | 801.6 MiB | 800.0 MiB |  29 | 141.9 KiB |
|                                    | remote   | Cold     |          200 | 6.28 s | 7.02 s   | 1.3 GiB    | 800.0 KiB | 803.2 MiB | 0 B       | 429 | 338.7 KiB |
|                                    |          | 10% miss |           20 | 806 ms | 603 ms   | 1.3 GiB    | 892.0 KiB | 883.5 MiB | 0 B       |  69 | 158.8 KiB |
|                                    |          | 1 miss   |            1 | 257 ms | 139 ms   | 1.9 GiB    | 816.0 KiB | 807.2 MiB | 0 B       |  31 | 140.2 KiB |
|                                    |          | Warm     |            0 | 123 ms | 148 ms   | 1.5 GiB    | 812.0 KiB | 803.2 MiB | 0 B       |  29 | 138.2 KiB |
| 200 × 4 MiB generated files        | local    | Cold     |          200 | 1.94 s | 10.77 s  | 1.1 GiB    | 1.6 MiB   | 803.1 MiB | 800.0 MiB | 829 | 599.1 KiB |
|                                    |          | 10% miss |           20 | 344 ms | 949 ms   | 1001.3 MiB | 1.7 MiB   | 883.3 MiB | 800.0 MiB | 109 | 188.0 KiB |
|                                    |          | 1 miss   |            1 | 119 ms | 157 ms   | 1.1 GiB    | 1.6 MiB   | 807.1 MiB | 800.0 MiB |  33 | 145.1 KiB |
|                                    |          | Warm     |            0 | 80 ms  | 119 ms   | 1.3 GiB    | 1.6 MiB   | 803.1 MiB | 800.0 MiB |  29 | 141.9 KiB |
|                                    | remote   | Cold     |          200 | 8.28 s | 5.13 s   | 1.2 GiB    | 1.6 MiB   | 808.8 MiB | 0 B       | 829 | 588.9 KiB |
|                                    |          | 10% miss |           20 | 997 ms | 322 ms   | 938.3 MiB  | 1.7 MiB   | 889.5 MiB | 0 B       | 109 | 184.2 KiB |
|                                    |          | 1 miss   |            1 | 375 ms | 136 ms   | 1.0 GiB    | 1.6 MiB   | 812.8 MiB | 0 B       |  33 | 142.0 KiB |
|                                    |          | Warm     |            0 | 119 ms | 118 ms   | 1.1 GiB    | 1.6 MiB   | 808.8 MiB | 0 B       |  29 | 138.3 KiB |
| 10 source dirs, 20 × 4 MiB each    | local    | Cold     |           10 | 525 ms | 4.12 s   | 636.7 MiB  | 40.0 KiB  | 800.1 MiB | 800.0 MiB |  49 | 152.8 KiB |
|                                    |          | 10% miss |            1 | 171 ms | 444 ms   | 689.3 MiB  | 44.0 KiB  | 804.1 MiB | 800.0 MiB |  31 | 141.0 KiB |
|                                    |          | 1 miss   |            1 | 164 ms | 406 ms   | 721.7 MiB  | 44.0 KiB  | 804.1 MiB | 800.0 MiB |  31 | 141.0 KiB |
|                                    |          | Warm     |            0 | 68 ms  | 104 ms   | 781.7 MiB  | 40.0 KiB  | 800.1 MiB | 800.0 MiB |  29 | 139.3 KiB |
|                                    | remote   | Cold     |           10 | 1.40 s | 3.01 s   | 1.5 GiB    | 52.0 KiB  | 800.4 MiB | 0 B       |  49 | 149.1 KiB |
|                                    |          | 10% miss |            1 | 445 ms | 240 ms   | 1.4 GiB    | 56.0 KiB  | 804.5 MiB | 0 B       |  31 | 136.8 KiB |
|                                    |          | 1 miss   |            1 | 441 ms | 203 ms   | 1.5 GiB    | 56.0 KiB  | 804.5 MiB | 0 B       |  31 | 136.8 KiB |
|                                    |          | Warm     |            0 | 115 ms | 113 ms   | 1.9 GiB    | 52.0 KiB  | 800.4 MiB | 0 B       |  29 | 134.8 KiB |
| 10 generated dirs, 20 × 4 MiB each | local    | Cold     |           10 | 1.80 s | 8.33 s   | 813.6 MiB  | 80.0 KiB  | 800.2 MiB | 800.0 MiB |  69 | 165.5 KiB |
|                                    |          | 10% miss |            1 | 362 ms | 875 ms   | 732.0 MiB  | 88.0 KiB  | 880.2 MiB | 800.0 MiB |  33 | 142.6 KiB |
|                                    |          | 1 miss   |            1 | 299 ms | 799 ms   | 809.0 MiB  | 88.0 KiB  | 880.2 MiB | 800.0 MiB |  33 | 142.6 KiB |
|                                    |          | Warm     |            0 | 120 ms | 71 ms    | 850.6 MiB  | 80.0 KiB  | 800.2 MiB | 800.0 MiB |  29 | 139.5 KiB |
|                                    | remote   | Cold     |           10 | 3.03 s | 1.59 s   | 590.1 MiB  | 92.0 KiB  | 800.7 MiB | 0 B       |  69 | 161.7 KiB |
|                                    |          | 10% miss |            1 | 905 ms | 192 ms   | 793.3 MiB  | 88.0 KiB  | 880.8 MiB | 0 B       |  33 | 138.6 KiB |
|                                    |          | 1 miss   |            1 | 895 ms | 158 ms   | 951.1 MiB  | 88.0 KiB  | 880.8 MiB | 0 B       |  33 | 138.6 KiB |
|                                    |          | Warm     |            0 | 101 ms | 70 ms    | 1013.2 MiB | 80.0 KiB  | 800.7 MiB | 0 B       |  29 | 135.0 KiB |

**Batched Spawns**

| Content                            | Strategy | Cache    | Cache Misses | Wall   | CPU Time | Peak RSS   | AC        | CAS       | OUT       | BEP | BEP Size  |
|------------------------------------|----------|----------|--------------|--------|----------|------------|-----------|-----------|-----------|-----|-----------|
| 200 × 4 MiB source files           | local    | Cold     |            1 | 1.41 s | 4.93 s   | 775.9 MiB  | 4.0 KiB   | 800.0 MiB | 800.0 MiB |  31 | 146.1 KiB |
|                                    |          | 10% miss |            1 | 922 ms | 3.01 s   | 928.8 MiB  | 8.0 KiB   | 880.1 MiB | 800.0 MiB |  31 | 141.9 KiB |
|                                    |          | 1 miss   |            1 | 908 ms | 2.76 s   | 1.0 GiB    | 8.0 KiB   | 804.1 MiB | 800.0 MiB |  31 | 141.9 KiB |
|                                    |          | Warm     |            0 | 69 ms  | 102 ms   | 1.0 GiB    | 4.0 KiB   | 800.0 MiB | 800.0 MiB |  29 | 136.0 KiB |
|                                    | remote   | Cold     |            1 | 4.46 s | 3.82 s   | 1.8 GiB    | 4.0 KiB   | 800.1 MiB | 0 B       |  31 | 143.0 KiB |
|                                    |          | 10% miss |            1 | 2.00 s | 500 ms   | 2.0 GiB    | 8.0 KiB   | 880.2 MiB | 0 B       |  31 | 138.5 KiB |
|                                    |          | 1 miss   |            1 | 1.84 s | 252 ms   | 1.7 GiB    | 8.0 KiB   | 804.2 MiB | 0 B       |  31 | 138.4 KiB |
|                                    |          | Warm     |            0 | 117 ms | 131 ms   | 1.8 GiB    | 4.0 KiB   | 800.1 MiB | 0 B       |  29 | 132.4 KiB |
| 200 × 4 MiB generated files        | local    | Cold     |            1 | 1.68 s | 9.00 s   | 794.3 MiB  | 804.0 KiB | 801.6 MiB | 800.0 MiB | 431 | 399.0 KiB |
|                                    |          | 10% miss |            1 | 996 ms | 3.24 s   | 1023.9 MiB | 888.0 KiB | 881.7 MiB | 800.0 MiB |  71 | 170.9 KiB |
|                                    |          | 1 miss   |            1 | 915 ms | 2.77 s   | 1.2 GiB    | 812.0 KiB | 805.6 MiB | 800.0 MiB |  33 | 147.2 KiB |
|                                    |          | Warm     |            0 | 73 ms  | 115 ms   | 1.1 GiB    | 804.0 KiB | 801.6 MiB | 800.0 MiB |  29 | 136.1 KiB |
|                                    | remote   | Cold     |            1 | 6.79 s | 1.93 s   | 851.8 MiB  | 804.0 KiB | 804.1 MiB | 0 B       | 431 | 393.0 KiB |
|                                    |          | 10% miss |            1 | 2.31 s | 273 ms   | 984.5 MiB  | 888.0 KiB | 884.5 MiB | 0 B       |  71 | 167.5 KiB |
|                                    |          | 1 miss   |            1 | 1.98 s | 198 ms   | 1.1 GiB    | 812.0 KiB | 808.2 MiB | 0 B       |  33 | 144.2 KiB |
|                                    |          | Warm     |            0 | 117 ms | 115 ms   | 1.2 GiB    | 804.0 KiB | 804.1 MiB | 0 B       |  29 | 132.3 KiB |
| 10 source dirs, 20 × 4 MiB each    | local    | Cold     |            1 | 858 ms | 3.68 s   | 659.4 MiB  | 4.0 KiB   | 800.0 MiB | 800.0 MiB |  31 | 144.1 KiB |
|                                    |          | 10% miss |            1 | 759 ms | 2.93 s   | 742.8 MiB  | 8.0 KiB   | 804.1 MiB | 800.0 MiB |  31 | 140.0 KiB |
|                                    |          | 1 miss   |            1 | 779 ms | 2.92 s   | 888.5 MiB  | 8.0 KiB   | 804.1 MiB | 800.0 MiB |  31 | 140.0 KiB |
|                                    |          | Warm     |            0 | 69 ms  | 105 ms   | 789.4 MiB  | 4.0 KiB   | 800.0 MiB | 800.0 MiB |  29 | 137.7 KiB |
|                                    | remote   | Cold     |            1 | 2.03 s | 1.20 s   | 1.5 GiB    | 4.0 KiB   | 800.1 MiB | 0 B       |  31 | 140.3 KiB |
|                                    |          | 10% miss |            1 | 1.70 s | 246 ms   | 1.3 GiB    | 8.0 KiB   | 804.2 MiB | 0 B       |  31 | 135.7 KiB |
|                                    |          | 1 miss   |            1 | 1.70 s | 221 ms   | 1.6 GiB    | 8.0 KiB   | 804.2 MiB | 0 B       |  31 | 135.8 KiB |
|                                    |          | Warm     |            0 | 118 ms | 127 ms   | 1.4 GiB    | 4.0 KiB   | 800.1 MiB | 0 B       |  29 | 133.3 KiB |
| 10 generated dirs, 20 × 4 MiB each | local    | Cold     |            1 | 1.38 s | 7.43 s   | 696.0 MiB  | 44.0 KiB  | 800.1 MiB | 800.0 MiB |  51 | 156.8 KiB |
|                                    |          | 10% miss |            1 | 895 ms | 3.31 s   | 745.1 MiB  | 52.0 KiB  | 880.2 MiB | 800.0 MiB |  33 | 141.6 KiB |
|                                    |          | 1 miss   |            1 | 880 ms | 3.29 s   | 779.8 MiB  | 52.0 KiB  | 880.2 MiB | 800.0 MiB |  33 | 141.6 KiB |
|                                    |          | Warm     |            0 | 56 ms  | 67 ms    | 808.9 MiB  | 44.0 KiB  | 800.1 MiB | 800.0 MiB |  29 | 137.9 KiB |
|                                    | remote   | Cold     |            1 | 4.23 s | 655 ms   | 689.9 MiB  | 44.0 KiB  | 800.5 MiB | 0 B       |  51 | 153.1 KiB |
|                                    |          | 10% miss |            1 | 2.20 s | 242 ms   | 851.4 MiB  | 52.0 KiB  | 880.6 MiB | 0 B       |  33 | 137.7 KiB |
|                                    |          | 1 miss   |            1 | 2.20 s | 213 ms   | 906.0 MiB  | 52.0 KiB  | 880.6 MiB | 0 B       |  33 | 137.7 KiB |
|                                    |          | Warm     |            0 | 99 ms  | 73 ms    | 1.0 GiB    | 44.0 KiB  | 800.5 MiB | 0 B       |  29 | 133.4 KiB |

**Copy Action**

Values in brackets are the relative difference vs. per-artifact and batched spawns.

| Content                            | Strategy | Cache    | Cache Misses       | Wall                          | CPU Time                      | Peak RSS                               | AC                                   | CAS                                    | OUT                        | BEP                | BEP Size                              |
|------------------------------------|----------|----------|--------------------|-------------------------------|-------------------------------|----------------------------------------|--------------------------------------|----------------------------------------|----------------------------|--------------------|---------------------------------------|
| 200 × 4 MiB source files           | local    | Cold     | 200<br> (+0, +199) | 417 ms<br> (-1.02 s, -997 ms) | 3.17 s<br> (-4.79 s, -1.76 s) | 782.5 MiB<br> (-217.7 MiB, +6.6 MiB)   | 0 B<br> (-800.0 KiB, -4.0 KiB)       | 0 B<br> (-801.6 MiB, -800.0 MiB)       | 800.0 MiB<br> (+0 B, +0 B) | 429<br> (+0, +398) | 301.3 KiB<br> (-44.7 KiB, +155.3 KiB) |
|                                    |          | 10% miss | 20<br> (+0, +19)   | 120 ms<br> (-76 ms, -802 ms)  | 432 ms<br> (-430 ms, -2.58 s) | 955.7 MiB<br> (+91.8 MiB, +26.9 MiB)   | 0 B<br> (-880.0 KiB, -8.0 KiB)       | 0 B<br> (-881.6 MiB, -880.1 MiB)       | 800.0 MiB<br> (+0 B, +0 B) | 69<br> (+0, +38)   | 157.5 KiB<br> (-5.1 KiB, +15.6 KiB)   |
|                                    |          | 1 miss   | 1                  | 93 ms<br> (-17 ms, -815 ms)   | 152 ms<br> (+3 ms, -2.61 s)   | 1002.7 MiB<br> (+26.7 MiB, -28.2 MiB)  | 0 B<br> (-804.0 KiB, -8.0 KiB)       | 0 B<br> (-805.6 MiB, -804.1 MiB)       | 800.0 MiB<br> (+0 B, +0 B) | 31                 | 142.7 KiB<br> (-958 B, +831 B)        |
|                                    |          | Warm     | 0                  | 81 ms<br> (+2 ms, +11 ms)     | 127 ms<br> (+13 ms, +25 ms)   | 1.0 GiB<br> (-119.3 MiB, -28.0 MiB)    | 0 B<br> (-800.0 KiB, -4.0 KiB)       | 0 B<br> (-801.6 MiB, -800.0 MiB)       | 800.0 MiB<br> (+0 B, +0 B) | 29                 | 141.3 KiB<br> (-612 B, +5.3 KiB)      |
|                                    | remote   | Cold     | 200<br> (+0, +199) | 1.52 s<br> (-4.76 s, -2.94 s) | 3.67 s<br> (-3.35 s, -149 ms) | 1.8 GiB<br> (+464.2 MiB, +26.2 MiB)    | 0 B<br> (-800.0 KiB, -4.0 KiB)       | 800.0 MiB<br> (-3.2 MiB, -76.0 KiB)    | 0 B<br> (+0 B, +0 B)       | 429<br> (+0, +398) | 293.7 KiB<br> (-45.0 KiB, +150.7 KiB) |
|                                    |          | 10% miss | 20<br> (+0, +19)   | 274 ms<br> (-533 ms, -1.73 s) | 474 ms<br> (-129 ms, -26 ms)  | 1.9 GiB<br> (+635.4 MiB, -119.3 MiB)   | 0 B<br> (-892.0 KiB, -8.0 KiB)       | 880.0 MiB<br> (-3.4 MiB, -148.0 KiB)   | 0 B<br> (+0 B, +0 B)       | 69<br> (+0, +38)   | 153.5 KiB<br> (-5.3 KiB, +15.0 KiB)   |
|                                    |          | 1 miss   | 1                  | 175 ms<br> (-82 ms, -1.66 s)  | 197 ms<br> (+58 ms, -55 ms)   | 1.4 GiB<br> (-559.9 MiB, -300.5 MiB)   | 0 B<br> (-816.0 KiB, -8.0 KiB)       | 804.0 MiB<br> (-3.2 MiB, -148.0 KiB)   | 0 B<br> (+0 B, +0 B)       | 31                 | 139.1 KiB<br> (-1.1 KiB, +791 B)      |
|                                    |          | Warm     | 0                  | 123 ms<br> (-43 µs, +6 ms)    | 137 ms<br> (-11 ms, +6 ms)    | 2.1 GiB<br> (+578.2 MiB, +320.6 MiB)   | 0 B<br> (-812.0 KiB, -4.0 KiB)       | 800.0 MiB<br> (-3.2 MiB, -76.0 KiB)    | 0 B<br> (+0 B, +0 B)       | 29                 | 137.8 KiB<br> (-373 B, +5.5 KiB)      |
| 200 × 4 MiB generated files        | local    | Cold     | 200<br> (+0, +199) | 2.73 s<br> (+797 ms, +1.05 s) | 8.14 s<br> (-2.62 s, -858 ms) | 948.8 MiB<br> (-154.7 MiB, +154.5 MiB) | 800.0 KiB<br> (-800.0 KiB, -4.0 KiB) | 801.6 MiB<br> (-1.6 MiB, -28.0 KiB)    | 800.0 MiB<br> (+0 B, +0 B) | 829<br> (+0, +398) | 550.7 KiB<br> (-48.4 KiB, +151.7 KiB) |
|                                    |          | 10% miss | 20<br> (+0, +19)   | 341 ms<br> (-3 ms, -655 ms)   | 906 ms<br> (-43 ms, -2.34 s)  | 1.2 GiB<br> (+278.1 MiB, +255.5 MiB)   | 880.0 KiB<br> (-880.0 KiB, -8.0 KiB) | 881.6 MiB<br> (-1.6 MiB, -52.0 KiB)    | 800.0 MiB<br> (+0 B, +0 B) | 109<br> (+0, +38)  | 182.6 KiB<br> (-5.4 KiB, +11.7 KiB)   |
|                                    |          | 1 miss   | 1                  | 113 ms<br> (-6 ms, -803 ms)   | 155 ms<br> (-2 ms, -2.62 s)   | 1.1 GiB<br> (-4.7 MiB, -137.8 MiB)     | 804.0 KiB<br> (-804.0 KiB, -8.0 KiB) | 805.6 MiB<br> (-1.6 MiB, -52.0 KiB)    | 800.0 MiB<br> (+0 B, +0 B) | 33                 | 144.3 KiB<br> (-885 B, -3.0 KiB)      |
|                                    |          | Warm     | 0                  | 83 ms<br> (+3 ms, +10 ms)     | 125 ms<br> (+6 ms, +10 ms)    | 1.3 GiB<br> (-33.8 MiB, +125.4 MiB)    | 800.0 KiB<br> (-800.0 KiB, -4.0 KiB) | 801.6 MiB<br> (-1.6 MiB, -28.0 KiB)    | 800.0 MiB<br> (+0 B, +0 B) | 29                 | 141.4 KiB<br> (-584 B, +5.3 KiB)      |
|                                    | remote   | Cold     | 200<br> (+0, +199) | 4.38 s<br> (-3.90 s, -2.41 s) | 1.71 s<br> (-3.42 s, -216 ms) | 912.1 MiB<br> (-334.1 MiB, +60.3 MiB)  | 800.0 KiB<br> (-800.0 KiB, -4.0 KiB) | 804.0 MiB<br> (-4.7 MiB, -76.0 KiB)    | 0 B<br> (+0 B, +0 B)       | 829<br> (+0, +398) | 540.2 KiB<br> (-48.7 KiB, +147.1 KiB) |
|                                    |          | 10% miss | 20<br> (+0, +19)   | 561 ms<br> (-436 ms, -1.75 s) | 255 ms<br> (-67 ms, -18 ms)   | 1021.0 MiB<br> (+82.7 MiB, +36.4 MiB)  | 880.0 KiB<br> (-880.0 KiB, -8.0 KiB) | 884.3 MiB<br> (-5.1 MiB, -160.0 KiB)   | 0 B<br> (+0 B, +0 B)       | 109<br> (+0, +38)  | 178.6 KiB<br> (-5.5 KiB, +11.1 KiB)   |
|                                    |          | 1 miss   | 1                  | 248 ms<br> (-127 ms, -1.73 s) | 135 ms<br> (-1 ms, -63 ms)    | 1.1 GiB<br> (+118.1 MiB, -6.0 MiB)     | 804.0 KiB<br> (-804.0 KiB, -8.0 KiB) | 808.0 MiB<br> (-4.8 MiB, -156.0 KiB)   | 0 B<br> (+0 B, +0 B)       | 33                 | 141.1 KiB<br> (-851 B, -3.0 KiB)      |
|                                    |          | Warm     | 0                  | 120 ms<br> (+2 ms, +3 ms)     | 132 ms<br> (+14 ms, +17 ms)   | 1.2 GiB<br> (+30.0 MiB, -33.9 MiB)     | 800.0 KiB<br> (-800.0 KiB, -4.0 KiB) | 804.0 MiB<br> (-4.7 MiB, -76.0 KiB)    | 0 B<br> (+0 B, +0 B)       | 29                 | 137.9 KiB<br> (-411 B, +5.5 KiB)      |
| 10 source dirs, 20 × 4 MiB each    | local    | Cold     | 10<br> (+0, +9)    | 539 ms<br> (+14 ms, -319 ms)  | 1.88 s<br> (-2.25 s, -1.81 s) | 651.7 MiB<br> (+15.0 MiB, -7.7 MiB)    | 0 B<br> (-40.0 KiB, -4.0 KiB)        | 0 B<br> (-800.1 MiB, -800.0 MiB)       | 800.0 MiB<br> (+0 B, +0 B) | 49<br> (+0, +18)   | 150.0 KiB<br> (-2.8 KiB, +5.8 KiB)    |
|                                    |          | 10% miss | 1                  | 115 ms<br> (-56 ms, -644 ms)  | 264 ms<br> (-180 ms, -2.67 s) | 716.7 MiB<br> (+27.3 MiB, -26.1 MiB)   | 0 B<br> (-44.0 KiB, -8.0 KiB)        | 0 B<br> (-804.1 MiB, -804.1 MiB)       | 800.0 MiB<br> (+0 B, +0 B) | 31                 | 140.3 KiB<br> (-723 B, +379 B)        |
|                                    |          | 1 miss   | 1                  | 114 ms<br> (-50 ms, -665 ms)  | 252 ms<br> (-154 ms, -2.67 s) | 822.1 MiB<br> (+100.4 MiB, -66.4 MiB)  | 0 B<br> (-44.0 KiB, -8.0 KiB)        | 0 B<br> (-804.1 MiB, -804.1 MiB)       | 800.0 MiB<br> (+0 B, +0 B) | 31                 | 140.3 KiB<br> (-765 B, +337 B)        |
|                                    |          | Warm     | 0                  | 72 ms<br> (+4 ms, +3 ms)      | 117 ms<br> (+13 ms, +12 ms)   | 849.2 MiB<br> (+67.5 MiB, +59.8 MiB)   | 0 B<br> (-40.0 KiB, -4.0 KiB)        | 0 B<br> (-800.1 MiB, -800.0 MiB)       | 800.0 MiB<br> (+0 B, +0 B) | 29                 | 138.9 KiB<br> (-360 B, +1.2 KiB)      |
|                                    | remote   | Cold     | 10<br> (+0, +9)    | 1.36 s<br> (-42 ms, -666 ms)  | 3.84 s<br> (+830 ms, +2.63 s) | 1.9 GiB<br> (+386.5 MiB, +390.0 MiB)   | 0 B<br> (-52.0 KiB, -4.0 KiB)        | 800.0 MiB<br> (-424.0 KiB, -100.0 KiB) | 0 B<br> (+0 B, +0 B)       | 49<br> (+0, +18)   | 146.0 KiB<br> (-3.1 KiB, +5.7 KiB)    |
|                                    |          | 10% miss | 1                  | 297 ms<br> (-148 ms, -1.40 s) | 599 ms<br> (+359 ms, +353 ms) | 1.3 GiB<br> (-117.4 MiB, +16.5 MiB)    | 0 B<br> (-56.0 KiB, -8.0 KiB)        | 804.0 MiB<br> (-440.0 KiB, -144.0 KiB) | 0 B<br> (+0 B, +0 B)       | 31                 | 136.1 KiB<br> (-773 B, +378 B)        |
|                                    |          | 1 miss   | 1                  | 288 ms<br> (-153 ms, -1.41 s) | 503 ms<br> (+300 ms, +282 ms) | 1.7 GiB<br> (+247.8 MiB, +155.5 MiB)   | 0 B<br> (-56.0 KiB, -8.0 KiB)        | 804.0 MiB<br> (-444.0 KiB, -148.0 KiB) | 0 B<br> (+0 B, +0 B)       | 31                 | 135.9 KiB<br> (-927 B, +127 B)        |
|                                    |          | Warm     | 0                  | 117 ms<br> (+2 ms, -2 ms)     | 114 ms<br> (+1 ms, -13 ms)    | 1.6 GiB<br> (-281.8 MiB, +168.1 MiB)   | 0 B<br> (-52.0 KiB, -4.0 KiB)        | 800.0 MiB<br> (-420.0 KiB, -100.0 KiB) | 0 B<br> (+0 B, +0 B)       | 29                 | 134.6 KiB<br> (-134 B, +1.4 KiB)      |
| 10 generated dirs, 20 × 4 MiB each | local    | Cold     | 10<br> (+0, +9)    | 2.72 s<br> (+924 ms, +1.34 s) | 6.31 s<br> (-2.03 s, -1.12 s) | 700.7 MiB<br> (-112.9 MiB, +4.7 MiB)   | 40.0 KiB<br> (-40.0 KiB, -4.0 KiB)   | 800.1 MiB<br> (-80.0 KiB, -28.0 KiB)   | 800.0 MiB<br> (+0 B, +0 B) | 69<br> (+0, +18)   | 162.8 KiB<br> (-2.8 KiB, +6.0 KiB)    |
|                                    |          | 10% miss | 1                  | 312 ms<br> (-50 ms, -584 ms)  | 683 ms<br> (-192 ms, -2.63 s) | 837.0 MiB<br> (+105.0 MiB, +92.0 MiB)  | 44.0 KiB<br> (-44.0 KiB, -8.0 KiB)   | 880.1 MiB<br> (-84.0 KiB, -52.0 KiB)   | 800.0 MiB<br> (+0 B, +0 B) | 33                 | 141.8 KiB<br> (-727 B, +209 B)        |
|                                    |          | 1 miss   | 1                  | 399 ms<br> (+100 ms, -481 ms) | 706 ms<br> (-93 ms, -2.58 s)  | 806.5 MiB<br> (-2.5 MiB, +26.7 MiB)    | 44.0 KiB<br> (-44.0 KiB, -8.0 KiB)   | 880.1 MiB<br> (-84.0 KiB, -52.0 KiB)   | 800.0 MiB<br> (+0 B, +0 B) | 33                 | 141.8 KiB<br> (-718 B, +208 B)        |
|                                    |          | Warm     | 0                  | 58 ms<br> (-62 ms, +2 ms)     | 79 ms<br> (+8 ms, +12 ms)     | 811.7 MiB<br> (-38.8 MiB, +2.8 MiB)    | 40.0 KiB<br> (-40.0 KiB, -4.0 KiB)   | 800.1 MiB<br> (-80.0 KiB, -28.0 KiB)   | 800.0 MiB<br> (+0 B, +0 B) | 29                 | 139.1 KiB<br> (-433 B, +1.2 KiB)      |
|                                    | remote   | Cold     | 10<br> (+0, +9)    | 2.75 s<br> (-281 ms, -1.48 s) | 526 ms<br> (-1.06 s, -129 ms) | 753.1 MiB<br> (+162.9 MiB, +63.2 MiB)  | 40.0 KiB<br> (-52.0 KiB, -4.0 KiB)   | 800.4 MiB<br> (-336.0 KiB, -88.0 KiB)  | 0 B<br> (+0 B, +0 B)       | 69<br> (+0, +18)   | 158.7 KiB<br> (-3.0 KiB, +5.6 KiB)    |
|                                    |          | 10% miss | 1                  | 635 ms<br> (-270 ms, -1.57 s) | 145 ms<br> (-47 ms, -97 ms)   | 910.3 MiB<br> (+117.0 MiB, +58.9 MiB)  | 44.0 KiB<br> (-44.0 KiB, -8.0 KiB)   | 880.4 MiB<br> (-352.0 KiB, -140.0 KiB) | 0 B<br> (+0 B, +0 B)       | 33                 | 137.8 KiB<br> (-745 B, +104 B)        |
|                                    |          | 1 miss   | 1                  | 582 ms<br> (-314 ms, -1.62 s) | 116 ms<br> (-42 ms, -97 ms)   | 925.0 MiB<br> (-26.1 MiB, +19.0 MiB)   | 44.0 KiB<br> (-44.0 KiB, -8.0 KiB)   | 880.4 MiB<br> (-352.0 KiB, -140.0 KiB) | 0 B<br> (+0 B, +0 B)       | 33                 | 137.8 KiB<br> (-743 B, +105 B)        |
|                                    |          | Warm     | 0                  | 103 ms<br> (+2 ms, +4 ms)     | 93 ms<br> (+23 ms, +20 ms)    | 990.8 MiB<br> (-22.3 MiB, -34.5 MiB)   | 40.0 KiB<br> (-40.0 KiB, -4.0 KiB)   | 800.4 MiB<br> (-324.0 KiB, -88.0 KiB)  | 0 B<br> (+0 B, +0 B)       | 29                 | 134.9 KiB<br> (-160 B, +1.5 KiB)      |
