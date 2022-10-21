---
created: 2021-03-06
last updated: 2021-03-06
status: Implemented
reviewers:
  - bergsieker
  - coeuvre
  - EricBurnett
  - larsrc-google
title: "Remote Persistent Workers"
authors:
  - ulfjack
---

# Abstract

This document proposes that Bazel passes information through the existing
[remote execution protocol](https://github.com/bazelbuild/remote-apis) to a
remote execution system such that that system can run actions using persistent
workers. Remote execution systems that do not support this silently ignore the
additional information and execute these actions in the normal way.

This document does not discuss how the remote execution system implements this
feature, e.g., how to find matching persistent worker processes in a potentially
large distributed system.

In our testing, we have achieved 2x improvements in build time for large
remotely executed builds. We expect further speedups with improvement to the
scheduling algorithm.

# Background

Bazel's
[persistent workers](https://github.com/bazelbuild/bazel/blob/master/site/docs/persistent-workers.md)
significantly improve local build times; the original
[blog post](https://blog.bazel.build/2015/12/10/java-workers.html) indicates an
impressive 4x improvement for Java builds. Unfortunately, workers are not
available with remote execution, making it impossible for a remote execution
system to achieve comparable build times at comparable CPU availability. If
there are a significant number of worker-supported actions on the critical path,
then this also applies to the end-to-end build time.

This document describes an way for Bazel to pass enough information through the
existing [remote execution API](https://github.com/bazelbuild/remote-apis) to a
remote execution system such that that system can run actions using persistent
workers. This is backwards compatible with existing remote execution systems,
which silently ignore the additional information and fall back to normal
execution.

In order to use persistent workers, Bazel rules annotate specific actions as 
worker-compatible and also annotate a subset of action inputs as 'tool inputs';
these are files that are required for the persistent worker process. Bazel then
takes the action command line, removes some of the arguments (arguments starting
with '@', '--flagfile' or '-flagfile' (with some additional exceptions), and
appends an additional `--persistent_worker` argument.

Furthermore, Bazel computes a 'worker key' consisting of the names and hashes of
the tool inputs and runfiles (these are implicitly considered tool inputs), the
action's environment variables, the rewritten command line, action mnemonic, as
well as a few internal flags. Actions with equal keys are routed to the same
pool of persistent worker processes for execution. It then uses a simple
protocol over stdin/stdout to send the persistent worker process the parameter
files that were removed earlier, as well as some metadata about the inputs.

As of 2021-03-06, Bazel supports both a binary protobuf and a json protocol, and
also has support for multiplex workers.

# Proposal

In order for a remote system to replicate the steps that Bazel performs, it
requires the same inputs. The environment and command line are already provided
to the remote system. What's missing are the tool inputs and the fact that the
action supports workers.

The remote execution protocol already has a generic 'node properties' field that
can be used to annotate action inputs as tool inputs. In addition, there is a
generic platform definition that can indicate support for workers or multiplex
workers.

In our prototype, we use a node property key of `bazel_tool_input` with an empty
value to indicate that an input file is a tool input. As of 2021-03-06, our
prototype only supports non-multiplex, binary protobuf persistent workers, so
the service assumes that the presence of node properties indicates support for
this.

In addition, our prototype implementation in Bazel sets `persistentWorkerKey`
as a platform option, with the value being the computed worker key. This is used
by the remote execution scheduler to route actions to a matching worker machine.

It would be trivial to extend the prototype to set a `persistentWorkerProtocol`
platform option to indicate the protocol (json or protobuf) as well as a
`persistentWorkerMultiplex` platform option to indicate support for multiplex
workers.

There are two open issues:
- Bazel supports a `--worker_extra_flag` flag which it uses to non-hermetically
  pass flags to persistent workers. These flags could be passed to the remote
  execution system as well.
- The persistent worker protocol is not formally specified and is currently just
  'whatever Bazel implements'. This is not ideal since we'd like multiple
  implementations that are fully compatible.

# Alternatives considered

- We considered not using persistent workers in the remote execution system.
  However, the benefits are very significant.
- We considered using dynamic execution. Dynamic execution automatically decides
  whether to execute actions locally or remotely, and can even do both in
  parallel. However, this has a number of shortcomings:
  - It requires the local machine to be compatible with the remote machines;
    e.g., it cannot be used if the local machine is a Mac and the remote
    execution system runs on Linux (or if it's x86_64 and ARM64). This can also
    an issue if the remote execution system runs actions inside Docker with a
    different OS version or set of tools.
  - It requires a beefy local machine, since the local machine is now again on
    the critical path compared to full remote execution.
  - It interacts badly with 'build without the bytes'.

# Backward-compatibility

Remote execution systems that do not support these node and platform properties
can silently ignore them and execute these actions in the normal way.
