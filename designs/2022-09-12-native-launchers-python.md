---
created: 2022-09-12
last updated: 2022-09-12
status: To be reviewed
reviewers:
  - TODO
title: Cross-platform native launchers for Python
authors:
  - groodt
---


# Abstract

This document describes an approach for launching `py_binary` artifacts hermetically using the resolved python toolchain.


# Background

Currently, `py_binary` is non-hermetic and launches inconsistently between platforms.

On macos and Linux, there is a [python_stub](https://github.com/bazelbuild/bazel/blob/master/src/main/java/com/google/devtools/build/lib/bazel/rules/python/python_stub_template.txt)
that is non-hermetic and requires a "bootstrap" Python interpreter on the host. The "shebang" can be overridden, but
a "shebang" is always dependent on the runtime host.

On Windows, there is a [native launcher](https://github.com/meteorcloudy/bazel/blob/master/src/tools/launcher/python_launcher.cc)
that launches `python.exe` on the host which then launches the `py_binary` with the same `python_stub` as macos and Linux.

Related issues:
* [py_binary with hermetic toolchain requires a system interpreter](https://github.com/bazelbuild/rules_python/issues/691)
* [Neither python_top nor python toolchain works with starlark actions on windows](https://github.com/bazelbuild/bazel/issues/7947#issuecomment-495265016)

This situation is undesirable because it assumes that the target platform has a bootstrapping python interpreter 
available and makes the hermetic Python interpreters available with `rules_python` less useful. It is also surprising to 
users who expect bazel to output self-contained binary artifacts for a target platform.

The reason this situation exists is because of "bootstrapping". Ultimately, *something* needs to find the python
interpreter in the runfiles and use that to launch the program. Currently, Bazel assumes the target platform will
be able to provide the "bootstrapping" functionality.


# Proposal

Extend the native launcher functionality to all platforms and use it to locate the relevant python interpreter and 
python program in the `runfiles` tree to launch the `py_binary`. No assumptions should be made about the target platform.

In pseudo-code, the proposal is as follows:

```
exec(env, runfiles-interpreter, ["interpreter_arg1",], "main.py", ["arg1",])
```

| Token                  | Description |
| ---------------------- | ----------- |
| env                    | Dictionary of key-value pairs for the environment of the process       |
| runfiles-interpreter   | The resolved python toolchain in runfiles        |
| ["interpreter_arg1",]  | An array of arguments to provide to the python interpreter        |
| "main.py"              | The python program to launch in runfiles        |
| ["arg1",]              | An array of arguments to provide to the python program as sys.argv[1:]        |

This native launcher idea has been proposed a few times by bazel contributors and the community:
* [Greg Roodt (Community)](https://github.com/bazelbuild/rules_python/issues/691#issuecomment-1174935972)
* [Yun Peng (Google)](https://github.com/bazelbuild/bazel/issues/7947#issuecomment-495265016)
* [Richard Levasseur (Google)](https://github.com/bazelbuild/rules_python/issues/691#issuecomment-1186379617)

Some related work has been done that fixes Linux to Windows cross-builds of the Windows launcher. See: [Fix Linux to Windows cross compilation of py_binary, java_binary and sh_binary using MinGW](https://github.com/bazelbuild/bazel/pull/16019)
This proposal would aim to go further and have these launchers available on all platforms, including cross_builds where appropriate toolchains are in place.

Once this proposal is implemented, it would enable cross-builds of hermetic `py_binary` for all major platforms. It
would also remove the complexity introduced by having so many chains of nested execution to launch a python program.

Finally, while this proposal is specific to python, this solution could perhaps be reused for `java_binary`, `sh_binary`
and perhaps be made available for any custom rules that require an interpreter to launch.


# Backward-compatibility

This proposal could require users to setup a cc toolchain for remote execution.
