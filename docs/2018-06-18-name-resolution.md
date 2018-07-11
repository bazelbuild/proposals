# Name resolution in Skylark

> Author: [laurentlb](https://github.com/laurentlb/)<br/>
> Created: 2018-06-18<br/>
> Status: Approved, to be implemented.<br/>
> Reviewers: [dslomov](https://github.com/dslomov/), [alandonovan](https://github.com/alandonovan/)<br/>

This document lists the differences between Bazel implementation of Skylark and
the [language specification](https://docs.bazel.build/versions/master/skylark/spec.html),
in relation to name resolution, shadowing, and rebinding.


## Shadowing of built-ins

The specification mentions there are 4 blocks:

*   Universe block (the built-ins)
*   Module block (the global variables and loaded values)
*   Function block (the parameters and local variables)
*   Comprehension block (loop variables in a comprehension)

A definition in a block may shadow any definition from a parent block.
(Comprehension blocks may be nested.)

In the current implementation, a variable in the module block cannot shadow a
variable in a universe block. Hence Bazel reports an error for these two
statements:


```python
load(":file.bzl", len="x")  # Error, len is already defined

print = 5  # Error, print is already defined
```


**Resolution**: Allow shadowing, follow the specification.

**Rationale:** Adding a new built-in in the language shouldn't be a breaking
change. A linter might report this as a warning to avoid confusion.


## Scope of a global variable

According to the specification, "if name is bound anywhere within a block, all
uses of the name within the block are treated as references to that binding,
even uses that appear before the binding."

Bazel currently rejects this code (but the specification allows it):


```python
def dbg(x):
  if is_debug:  # static error, is_debug is not defined
    print("-->", x)

is_debug = True

dbg("hello")
```


However, Bazel allows this code (also valid according to the specification):


```python
def dbg(x):
  if is_debug():
    print("-->", x)

def is_debug(): return True

dbg("hello")
```


In the current implementation, global variables are visible from their
definition point to the end of the file. Functions are visible in the entire
file, because it is common for a function to call another function defined later
in the file.

The specification makes no distinction between variables and functions: both are
visible in the entire file. It is simpler and more consistent.

Note that, for both the Bazel implementation and the specification, it is a
runtime error to access a symbol before it has been evaluated:


```python
def dbg(x):
  if is_debug:  # spec: runtime error; current impl: static error
    print("-->", x)

dbg("hello")

is_debug = True
```



```python
def foo(): return 1

foo()
bar()  # runtime error, in both cases

def bar(): return 2
```


**Resolution**: Follow the specification: a global variable is visible in the
entire file. Encourage the use of a linter to catch issues. _This is not a
breaking change for Bazel._


## Scope of a local variable

According to the specification, this code should be valid:


```python
def f():
  if False:
    print(a)
  a = 2
  print(a)
f()
```

Bazel currently throws a static error ("a is not defined"). In Bazel, variables
are visible between the definition point to the end of the function body. In the
specification, variables are visible in the entire block.

In practice, it is very rare to refer to a variable before its definition point
(when it happens, it's usually a programmer mistake), although we might
construct such an example:


```python
def foo(li):
  for i in li:
    if i % 2 == 0:
      print(a)
    else:
      a = i

foo([1, 2, 5, 4])
```


While Bazel rejects the previous example, it allows this one:


```python
def foo(li):
  for i in li:
    if i % 2 != 0:
      a = i
    else:
      print(a)

foo([1, 2, 5, 4])
```

Forcing definitions to precede usage is common in many languages, like C++ and
Java. It improves readability, and allows better completion in an IDE (we know
the set of available variables even if the code is not complete).

However, this restriction doesn't give much guarantee. For consistency with the
previous section, it's probably better to remove it. This check would be better
done in a separate linter (which can for example do control flow analysis).

**Resolution**: Follow the specification: a variable is visible in the entire
block. _This is not a breaking change for Bazel._


## Lexical binding of variables

Bazel name resolution is not truly static.


```python
data = []

def fct(x):
  if x:
    data = []

  data.append(1)
```

The function above will either add an element to the global list, or add it to a
local list, depending on x. This is arguably an implementation bug.

Based on the specification, data should refer only to the local variable. So if
x is false, there will be a runtime error (data is not initialiazed).

**Resolution**: Use static name resolution, follow the specification.

This is a **breaking change** for Bazel.


## Rebinding in BUILD files

According to the specification, "it is a static error to bind a global variable
already explicitly bound in the file."

Bazel implements this check in bzl files, but not in BUILD files for historical
reasons (legacy code should be updated). This should be an error:


```python
ab = 2
ab = 3
```

**Resolution**: Update Bazel, forbid rebinding of global variables in BUILD files.

This is a **breaking change** for Bazel.

## Backward-compatibility

Two sections in this document constitute breaking changes to Bazel:

 * [Lexical binding of variables](#lexical-binding-of-variables) is fixing a bug
   and a surprising behavior (almost every modern language uses lexical
   binding). We also believe very few files rely on this behavior. User impact
   should be low.

* [Rebinding in BUILD files](#rebinding-in-build-files) is making the language
  stricter in BUILD files. We believe the benefits are greater than the
  migration costs. In particular, it improves consistency (difference between
  BUILD and bzl files), increases files readability, and reduces maintenance
  cost in the long-term.


For a smooth rollout, we'll use the
[backward compatibility process](https://docs.bazel.build/versions/master/skylark/backward-compatibility.html).

A linter will also notify the users about it.
