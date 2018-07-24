---
created: 2018-07-18
last updated: 2018-07-24
status: To be reviewed
reviewers: ulfjack (lead), lberki, dslomov
title: Test execution on Windows without Bash
tracking issues: "#5508, #4691, #4319"
author: laszlocsomor
---

# Abstract

Let's change how Bazel runs tests on Windows to no longer require Bash.

# Background

For every test, Bazel requires a complex Bash script to set up the environment
and to run the test. Therefore Windows users need to install MSYS2 Bash to run
tests. This is undesirable (see
[issue #4319](https://github.com/bazelbuild/bazel/issues/4319)).

Bazel should run tests without requiring Bash (see
[issue #5508](https://github.com/bazelbuild/bazel/issues/5508)). This document
explains how.

# Current design

## Test execution

Bazel runs tests by executing `TestRunnerAction`s. Test actions are similar to
build actions (e.g. `SpawnAction`): they take a list of input files, execute a
command, and produce some output files.

The inputs of the test action are the test binary and its dependencies, plus
some extra helper files such as the test wrapper script
(`@bazel_tools//tools/test/test-setup.sh`) and optionally the coverage collector
and LCOV merger tools.

The command of the test action is the test wrapper script, plus additional
user-specified arguments. This script initializes the environment for the actual
test binary, then runs the test.

The outputs of the test action are the XML test log and the "undeclared outputs"
file. The XML test log is an XML file that records two things: metadata about
the test (such as the test target name, whether the test passed or failed), and
the textual text log (that is, the output to `stdout` and `stderr`). The
undeclared outputs file is a zip archive of files that the test created and are
potentially interesting to the user.

## Test termination

Tests may terminate:

*   cleanly, if the test process terminates by itself (regardless of whether the
    test passed or failed)

*   abruptly, if the test process is killed:

    *   by the user, interrupting test execution using Ctrl+C

    *   by Bazel, when a test times out or another test fails and
        `--test_keep_going` is disabled).

In every case the test action produces an XML test log.

Bazel on Windows terminates tests abruptly in the following locations:

*   [`WindowsSubprocesses.terminate()` in `WindowsSubprocess.waiterThreadFunc()`](https://github.com/bazelbuild/bazel/blob/a9c71b4c8a6d8b998c950c35c1e0a697a0d68ffa/src/main/java/com/google/devtools/build/lib/windows/WindowsSubprocess.java#L159),
    when the test times out, and

*   [`WindowsSubprocess.destroy()` in `FutureCommandResultImpl.waitForProcess()`](https://github.com/bazelbuild/bazel/blob/a9c71b4c8a6d8b998c950c35c1e0a697a0d68ffa/src/main/java/com/google/devtools/build/lib/shell/FutureCommandResultImpl.java#L99),
    when the test is interrupted.

## Test wrapper control flow

1.  Absolutizes path-storing environment variables.

    **Why**: At the time of creating the `TestRunnerAction` (along with its
    environment), Bazel doesn't yet know the execution root the test will run
    under. `test-setup.sh` absolutizes the envvars by making them relative to
    `$PWD`.

1.  Creates some directories, e.g. for the undeclared outputs, the shard status
    file, the XML test log, and the test temp directory.

1.  Exports some environment variables.

    **Why**: Tests and test runners require envvars such as `$TEST_TMPDIR` and
    `$TEST_SHARD_INDEX`.

1.  Defines `rlocation()` to look up paths of data-dependencies.

    **Why**:

    *   `test-setup.sh` itself looks up the test executable's path.

    *   For sake of shell tests (in case the actual test is a `sh_test`). This
        use-case does not apply for the subset of shell tests that use the Bash
        runfiles library in `@bazel_tools//tools/bash/runfiles`.

1.  Defines `encode_output_file()`.

    This function runs `perl` and `sed` to sanitize the textual test log for the
    test XML test log's CDATA section. `write_xml_output_file()` calls this
    function.

1.  Defines `write_xml_output_file()` that:

    *   Creates the test XML file, with the help of `encode_output_file()`.

    *   Removes `${XML_OUTPUT_FILE}.log`.

        **Why**: `${XML_OUTPUT_FILE}.log` is a temporary file containing the
        test's raw output.

1.  Changes the current directory to the test's runfiles directory (only when
    coverage collection is disabled).

    **Why**: Actions run in the execroot by default. Changing the directory
    prevents a locally executed, non-sandboxed test from accessing undeclared
    inputs files.

1.  Adds `.` to `$PATH`.

    **Why**: To run the test executable without having to add `./` if the binary
    is in the current directory.

    In fact this step is unnecessary, because the test executable's path is
    always absolute.

1.  Sets `$TEST_PATH` to the absolute path of the test executable.

    If `$TEST_SHORT_EXEC_PATH` is defined, it sets an alternative `$TEST_PATH`.
    
    **Why**: To avoid too long paths on Windows with remote execution.

1.  Traps all signals to be handled by `write_xml_output_file()`.

    **Why**: If the test is abruptly terminated (e.g. the user interrupts test
    execution or the test times out), Bash executes the signal handler and
    `write_xml_output_file()` writes an output file, which records the fact that
    the test terminated abruptly.

1.  Runs the test:

    If it can, runs the test as a subprocess and redirects the test's output to
    `${XML_OUTPUT_FILE}.log` while also streaming the output to stdout via
    `less`; otherwise runs the test directly and `tee` the test's output to
    `${XML_OUTPUT_FILE}.log`.

    In both cases, runs the test via the `tools/test/collect_coverage.sh` if
    requested, which:

    1.  Absolutizes and exports some path-storing envvars (e.g.
        `$COVERAGE_MANIFEST`, `$COVERAGE_DIR`).

    1.  Changes the current directory to the test's workspace, runs the
        test, stores the exit code.

    1.  `exec()`s the `$LCOV_MERGER`


    `collect_coverage.sh` runs in "legacy mode" if `$LCOV_MERGER` is undefined.
    This mode triggers Google-specific code paths that rely on `/usr/bin/lcov`.
    This use-case is unsupported on Windows.

1.  Resets all signal handlers, calls `write_xml_output_file()`.

    **Why**: The test terminated normally so it's safe to reset the default
    signal handlers.

1.  Writes the manifest- and annotation files for the undeclared outputs.

    **Why**: Tests may produce valuable output files in the
    `$TEST_UNDECLARED_OUTPUTS_DIR` directory. These outputs are undeclared, they
    are not part of the test acton's signature, so Bazel is unaware of them.
    Bazel archives the entire directory to retrieve these files from the from
    the sandbox or remote machine.

1.  Creates a zip file of the undeclared outputs.

# Requirements of the solution

## No extra software

Running tests with Bazel must require no extra software on a fresh Windows 7
desktop installation other than what the tested language requires (compiler,
runtime, etc.).

We require Windows 7 compatibility because that is the oldest Windows version
Bazel supports.

## Drop-in replacement

The new solution must be a drop-in replacement for the old test execution
mechanism, meaning all features that the old design provides &mdash; such as
writing an XML test log even upon abrupt test termination, or running under a
coverage collector &mdash; must either keep working under the new design, or be
dropped for a good reason.

## Guarded by a flag

The new solution must be guarded by a flag. This allows us to roll out the
feature in multiple stages, and users to easily revert to the old behavior in
case the new one is buggy.

# Non-requirements of the solution

The solution does not have to cover remote test execution, because Bazel does
not manage remote processes. To run a test remotely, Bazel sends a request to a
service and waits for the reply which contains the XML test log and all other
test outputs.

# Design

## Constraints

Windows doesn't support signal handlers and processes cannot run custom cleanup
routines upon termination. To interrupt a test, Bazel currently forcefully
terminates the test process (see [Test termination](#test-termination)), leaving
no chance for cleanup. Therefore in order to capture the textual test log and
convert it to XML even when the test is interrupted, the test and the log
capturer cannot run in the same process.

Windows doesn't support process replacement (`exec(3)` on Unixes), therefore in
order to set up the test's environment, the test setup process must create the
test process.

## Processes

There will be two or three processes per test (same numbers as today):

*   a parent process, which runs the test wrapper

*   a child process, which runs either the test binary or the coverage collector

*   optionally a second child process after the first one finished, which runs
    the `$LCOV_MERGER` when coverage collection is requested

## Abrupt test termination

We change Bazel not to forcefully terminate the test wrapper process upon
interruption, but instead:

1.  ask the process to interrupt and shut down, by sending it a control message
    on some channel

1.  wait some time for the process to complete its shutdown protocol

1.  forcefully terminate the process only if it's still running after a timeout.

### Interruption request

The communication channel between Bazel and the test wrapper process will be the
test wrapper's `stdin`.

Using `stdin` is simple and convenient: the only supported control message is
the request for interruption. For now a single byte will suffice as this
message. This communication protocol is easily exendable if necessary.

Using `stdin` is also safe: no other process has a handle to the test wrapper's
`stdin`, so no other process will inadvertently send the interruption request.

### Shutdown protocol

When requested to interrupt and shut down, the test wrapper should exit as soon
as possible.

The primary output of test execution is the XML test log: it carries the most
useful information for the user. The XML file records the test's status (passed
or failed) and the test's textual output. The test wrapper should ensure it
always writes the XML test log.

Textual test outputs are typically around a few MBs, though at their extreme
reach sizes of several GBs. To avoid having to write the whole XML file as part
of the shutdown protocol, the test wrapper will continuously convert the log as
the test is running. When requested to interrupt, the test wrapper only has to
convert the tail of the test log, append the end of the XML file, and exit.

To make sure that the test wrapper has read access the child process' output,
the test wrapper:

1.  creates a temporary file under `$TEST_TMPDIR`, opens it for writing and read
    sharing

1.  creates the child process such that `stdout` and `stderr` are redirected to
    the temporary file.

After the test wrapper finished writing the XML test log, it starts archiving
the undeclared outputs. This operation may not finish within the forceful
termination timeout, but that's fine: the most important output is the XML file.
If the test was interrupted, retrieving the undeclared outputs is done on a
best-effort basis.

### Termination timeout

The timeout should be long enough for the test wrapper to finish writing the XML
file, terminate the active child process, and exit. We established that the test
wrapper will continuously covert the test log, so we expect that finishing up
the XML file is faster than writing the entire multi-GB test log.

A timeout of 1 second seems to suffice.

## `rlocation()` support for `sh_test` rules

The new test execution mechanism will not define `rlocation()` for the benefit
of `sh_test` rules. See the [Backward compatibility](#backward-compatibility)
section for details.

# Implementation language

We'll implement the test wrapper in C++, compile it as a x86\_64 Windows binary,
and bundle it with Bazel as `@bazel_tools//tools/test:test-wrapper.exe`.

Rationale:

*   we have experience with C++

*   Bazel already contains C++ code, so we introduce no new language to the code
    base

*   the binary only has to run on Windows on x86\_64 CPUs: even if remote
    execution supported mixing Windows host with Linux executor or the other way
    around, there's no interpreted or JIT'ed language that runs on all platforms
    without requiring any third-party runtime

*   a statically linked binary requires no runtime and runs on a fresh Windows 7
    installation without additional software

*   it's easy to interface with the Windows API

The alternative would be to build a .NET application: according to a
[Microsoft blog post](https://blogs.msdn.microsoft.com/astebner/2007/03/14/mailbag-what-version-of-the-net-framework-is-included-in-what-version-of-the-os/)
all versions of Windows 7 and of Windows Server 2008 R2 (the equivalent server
version) include the .NET framework 3.5.

# Design adequacy

Addressing every step in the [current design](#current-design):

*   steps 1, 2, 3: We'll use Windows API functions or custom logic for these. We
    pass environment variables to `CreateProcessW` to export them for the child
    process(es). 

*   step 4: The new test wrapper will not do this, see
    [Backward compatibility](#backward-compatibility). To look up the test
    binary's path, we'll use the C++ runfiles library in
    `@bazel_tools//tools/cpp/runfiles`.

*   step 5: Same as steps 1, 2, 3.

*   step 6: We'll implement the encoding as custom logic.

*   step 7: We'll use `SetCurrentDirectoryW`, or if the path is too long, create
    a junction under `$TEST_TMPDIR` pointing at it.

*   step 8: This step is unnecessary. The new test wrapper will look up the test
    binary's path using the C++ runfiles library; see step 4.

*   step 9: If the path is too long, we'll create a junction like in step 7.

*   step 10: This step is unnecessary by design, see
    [Interruption request](#interruption-request).
    
*   step 11: The test wrapper opens the file that the test's textual output is
    redirected to with read sharing, enabling to stream the output to its own
    `stdout`.

    To run under the coverage collector, Bazel creates the
    [child process](#processes) to run the coverage collector, then a second
    child process to run the `$LCOV_MERGER`.

*   step 12: This step is unnecessary on Windows, see
    [Constraints](#constraints).

*   step 13: We'll use Windows API functions to list the directory
    (`FindFirstFileW`) and to write the files. It's enough to open these files
    with read sharing and without deletion sharing, because if the test wrapper
    is forcefully terminated then the OS closes its file handles.

*   step 14: We'll use the zip compressor in `//third_party/ijar`.

# Backward-compatibility

The new solution will not define `rlocation()` as a Bash function, potentially
breaking existing `sh_test` rules that do not use the Bash runfiles library in
`@bazel_tools//tools/bash/runfiles`. We anticipate no breakages though because
in a survey conducted in March-April 2018,
[no Windows user reported using Bash](https://groups.google.com/d/msg/bazel-discuss/FZWbP6hdoiw/VVtH2lXbAQAJ).
(Bazel's own shell tests are also unaffected, because the ones that run on
Windows already use the Bash runfiles library.)

In the off-chance that someone is affected and are unable to migrate their tests
to the Bash runfiles library, we'll update the Bash launcher in
`@bazel_tools//tools/launcher` to load a Bash script before the main test
script, which will define `rlocation()` with the same body as `test-setup.sh`
does today.

In every other respect the new system will be a drop-in replacement for the old
test execution mechanism. We will roll it out in several stages (see
[rollout plan](#rollout-plan)) so users will have time to test it, report bugs,
and revert to the old mechanism in case they discover bugs.

# Rollout plan

We will roll out this feature over several Bazel minor versions:

1.  version `0.N.*`: contains both the new and old test execution mechanisms and
    supports the `--[no]windows_bashless_test` flag. By default the flag is
    disabled and Bazel uses the old (Bash-based) test execution. We ask users to
    enable the flag and report bugs. We move on to the next stage when all known
    bugs are fixed.

1.  version `0.N+k.*` (`k` &gt; 0): the flag is enabled by default. We ask users
    to file bugs whenever they find a use-case to disable the flag. We move on
    to the next stage if no new bugs are reporter for a version.

1.  version `0.N+m.*` (`m` &gt; `k`): the flag is a no-op and Bazel no longer
    contains the code for the old test execution. We ask users to remove the
    flag from their `.bazelrc` files. We move on to the next stage
    unconditionally.

1.  version `0.N+m+1.*`: the flag is no longer supported.
