# Bazel Proposals and Design Documents

This is an index of all design documents and proposals for [Bazel](https://bazel.build).

These documents were publicly sent for review on the [bazel-discuss](https://groups.google.com/forum/#!forum/bazel-discuss) mailing list.


# Index

### Implemented

| Last updated | Title                                                                                                                                      | Author(s) alias   | Category              |
|--------------|--------------------------------------------------------------------------------------------------------------------------------------------|-------------------|-----------------------|
|   2018-04-26 | [Per-Rule Execution Platform Constraints](https://docs.google.com/document/d/1p1J2ktWTpoKvNATjC6U29vhz1_-Dgbe9YRPr5wisfzY)                 | jcater            | Configurability       |
|   2017-03-03 | [Label-keyed String Dictionary Type for Build Attributes](https://bazel.build/designs/2017/03/03/label-keyed-string-dict-type.html)        | mstaib            | Configurability       |
|   2016-10-18 | [Invalidation of remote repositories](https://bazel.build/designs/2016/10/18/repository-invalidation.html)                                 | dmarting          | External Repositories |
|   2016-10-11 | [Distribution Artifact for Bazel](https://bazel.build/designs/2016/10/11/distribution-artifact.html)                                       | aehlig            | Release               |
|   2016-09-30 | [Central cache for external repositories](https://bazel.build/designs/2016/09/30/repository-cache.html)                                    | jingwen           | External Repositories |
|   2016-09-05 | [Building Python on Windows](https://bazel.build/designs/2016/09/05/build-python-on-windows.html)                                          | pcloudy           | Python, Windows       |
|   2016-06-21 | [Specifying environment variables](https://bazel.build/designs/2016/06/21/environment.html)                                                | aehlig            | Bazel                 |
|   2016-06-02 | [Sandboxing](https://bazel.build/designs/2016/06/02/sandboxing.html)                                                                       | philwo            | Bazel                 |
|   2016-05-26 | [Implementing Beautiful Error Messages (Loading Phase)](https://bazel.build/designs/2016/05/26/implementing-beautiful-error-messages.html) | laurentlb         | Bazel                 |
|   2016-05-23 | [Beautiful error messages](https://bazel.build/designs/2016/05/23/beautiful-error-messages.html)                                           | laurentlb         | Bazel                 |
|   2016-04-18 | [Parameterized Skylark Aspects](https://bazel.build/designs/skylark/parameterized-aspects.html)                                            | dslomov, lindleyf | Skylark               |
|   2016-02-16 | [Generating C++ crosstool with a Skylark Remote Repository](https://bazel.build/designs/2016/02/16/cpp-autoconf.html)                      | dmarting          | Toolchains            |
|   2015-07-02 | [Skylark Remote Repositories](https://bazel.build/designs/2015/07/02/skylark-remote-repositories.html)                                     | dmarting          | External repositories |

### Approved

| Last updated | Title                                                                                                              | Author(s) alias    | Category |
|--------------|--------------------------------------------------------------------------------------------------------------------|--------------------|----------|
|   2018-04-19 | [Separating Build API from Bazel](https://docs.google.com/document/d/1UDEpjP_qWQRYsPRvx7TOsdB8J4o5khfhzGcWplW7zzI) | cparsons           | Skylark  |
|   2017-07-13 | [Improved Command Line Reporting](https://bazel.build/designs/2017/07/13/improved-command-line-reporting.html)     | ccalvarin          | Bazel    |
|   2016-08-04 | [Extensibility For Native Rules](https://bazel.build/designs/2016/08/04/extensibility-for-native-rules.html)       | dslomov            | Skylark  |
|   2016-06-06 | [Declared Providers](https://bazel.build/designs/skylark/declared-providers.html)                                  | dslomov, laurentlb | Skylark  |

### Under review

| Last updated | Title                                                                                                   | Author(s) alias | Category  |
|--------------|---------------------------------------------------------------------------------------------------------|-----------------|-----------|
|   2018-05-23 | [Crosstool in Skylark](https://docs.google.com/document/d/1Nqf16jqDGWSrPp4VuRxh0iNnVBoAXsO0meDH69J9xoc) | hlopko          | C++       |
|   2018-04-20 | [Bazel Rules Curation](https://docs.google.com/document/d/1oYQ-cqmqrpVE02rphobn4F_Q-lqvch4IiUlqEy9q2Fs) | laurentlb       | Community |

### Draft

| Last updated | Title                                                                                                                           | Author(s) alias | Category |
|--------------|---------------------------------------------------------------------------------------------------------------------------------|-----------------|----------|
|   2018-03-23 | [Moving Skylark out of Bazel](https://docs.google.com/document/d/15ysfoMXRqZDdz0OOY1mtpeWd7LjDnXKl4fOVSLGACAY/edit?usp=sharing) | brandjon        | Skylark  |
|   2018-03-05 | [Output Map Madness](https://docs.google.com/document/d/1ic9lJPn-0VqgKcqSbclVWwYDW2eiV-9k6ZUK_xE6H5E/edit)                      | brandjon        | Skylark  |
|   2016-07-25 | [Saner Skylark Sets](https://bazel.build/designs/skylark/saner-skylark-sets.html)                                               | dslomov         | Skylark  |

### WIP

| Last updated | Title                                                                                                                           | Author(s) alias | Category              |
|--------------|---------------------------------------------------------------------------------------------------------------------------------|-----------------|-----------------------|
|   2018-06-28 | [Shrinking the Bazel binary](https://docs.google.com/document/d/1Igmv-2GfXkoVFWTXvBYPeniQom8nLAwzqzridDlBIS4/edit)              | buchgr, twerth  | Bazel                 |
|   2018-06-20 | [C++ rules skylark migration plan](https://docs.google.com/document/d/1Adqu7--verca4gCh3ZdnVMjjCz4VzurHxKEZAh9u03E)             | hlopko          | C++                   |
|   2018-06-18 | [Name resolution](https://docs.google.com/document/d/1-atlB3j59XqKTDIE8ibBDT_cVAp_QGFQoDwdm3ITVG0)                              | laurentlb       | Skylark               |
|   2018-06-15 | [Skylark API to the C++ toolchain](https://docs.google.com/document/d/1M8JA7kzZnWpLZ3WEX9rp6k2u_nlwE8smsHYgVTSSJ9k)             | hlopko          | C++, Skylark          |
|   2018-05-31 | [Skylark Build Configuration](https://docs.google.com/document/d/1vc8v-kXjvgZOdQdnxPTaV0rrLxtP2XwnD2tAZlYJOqw)                  | gregce          | Configurability       |
|   2018-04-26 | [Garbage Collection for the Repository Cache](https://docs.google.com/document/d/1IuciCmnY0Z9naciq10G2zb94mCb9xfpFLh5ZIgMcPqU/) | aehlig          | External Repositories |

### Dropped

| Last updated | Title                                                                                                           | Author(s) alias | Category              |
|--------------|-----------------------------------------------------------------------------------------------------------------|-----------------|-----------------------|
|   2018-06-14 | [Platforms and Configurations](https://docs.google.com/document/d/1XyGTHgcfI9aL-JHpC-xMztEPZQauXhFzBZXTKMbgFRw) | jcater          | C++, Skylark          |
|   2016-09-19 | [Recursive WORKSPACE file parsing](https://bazel.build/designs/2016/09/19/recursive-ws-parsing.html)            | kchodorow       | External Repositories |
|   2015-03-06 | [bazel init a.k.a ./configure for Bazel](https://bazel.build/designs/2015/03/06/bazel-init.html)                | dmarting        | Bazel                 |

