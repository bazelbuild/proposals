---
# Enabling global exec\_groups 

**Status**: Draft

**Author**: [susinmotion](https://github.com/susinmotion)

**Reviewers:** [katre](https://github.com/katre)

**Date:** 2022-06-28


# Introduction

Sometimes, rules create multiple actions that have different platform restrictions from each other. For example, one action might use a Mac-only tool, but others can run on any platform. Such rules can currently use exec\_groups to express their platform constraints by declaring  a set of exec\_groups at the rule level, and using them at the action level. 

This is burdensome for rule owners for the following reasons:



*   exec\_group syntax is not straightforward
*   Rules that use helper methods that create actions need to declare the exec\_groups that those actions use, but it’s not always obvious which exec\_groups those are
*   exec\_group names aren’t guaranteed to be unique, so it’s possible to overwrite one exec\_group with another that does something different
*   There’s no good solution for cases where multiple helper methods are used

# Proposal

In order to make it as easy as possible for rule owners to expose rules’ platform requirements, we can expose standard global exec\_groups representing the most common execution constraints.


## **API changes**


### Current:


```
my_rule = rule(...
    implementation = _my_rule_impl,
            exec_groups = { "my_exec_group" : 
        exec_group(
            exec_compatible_with = ["//path_to/my:constraint"]}
          )

def _my_rule_impl(ctx):
	…
	ctx.actions.run(...
		exec_group = "my_exec_group"
	)
```



### Proposed


```
my_rule = rule(...) // no need to declare exec_groups here, as long as only standard global exec groups are used

def _my_rule_impl(ctx):
	…
	ctx.actions.run(...
		exec_group = "global.mac"
	)
```



## Implementation

Define standard exec\_groups reflecting the most common platform constraints:



*   exec\_compatible\_with = [“@bazel\_platforms/os:linux”]
*   exec\_compatible\_with = [“@bazel\_platforms/os:macos”]
*   exec\_compatible\_with = []

These should use a standard and unique naming scheme (e.g. “global.mac” instead of “mac”) to avoid conflicting with existing custom exec\_groups. 

Set a default value for exec\_groups in [StarlarkRuleContext](https://github.com/bazelbuild/bazel/blob/eeb2e04e52cfad165a1bde33ce2d83a392f53d00/src/main/java/com/google/devtools/build/lib/analysis/starlark/StarlarkRuleContext.java#L750) that includes these standard exec\_groups in addition to the defaults exposed by the toolchains. This should work with declaring custom exec\_groups since we [don’t overwrite the default ones](https://github.com/bazelbuild/bazel/blob/master/src/main/java/com/google/devtools/build/lib/packages/RuleClass.java#L1528-L1534) unless they have the same name.


## Limitations

Using a global exec\_group at action creation time doesn’t configure prerequisite tools “for free”. Authors must still use cfg = config.exec\_group(“exec\_group\_name”) to keep tool configuration in sync unless a default constraint is declared at the rule definition level using “exec\_compatible\_with”.

Using an unstructured namespace for exec\_groups makes it possible for an exec\_group key used in an action to refer to multiple different definitions. This isn’t ideal, but in the interest of not adding complexity and making the minimal set of changes to make the API more usable, I won’t address the naming issue in this proposal.


# Alternatives 
Keep this change internal-only and don’t expose global exec\_groups to Bazel.
