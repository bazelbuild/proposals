---
created: 2018-10-09
last updated: 2018-10-09
status: In Review
reviewers:
  - gregce
title: Platform Inheritance
authors:
  - katre
discussion thread: https://groups.google.com/forum/#!topic/bazel-dev/wKacIE4MIRM
---

# Abstract

Heavy users of [Bazel platforms](https://docs.bazel.build/versions/master/platforms.html) frequently have many platforms that are very similar, with a small number of constraint values that are different. This design proposal offers a way for platforms to inherit constraint values and remote execution properties from a "parent" platform.


# Background

Currently, platforms in Bazel (declared with the [`platform` rule](https://docs.bazel.build/versions/master/be/platform.html#platform) do not have any form of inheritance. If users want to declare a platform that is semantically "other platform, with added constraint", they have to copy and paste the constraint values from the first platform to the second, and keep them in sync manually.

The ability to define a parent for a platform would be useful for the execution platforms defined by remote execution systems, which are frequently a small set of base platforms, and then additional platforms which take the base and add extra capabilities, such as network access or extra memory. See the GitHub issue at: https://github.com/bazelbuild/bazel/issues/6218.


# Proposal

A platform that inherits from another will receive all of its constraint values, except for those that conflict with the constraints defined directly on the child platform. Because of the problems of merging constraint values and the possibilities for conflicts, platforms have a single parent that they inherit from, instead of a set of other platforms that they depend on.

A new attribute, `parents`, will be added to the existing `platform` rule to set the base which the new platform inherits from. The attribute will be a list of labels, but it will be an error to have more than one value. This will, however, allow for later migration to allowing multiple inheritance of platforms. All constraint values set directly on the new platform will override values for the same constraint setting from the parent platform.

## Example

Example:
```
constraint_setting(name = "fruit")
constraint_value(name = "banana", constraint_setting = ":fruit")
constraint_value(name = "apple", constraint_setting = ":fruit")
constraint_setting(name = "suit")
constraint_value(name = "hearts", constraint_setting = ":suit")
constraint_value(name = "clubs", constraint_setting = ":suit")

platform(
    name = "base",
    constraint_values = [
      ":banana",
      ":hearts",
    ],
)

platform(
    name = "extend",
    parents = [":base"],
    constraint_values = [
      ":clubs",
    ],
)
```


In this example, the `extend` platform will have the constraint values `:banana` (inherited from the parent platform) and `:clubs` (which overrides the constraint `:hearts` set on the parent platform).


## Remote Execution Properties

The
[`remote_execution_properties`](https://docs.bazel.build/versions/master/be/platform.html#platform.remote_execution_properties)
attribute of a platform also needs to be inherited from the parent, but the
merging behavior is slightly more complicated because the attribute is a single
string.

If the parent does not set the remote execution properties attribute, the value from the child will be used. If the parent is set but not the child, the parent will be used. If both are set, the child value will be used, but the literal string `{PARENT_REMOTE_EXECUTION_PROPERTIES}` will be replaced with the parent's attribute value, allowing the child to contain it. It is the responsibility of whatever system consumes this property to handle any duplicated data that is present: Bazel does not consume this data directly.


# Implementation

The main implementation work for this will be to add the functionality to the [PlatformInfo.Builder](https://source.bazel.build/bazel/+/master:src/main/java/com/google/devtools/build/lib/analysis/platform/PlatformInfo.java;l=104?q=PlatformInfo) to set the parent platform and handle the merge of the constraint values and the remote execution properties. To merge the values, functionality will be added to the [ConstraintCollection](https://source.bazel.build/bazel/+/master:src/main/java/com/google/devtools/build/lib/analysis/platform/ConstraintCollection.java?q=ConstraintCollection) to check a parent constraint collection if the current collection does not have a value.


# Backward-compatibility

As the `parent` attribute is a new addition to the `platform` rule, there is no backwards compatibility concern.

