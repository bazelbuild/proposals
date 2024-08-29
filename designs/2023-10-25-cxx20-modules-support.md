---
created: 2023-10-25
last updated: 2023-10-25
status: To be reviewed
reviewers:
title: C++20 Modules Support
authors:
  - PikachuHy
discussion thread:
---

# Abstract
A solution of support for C++20 Modules is presented. The solution introduces a new compilation model for handling C++ code. The new model is inspired by the one used by CMake. The proposed approach utilizes the Skyframe Restart Mechanim to handle dynamic dependencies. Currently, only Clang is supported.

# Introduction
C++20 Modules have received much attention in recent years due to their encapsulation and isolation, which offer important benefits in term of compilation time speedup. However, it has been found to be challenging for build system to support C++20 Modules.

The C++20 modules introduce dependencies between the translation units. It brings the new chanllenge to the bulild systems to dicscover and maintian the depdendencies of the project. The dependencies of a translation unit are easy to change and can be changed frequently, which require the build system to support dynamic dependencies. Currently CMake 3.26 and above provides C++20 Modules support with Ninja. There are basic support for C++20 modules based on Starlark rules ( [rules_cc_module](https://github.com/rnburn/rules_cc_module) and [rules_ll](https://github.com/eomii/rules_ll) ). But neither of them supports discover dependencies and dynamic dependencies, limiting their usability.

The proposal presents a solution for supporting C++20 Modules in Bazel. In this solution, a new C++ compilation model is adopted to support discovering dependencies and handling dependencies. To support dynamic dependencies, the Skyframe restart mechanism is utilized. 

# Proposal
There are two challenges in supporting C++20 Modules in Bazel.

1. Translation units must be compiled in an order. How to determine the compilation order?
2. Dependency graph may change after source code got changed (e.g. insert or remove `import` statements). How to adapt to such changes in time?

To address the above-mentioned issues, a new compilation model is adopted to Bazel. This model is inspired by the one used in CMake for supporting C++20 Modules, but it has been modified to suit the specific needs of Bazel. The new compilation model is as follows:

- scanning C++20 Modules dependencies. The tool `clang-scan-deps` is employed to scan the dependencies of translation units, and the scan results are stored in `.ddi` files. Each translation unit has a corresponding `.ddi` file. The `.ddi` here is an abbreviation for Direct Dependency Information.
- collecting C++20 Modules dependencies. All C++20 Modules information from `.ddi` files is consolidate into a new `.CXXModules.json` file
- using C++20 Modules dependencies. For each translation unit,  a compiler response file (a `.modmap` file) is generated based on its corresponding `.ddi` file and the `.CXXModules.json` file. An auxiliary file named `.modmap.input` is also generated for easier access to inputs files.
- handling C++20 Modules dynamic dependencies. Before the action `CppCompileAction` is executed, the C++20 Modules dependencies that the aciton really needs are obtained through Action Input Discovery based on its corresponding `.modmap` file. And the Skyframe Restart Mechanism is utilized to ensure that all the necessary C++20 Modules dependencies are generated correctly during compilation.
- compiling C++20 Modules. A two-phase compilation is employed when a translation unit is a module interface unit. The two-phase compilation takes 2 steps to compile a source file to an object file, which may has higher parallelism.

In the upcoming subsections, we will provide detailed explanations of each step in the new compilation model. To provide a more vivid explantion of each step, we will use a case study throughout the entire explantion. For the sake of simplicity, only three files are taken into consideration: `main.cc`, `foo.cppm` and `bar.cppm`. The `main.cc` provides `main` function. Both `foo.cppm` and `bar.cppm` are module interface unit. The `foo.cppm` provides C++ module `foo` with a function `foo`; while the `bar.cppm` provides C++ module `bar` with a function `bar`. The `main` function uses the function `foo` and the function `foo` uses the function `bar`. Here is the code samples.
```cpp
// bar.cppm
export module bar;
export void bar() {
}

// foo.cppm
export module foo;
import bar;
export void foo() {
    bar();
}

// main.cc
import foo;
int main() {
	foo();
    return 0;
}
```
And the Bazel build configuration is here.
```python
cc_library(
    name="demo",
    srcs=["main.cc"],
    module_interfaces=["foo.cppm", "bar.cppm"],
    copts=["-std=c++20"]
)
```
The new attribute `module_interfaces` is indicates that `foo.cppm` and `bar.cppm` are module interface units.
## Scanning C++20 Modules dependencies
The [P1689 paper](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2022/p1689r5.html) provides the format for describing dependencies of source files. In Clang, `clang-scan-deps` provides the functionality to obtain source files dependencies. The `clang-scan-deps` provides a command-line interface to scan the source files. The single file mode provided by `clang-scan-deps` is employed instead of the batch mode. The following command can be used to scan the `foo.cppm` file.
```bash
# scan deps
clang-scan-deps -format=p1689 -- /usr/bin/clang++ -std=c++20 -c foo.cppm -o foo.o
```
And the scan result is like that.
```json
{
  "revision": 0,
  "rules": [
    {
      "primary-output": "foo.o",
      "provides": [
        {
          "is-interface": true,
          "logical-name": "foo",
          "source-path": "foo.cppm"
        }
      ],
      "requires": [
        {
          "logical-name": "bar"
        }
      ]
    }
  ],
  "version": 1
}
```
The result indicates that the source file `foo.cppm` provides a C++20 module with the logical name `foo` and requires a C++20 module with the logical name `bar`. The dependency information for other source files can be obtained through a similar approach. All scan results are stored in `.ddi` files. Each source file has a corresponding `.ddi` file. 
When scanning the source files with `clang-scan-deps`, it is required to use the same compilation flags as those used during the actual compilation process. To meet the requirement, the action `CppCompileAction`is employed with the following modification.

- action tool: use a new tool `deps-scanner`. The default implementation is a script wrapper that internally calls `clang-scan-deps`. The compilation flags are passed from the outside, simulating the actual compilation of a source file. Since the `clang-scan-deps` does not have the capability to write the scan result to a specific output file, redirection is used to write the standard output to a file. In our implementation, the output file is specified by exposing the environment variable `DEPS_SCANNER_OUTPUT_FILE`.
- action name: use a new action name `c++20-deps-scanning`, which is derived from an existing action name `c++-header-parsing`. The adoption of a new action not only serves to clearly express the intent of the action as a dependency scanning process but also provides greater flexibility in managing `CompileBuildVariables`. Consequently, due modifications are necessary in the associated action confgiuration to align with the updated action name.
- inputs: header files should be specified as inputs. However, the Built Module Interface (BMI) files are not required.
- input discovery and input pruing: `.d` file is generated after scanning. The `.d` file contains the transitional  dependency information for headers and is used to determine which header files are actually needed for the compilation.

The scanning process is illustarted as follows.
```
              ┌────────┐                  ┌────────┐                  ┌────────┐
              │foo.cppm│                  │bar.cppm│                  │main.cc │
              └────┬───┘                  └────┬───┘                  └────┬───┘
c++20-deps-scanning│                           │                           │
                   │                           │                           │
          ┌────────┴──────┐           ┌────────┴──────┐           ┌────────┴──────┐
          ▼               ▼           ▼               ▼           ▼               ▼
      ┌───────┐       ┌───────┐   ┌───────┐       ┌───────┐   ┌────────┐      ┌───────┐
      │foo.ddi│       │foo.d  │   │bar.ddi│       │bar.d  │   │main.ddi│      │main.d │
      └───────┘       └───────┘   └───────┘       └───────┘   └────────┘      └───────┘
```
The default implement of  `deps-scanner` is as follows.
```bash
#!/bin/bash
#
# Ship the environment to the C++ action
#
set -eu

# Set-up the environment


# Call the C++ compiler
/path/to/clang-scan-deps -format=p1689 -- /path/to/clang "$@" > $DEPS_SCANNER_OUTPUT_FILE
```
## Collecting C++20 Modules dependencies
All the information of C++20 Modules dependencies is processed on a library basis and is stored in a `.CXXModules.json` file. The dependency collection is performed using a dedicated action `Cpp20ModulesInfoAction`. After scanning C++20 Modules dependencies, the results of each source file is stored in the corresponding `.ddi` file. The goal of the `Cpp20ModulesInfoAction` is to integrate the information from these `.ddi` files. The integrated information is written back to a `.CXXModules.json` file for future use.
The information stored in each `.ddi` file follows the format described in the P1689 paper. However, it is redundant for practical use since we use single file mode. For each source file, the most crucial aspect is to known which C++20 Modules it requires. If it is a module interface, what is the module name it provides. Therefore, a more simplified format that focuses on the dependencies of a source code file and the module name can be more practical for further use. The intermediate format contains three fields.

- `needProduceBMI`: true of false. if the source file is a module interface unit, it is true.
- `moduleName`: the module name provided by module interface unit. if the source file is not a module interface unit, it is empty.
- `requireModules`: an array of module name required by the source file

with the new format, the scan result of `foo.cppm` can be simplified to
```json
{
  "needProduceBMI": true,
  "moduleName": "foo",
  "requireModules": ["bar"]
}
```
The format of `.CXXModules.json` file contains two fields.

- `modules`: a dict. The key is module name, and the value is the corresponding BMI file. 
- `usages`: a dict. The key is module name, and the value is an array of module name required by the module.

In our three files example, the final information stored in `.CXXModules.json` file may like as follows.
```json
{
  "modules": {
    "bar": "bazel-out/k8-fastbuild/bin/tmp2/_objs/foobar/1/bar.pcm",
    "foo": "bazel-out/k8-fastbuild/bin/tmp2/_objs/foobar/1/foo.pcm"
  },
  "usages": {
    "foo": [
      "bar"
    ]
  }
}
```
The collection process is illustarted as follows.
```
              ┌────────┐                  ┌────────┐                  ┌────────┐
              │foo.cppm│                  │bar.cppm│                  │main.cc │
              └────┬───┘                  └────┬───┘                  └────┬───┘
c++20-deps-scanning│                           │                           │
                   │                           │                           │
              ┌────▼───┐                  ┌────▼───┐                  ┌────▼───┐
              │foo.ddi │                  │bar.ddi │                  │main.ddi│
              └────┬───┘                  └────┬───┘                  └────┬───┘
                   │                           │                           │
                   └───────────────────────────┼───────────────────────────┘
                                               │Cpp20ModulesInfoAction
                                      ┌────────▼───────────┐
                                      │demo.CXXModules.json│
                                      └────────────────────┘
```

## Using C++20 Modules dependencies

Prior to the incorporation of C++20 Modules, the compilation of C++ source files solely necessitated the inclusion of relevant header files. However, with the integration of C++20 Modules, the compilation process now mandates the corresponding BMI files. These imperative BMI files are indicated to the compiler through the utilization of the `-fmodule-file=<module-name>=<path/to/BMI>` option. This option serves to apprise the compiler of the exact location of the BMI file associated with each module name upon which the source file is dependent.

After conduction the scanning and collection phases to identify dependencies in C++20 Modules, the retrieval of specific BMI files that a given source file depends on can be accomplished. This retrieval process involves the utilization of two interal components: the `.ddi` file and the `.CXXModules.json` file. The `.ddi` file provides essential information pertaining to the module names upon which the source file relies, while the `.CXXModules.json` file facilitates the corrlation between module names and their corresponding BMI files.
The process of obtaining the dependent BMI files for a specific C++ source file is as follows.

- get direct dependent module name from the scan result of the source file
- get all known C++20 Modules information
- use topological sorting to traverse the dependency modules of the direct dependent module to obtain all indirect dependency modules

Finally, the options related to the modules `-fmodule-file=<module-name>=<path/to/BMI>` are written into the compiler's response file, which has the `.modmap` file extension. Additionally, to facilitate input discovery and obtain input BMI files, an auxiliary file with file extension `.modmap.input` is generated. It only records the paths of the BMI files, with each path being recorded on a separate line.

The entire process is accomplished through the newly added `Cpp20ModuleDepMapAction`.

In our three files example, the final information stored in `.modmap` file may like as follows.

- the `bar.modmap` is empty
- the `foo.modmap` only has one line
```
-fmodule-file=bar=bazel-out/k8-fastbuild/bin/tmp2/_objs/foobar/1/bar.pcm
```
the `foo.modmap.input` is
```
bazel-out/k8-fastbuild/bin/tmp2/_objs/foobar/1/bar.pcm
```

- the `main.modmap`has two lines, due to indirect dependencies are required here.
```
-fmodule-file=bar=bazel-out/k8-fastbuild/bin/tmp2/_objs/foobar/1/bar.pcm
-fmodule-file=foo=bazel-out/k8-fastbuild/bin/tmp2/_objs/foobar/1/foo.pcm
```
the `main.modmap.input` is 
```
bazel-out/k8-fastbuild/bin/tmp2/_objs/foobar/1/bar.pcm
bazel-out/k8-fastbuild/bin/tmp2/_objs/foobar/1/foo.pcm
```
The `.modmap` file construction process is illustarted as follows. (ignore `bar.cppm` and `main.cc` to keep clean)
```
┌────────┐
│foo.cppm│
└────┬───┘
     │
     │
┌────▼───┐          ┌────────────────────┐
│foo.ddi │          │demo.CXXModules.json│
└────┬───┘          └─────────┬──────────┘
     │                        │
     └──────────┬─────────────┘
                │
                │Cpp20ModuleDepMapAction
      ┌─────────┴─────────────┐
      │                       │
      │                       │
┌─────▼────┐        ┌─────────▼──────┐
│foo.modmap│        │foo.modmap.input│
└──────────┘        └────────────────┘
```

## Handling C++20 Modules dynamic dependencies

In Bazel, a build consists of three phases: loading, analysis and execution. The action graph is generated on analysis phase. However, the dependencies of a source file using C++20 Modules cannot be determined during the analysis phase because the external command `clang-scan-deps` cannot be execute and thus the dependency information cannot be obtained. To solve this problem, the `clang-scan-deps` needs to be executed during the execution phase, and the support for dynamic dependencies is requried. The method in Bazel to implement dynamic dependencies is through the Skyframe Restart Mechanism.

Skyframe is a framework that performs parallel evaluation of dependency graphs. Each node in the graph corresponds with the evaluation of a SkyFunction with a SkyKey specifying its parameters and SkyValue specifying its result. The computational model is such that a SkyFunction may lookup SkyValues by SkyKey, triggering recursive, parallel evaluation of additional SkyFunctions. Instead of blocking, which would tie up a thread, when a requested SkyValue is not yet ready because some subgraph of computation is incomplete, the requesting SkyFunction observes a null getValue response and should return null instead of a SkyValue, signaling that it is incomplete due to missing inputs. Skyframe _restarts_ the SkyFunctions when all previously requested SkyValues become available.

Whenever a SkyFunction requests a dependency that is unavailable, getValue() will return null. The function should then yield control back to Skyframe by itself returning null. At some later point, Skyframe will evaluate the unavailable dependency, then restart the function from the beginning — only this time the getValue() call will succeed with a non-null result.

In Bazel, the compilation of C++ source files is handled by the action `CppCompileAction`. When constructiong the `CppCompileAction` for a specific C++ source file, we may not know which BMI files the source file depends on. Therefore, we prepare all the BMI files in advance. At this stage, these BMI files serve as placeholder and are not generated yet. In the context of using C++20 Modules dependencies, we obtain `.modmap` and `.modmap.input` files. Before executing the `CppCompileAction`, the contents of the `.modmap.input` file are read using Action Input Discover. Based on the information retrieved, the actual BMI files that the source file depends on are determined. These truly dependent BMI files are then provided as inputs to the `CppCompileAction`, and the compiler consumes these dependent BMI files using the `.modmap` reponse file. It's worth mentioning that BMI files are generated by another `CppCompileAction`, but in this case, the source file of the `CppCompileAction` is the C++20 Module Interface file. The generation of BMI files will be discussed in the next subsection.

In our three files example, when constructing the `CppCompileAction` for `bar.cppm`, the BMI files are `foo.pcm` and `bar.pcm`. Since `bar.cppm` does not depend on anything, its `bar.modmap` and `bar.modmap.input` files are emtpy. Therefore, the final inputs for the `CppCompileAction` consist only of the source file `bar.cppm`. In reality, there may be other files as well, such as the compiler toolchain, header files, and the `.modmap` and `.modmap.input` file that are required by all actions, but let's ignore those for simplicity here.

When constructing the `CppCompileAction` for `foo.cppm`, the BMI files are also `foo.pcm` and `bar.pcm`. Since the module `foo` depends on the module `bar`, the corresponding `foo.modmap` and `foo.modmap.input` record information related to the module `bar`, as shown in the example of subsection using C++20 Modules dependencies. Therefore, the final inputs for the `CppCompileAction` are `foo.cppm` and `bar.pcm`. Similary, the final inputs for the `CppCompileActoin` of `main.cc` are `main.cc`, `foo.pcm` and `bar.pcm`.

During compilaton, if `bar.cppm` is compiled first and then `foo.cppm`, everything works fine. However, if `foo.cppm` is compiled first, it will be noticed that `bar.pcm` is missing. At this point, the Skyframe Restart Mechanism is triggered, which postpones the compilation of `foo.cppm` until after `bar.cppm`has been compiled. This ensures that the compilation is completed in the correct topological order.

In general, the BMI files are prepared in advance, and before the execution of the `CppCompileAction`, only the BMI files that the action actually depends on are retained through Action Input Discover. During the execution of the `CppCompileAction`, the presence of the required BMI files is ensured using the Skyframe Restart Mechanism.

## Compiling C++20 Modules with two-phase compilation

In Clang, there are two methods to generate BMI. One is by using the `-fmodule-output` option to generate BMI as a by-product of the compilation, which is called one-phase compilation. In our three files example, the `foo` and `bar` modules can be compiled with the following commands.
```bash
clang++ -std=c++20 -fmodule-output=bar.pcm -c bar.cppm -o bar.o
clang++ -std=c++20 -fmodule-file=bar=bar.pcm -fmodule-output=foo.pcm -c foo.cppm -o foo.o
```
The `-fmodule-file=bar=bar.pcm` option here is used to specify the dependent BMI `bar`.
The compilation process is illustarted as follows.
```
         ┌────────┐
         │bar.cppm│
         └───┬────┘
             │
             │-fmodule-output=bar.cpm
             │
    ┌────────┴─────────┐
    │                  │
┌───▼────┐        ┌────▼───┐          ┌────────┐
│bar.o   │        │bar.pcm │          │foo.cppm│
└────────┘        └────┬───┘          └───┬────┘
                       │                  │
                       └────────┬─────────┘
                                │
                                │-fmodule-file=bar=bar.pcm
                                │-fmodule-output=foo.pcm
                                │
                      ┌─────────┴────────┐
                      │                  │
                  ┌───▼────┐        ┌────▼───┐
                  │foo.o   │        │foo.pcm │
                  └────────┘        └────────┘
```
The other method is by using the `--precompile` option to compile the source file into an object file in two steps, which is called two-phase compilation. 
The `foo` and `bar` modules can be compiled with the following commands.
```bash
clang++ -std=c++20 --precompile -c bar.cppm -o bar.pcm
clang++ -std=c++20 -c bar.pcm -o bar.o
clang++ -std=c++20 --precompile -fmodule-file=bar=bar.pcm -c foo.cppm -o foo.pcm
clang++ -std=c++20 -fmodule-file=bar=bar.pcm -c foo.pcm -o foo.o
```
The compilation process is illustarted as follows.
```bash
        ┌────────┐               ┌────────┐
        │bar.cppm│               │foo.cppm│
        └────┬───┘               └───┬────┘
--precompile │                       │ --precompile
             │                       │
        ┌────▼───┐                   │
        │bar.pcm ├───────────────────┤ -fmodule-file=bar=bar.pcm
        └────┬───┘                   │
             │                       │
             │                       │
        ┌────▼───┐               ┌───▼────┐
        │bar.o   │               │foo.pcm │
        └────────┘               └───┬────┘
                                     │
                                     │
                                 ┌───▼────┐
                                 │foo.o   │
                                 └────────┘
```
In our solution, the two-phase compilation is adopted. The advantage of two-phase compilation is that it has the potential to compile faster due to higher parallelism. 

After the introduction of C++20 Modules, different source files need to be handled separately. In the case of C++20 Module Interface Unit, a two-stage compilation process is employed, where BMI files are generated in the first stage and Object Files are generated in the second stage. The benefit of using a two-stage compilation process is that concurrency during compilation can be improved. For C++20 Module Implementation Units and other Translation Uints, the source files are compiled into Object Files only. Apart from the handling of dependencies on C++20 Modules dependencies, the overall process remains the same as in the absence of C++20 Modules. During compilation, the `CppCompileAction` still use s the action name `c++-compile`. Therefore, we only discuss the scenario where the source file is C++20 Module Interface Unit.

To handle C++20 Module Interface Unit, the approach used in Bazel's existing Clang Module handling is referenced, and the `CppCompileAction`is still reused. A new action name, `c++20-module-compile` (similar to `c++-module-compile` for Clang Module), is introduced to compile C++20 Module Interface Units into BMI files. Additionally, a new action name, `c++20-module-codegen` (similar to `c++-module-codegen` for Clang Module), is added to compile BMI files into Object Files. For action inputs, both actions require the `.d` file,  the `.modmap` and `.modmap.input` files, as well as all the BMI files. The `.d` file is used for input pruning, and all the BMI files are placeholder and are not generated yet.

In our three files example, when compile `foo.cppm`, the first step is to use a `CppCompileAction` to compile it into `foo.pcm`. The inputs for this action are `foo.cppm`, `bar.cpm`, `foo.modmap`, `foo.modmap.input` and `foo.d`. The action name for this step is `c++20-module-compile`. It is also necessary to include the releant flags for headers files (e.g. `-I`).

The first step of the compilation process is shown in the following diagram. The direct dependencies required for the compilation are depicted, while ignoring the `foo.modmap.input` and the specific compilation detals of `bar.cppm` to `bar.cpm`.
```
                                              ┌────────────────────┐
                                              │demo.CXXModules.json│
                                              └─────┬──────────────┘
                                                    │
                                                    │
                                  ┌───────┐         │          ┌───────────┐
                               ┌─►│foo.ddi├─────────┤          │ bar.cppm  │
                               │  └───────┘         │          └─────┬─────┘
                               │                    │                │
                               │                    │                │
                               │                    │                │
                               │                    │                │
┌────────┐ c++20-deps-scanning │  ┌───────┐   ┌─────▼────┐      ┌────▼─────┐
│foo.cppm├─────────────────────┴─►│foo.d  │   │foo.modmap│      │ bar.pcm  │
└───┬────┘                        └──┬────┘   └────┬─────┘      └────┬─────┘
    │                                │             │                 │
    │                                │             │                 │
    └────────────────────────────────┴─────────────┼─────────────────┘
                                                   │c++20-module-compile
                                              ┌────▼─────┐
                                              │ foo.pcm  │
                                              └──────────┘
```
In the second step, a `CppCompileAction` is used to compile `foo.pcm` into `foo.o`. The inputs for this action are `foo.pcm`, `bar.pcm`, `foo.modmap`, `foo.modmap.input` and `foo.d`. The action name for this step is `c++20-module-codegen`. No specific flags related to header files are required.

The following diagram illustartes the second step of the compilation process. The direct dependencies required for the compilation are depicted, while ignoring the `foo.modmap.input` file and the specific compilation details of `bar.cppm` to `bar.cpm` and `foo.cppm` to `foo.pcm`.
```
                                              ┌────────────────────┐
                                              │demo.CXXModules.json│
                                              └─────┬──────────────┘
                                                    │
                                                    │
                                  ┌───────┐         │          ┌───────────┐
                               ┌─►│foo.ddi├─────────┤          │ bar.cppm  │
                               │  └───────┘         │          └─────┬─────┘
                               │                    │                │
                               │                    │                │
                               │                    │                │
                               │                    │                │
┌────────┐ c++20-deps-scanning │  ┌───────┐   ┌─────▼────┐      ┌────▼─────┐
│foo.cppm├─────────────────────┴─►│foo.d  │   │foo.modmap│      │ bar.pcm  │
└───┬────┘                        └──┬────┘   └────┬─────┘      └────┬─────┘
    │c++20-module-compile            │             │                 │
┌───▼───┐                            │             │                 │
│foo.pcm├────────────────────────────┴─────────────┼─────────────────┘
└───────┘                                          │c++20-module-codegen
                                              ┌────▼─────┐
                                              │ foo.o    │
                                              └──────────┘
```
The two actions share the same inputs: `bar.pcm`, `foo.modmap`, `foo.modmap.input` and `foo.d`. The `bar.pcm` is obtained from the first step's `CppCompileAction` when compiling `bar.cppm`with the action name `c++20-module-compile`. The `foo.modmap` and `foo.modmap.input` is derived from the `Cpp20ModuleDepMapAction`. The `foo.d` is obtained from the `CppCompileAction` with the action name `c++20-deps-scanning`. When compiling `foo.cppm`, there is no need to wait for `bar.o` to be generated. Only the completion of compiling `bar.cppm` is required. This allows simultaneous execution of compiling `foo.cppm` into `foo.pcm` and compiling `bar.pcm` into `bar.o`. thereby improving concurrency during compilation.

## Handing inter-library dependencies for C++20 Modules
In Bazel, the information transfer between libraries is done using the `CcCompilationContext` class, which contains information related to header files and macro definitions. With the use of C++20 Modules, it is necessary to enhance the `CcCompilationContext` class to include information about C++20 Modules BMI files and `.CXX20Modules.json` files. 

In Collecting C++20 Modules dependencies subsection, only the C++20 Modules information of the current library is recorded in the `.CXX20Modules.json` file. However, when considering inter-library dependencies, it is necessary to include all the C++20 Modules information from the dependent libraries in the `.CXX20Modules.json` file of the current library as well. By consolidating the information, it becomes easier to access and utilize the C++20 Modules information within the current library while avoiding multiple searches across different `.CXXModules.json` files. 

Let's continue using the previous three files as examples. In this case, the Bazel configuration needs to be modified to build three libraries instead of just one. The new configuration is as follows.
```python
cc_library(
    name="bar",
    module_interfaces=["bar.cppm"],
    copts=["-std=c++20"]
)
```
```python
cc_library(
    name="foo",
    module_interfaces=["foo.cppm"],
    copts=["-std=c++20"],
    deps = [":bar"]
)
```
```python
cc_library(
    name="demo",
    srcs=["main.cc"],
    copts=["-std=c++20"],
    deps = [":foo"]
)
```
Now there are three libraries, each with its own `.CXX20Modules` file. They are named `bar.CXX20Modules.json`, `foo.CXX20Modules.json` and `demo.CXX20Modules.json`.
The content of `bar.CXX20Modules.json` is as follows.
```python
{
  "modules": {
    "bar": "bazel-out/k8-fastbuild/bin/tmp2/_objs/foobar/1/bar.pcm",
  },
  "usages": {
  }
}
```
The content of `foo.CXX20Modules.json` is as follows.
```python
{
  "modules": {
    "bar": "bazel-out/k8-fastbuild/bin/tmp2/_objs/foobar/1/bar.pcm",
    "foo": "bazel-out/k8-fastbuild/bin/tmp2/_objs/foobar/1/foo.pcm"
  },
  "usages": {
    "foo": [
      "bar"
    ]
  }
}
```
The content of `demo.CXX20Modules.json` is as follows.
```python
{
  "modules": {
    "bar": "bazel-out/k8-fastbuild/bin/tmp2/_objs/foobar/1/bar.pcm",
    "foo": "bazel-out/k8-fastbuild/bin/tmp2/_objs/foobar/1/foo.pcm"
  },
  "usages": {
    "foo": [
      "bar"
    ]
  }
}
```
The collection process with inter-library dependencies is illustarted as follows.
```
                                                ┌─────────┐
                                                │ bar.cppm│
                                                └────┬────┘
                                                     │c++20-deps-scanning
                                                ┌────▼────┐
                                                │ bar.ddi │
                                                └────┬────┘
                                                     │Cpp20ModulesInfoAction
                                                     │
              ┌────────┐                   ┌─────────▼──────────┐       ┌─────────┐
              │foo.cppm│                   │ bar.CXXModules.json│       │main.cc  │
              └────┬───┘                   └─────────┬──────────┘       └────┬────┘
c++20-deps-scanning│                                 │                       │c++20-deps-scanning
              ┌────▼───┐                             │                  ┌────▼────┐
              │foo.ddi │                             │                  │main.ddi │
              └────┬───┘                             │                  └────┬────┘
                   │                                 │                       │
                   └─────────────────┬───────────────┘                       │
                                     │Cpp20ModulesInfoAction                 │
                            ┌────────▼───────────┐                           │
                            │ foo.CXXModules.json│                           │
                            └────────┬───────────┘                           │
                                     │                                       │
                                     │                                       │
                                     └─────────────────────────┬─────────────┘
                                                               │Cpp20ModulesInfoAction
                                                      ┌────────▼───────────┐
                                                      │demo.CXXModules.json│
                                                      └────────────────────┘
```

## Changes to rules' attributes
For `cc_library`and `cc_binary`, the additional attribute `module_interfaces` is added instead of resuing the existing attribute `srcs`. The attribute `module_interfaces` indicates that the source files here are module interface units. CMake also has a similar design, and it achieves this by specifying `FILE_SET`. In our three files example, the CMake configuration may be writen as follows.
```cmake
add_library(demo main.cc)
target_sources(demo PUBLIC
        FILE_SET cxx_modules TYPE CXX_MODULES FILES
        foo.cppm bar.cppm
)
```
The reason is that distinguishing whether a source file is a Module Interface solely based on its file extension is not feasible.  For C++20 Module Inteface Unit, it is necessary to perform precompilation to generate BMI files with addtional flags. In this process, the `srcs` attribute cannot be directly reused as it is not possible to differentiate which files are C++20 Module Interface Unit. This is because the file extension for C++20 Module Interface Unit are implementation-specific, such using `.cppm` for clang and `.ixx` for MSVC. Considering generality, it is not of concern what the file extension is. Any file included as a value of the `module_intefaces` attribute is regarded as a C++20 Module Interface Unit.

The following writing styles are all valid, as long as they are your liking.
```python
cc_library(
    name="foo",
    module_interfaces=["foo.cppm", "foo.cc", "foo.cpp", "foo.cxx", "foo.ixx"],
    copts=["-std=c++20"],
)
```
# Compatibility

- rules_cc

The compatibility issue with `rules_cc` arises from the need to support C++20 Modules. To enable this support, a series of new action names have been introduced, namely `c++20-deps-scanning`, `c++20-module-compile`, and `c++20-module-codegen`. However, the current version of rules_cc has not been updated to include these changes. As a result, it is not possible to build C++20 Modules code correctly using rules_cc.

If a commit updates rules_cc to include these new action names, it may lead to compatibility issues with older versions of Bazel. This is because the older versions of Bazel do not recognize these new action names, making it impossible to use the updated rules_cc with the older Bazel versions.

- feature `cpp20_module`

Simultaneous support for both the usage and non-usage of C++20 Modules can be achieved. To utilize C++20 Modules, the `cpp20_module` feature needs to be enabled. When this feature is activated, the build process will incorporate the necessary modifications and actions required for C++20 Modules. Conversely, if the feature is disabled, the build process will remain unaffected and function similarly to previous versions of Bazel, without any supplementary ramifications.

# Future Plan
At present, all the work is being conducted based on Clang, therefore, Clang is the only supported compiler. Future efforts will include the support for GCC and MSVC.

# References

- [import CMake; C++20 Modules](https://www.kitware.com/import-cmake-c20-modules/)
- [Ninja Dynamic Dependencies](https://ninja-build.org/manual.html#ref_dyndep)
- [How to Use C++20 Modules with Bazel and Clang](https://buildingblock.ai/cpp20-modules-bazel)
