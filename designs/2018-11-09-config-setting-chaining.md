---
created: 2018-11-09
last updated: 2018-11-09
status: Implemented
reviewers:
  - serynth
title: Config Setting Chaining
authors:
  - gregestren
discussion thread: https://groups.google.com/d/topic/bazel-dev/HWBijMHRTrg/discussion
---

# Config Setting Chaining

## Abstract

This document proposes extensions to
[`config_setting`](https://docs.bazel.build/versions/master/be/general.html#config_setting)
that enable `AND` and `OR` chaining, i.e. the ability to express *"this condition
matches if `config_settings` A, B, **and** C are true"* and *"this condition matches
if `config settings` A, B, **o**r C are true"*.

This addresses long-standing user feedback on the awkwardness of expressing
these combinations with the current API.

These extensions will be added to a standard Starlark library rather than core
Bazel.


## Background

[Configurable
attributes](https://docs.bazel.build/versions/master/configurable-attributes.html)
is a powerful Bazel feature that lets rules customize their settings based on how
the build is invoked.

For example, a C++ binary may choose to include ARM-specific dependencies for
builds that target ARM architectures.

The criteria that determine which "choice" a rule should make are modeled with
the
[`config_setting`](https://docs.bazel.build/versions/master/be/general.html#config_setting)
rule:

```python
config_setting(
    name = "if_arm",
    values = {"cpu": "arm"}
)
```

This rule "matches" only for builds that are invoked with `--cpu=arm`. This
means

```python
cc_binary(
    name = "mybinary",
    srcs = ["mybinary.cc"],
    deps = select({
        ":if_arm": [":arm_deps"],
        "//conditions:default": []
  })
)
```

declares a C++ binary that links `":arm_deps"` for `--cpu=arm` builds and no
additional deps otherwise.

`config_setting` is an intentionally simple rule: it matches if every entry in
its `values` attribute matches. This keeps both the model and its implementation
logic simple and adds less runtime overhead than more sophisticated models
like regex parsing.

Experience shows that this simplicity sometimes makes it hard for users to
express what they want. One of the most commonly expressed limitations is lack
of boolean chaining. If you have `config_setting`s `:config1`, `:config2`, and
`:config3` and want a `select` branch to trigger if any of them are true, you can
write:

```python
select({
    ":config1": [":desired_branch"],
    ":config2": [":desired_branch"],
    ":config3": [":desired_branch"],
    "//conditions:default": [":other_branch"],
})
```

This is obviously redundant, scales poorly, and risks bugs if you, for example,
change the value of `":desired_branch"` but accidentally forget to update a
line.

If you want the `select` to trigger when *all* conditions are true, the story is
even worse. There are basically two practical options. One is to define a new
`config_setting` `:config1_and_2_and_3` and manually copy all `values` entries
from the original ones there. This has the same scalability and bug issues as
above. The other is to "chain" `select`s by having an initial `select` trigger
on `:config1`, have that branch resolve to a dependency with its own `select`
triggering on `:config2`, and so on. This is extremely verbose, hard to read,
and doesn't work for non-label-based attributes.

## `selects.with_or`

The [Skylib](https://github.com/bazelbuild/bazel-skylib) module
[`selects`](https://github.com/bazelbuild/bazel-skylib/blob/master/lib/selects.bzl)
provides a partial solution to this problem. This defines a Starlark macro that
replaces `select` with an ehanced version that supports `OR` chaining:

```python
sh_binary(
    name = "my_target",
    srcs = ["always_include.sh"],
    deps = selects.with_or({
        (":config1", ":config2", ":config3"): [":standard_lib"],
        ":config4": [":special_lib"],
    }),
)
```

This works well for embedding `OR` chains directly into `select` statements. But
it's not reusable (since it binds the conditions and values they select together,
so other `select`s can't re-use them) and doesn't offer a solution for `AND`.

## Proposal

This document proposes new Starlark macros that leverage the
[`alias`](https://docs.bazel.build/versions/master/be/general.html#alias) rule
to create new  `config_setting`s that `OR` or `AND`-wrap other
`config_setting`s. Because these are real `config_setting`s, they can be used
anywhere `config_setting`s are accepted. This makes them protable and easy to
use.

This proposal is inspired by ideas expressed by [@jfancher](https://github.com/jfancher) in
[https://github.com/bazelbuild/bazel/issues/6449](https://github.com/bazelbuild/bazel/issues/6449#issuecomment-431471309).

### API

Given `config_setting`s `:config1`, `:config2`, and `:config3`, a new macro in
the [Skylib](https://github.com/bazelbuild/bazel-skylib)
[`selects`](https://github.com/bazelbuild/bazel-skylib/blob/master/lib/selects.bzl)
module called `config_setting_group` provides `OR` or `AND` chaining as follows:

**OR:**
```python
load("@bazel_skylib//:lib.bzl", "selects")

selects.config_setting_group(
    name = "any_config",
    match_any = [":config1", ":config2", ":config3"]
)
```

**AND:**
```python
load("@bazel_skylib//:lib.bzl", "selects")

selects.config_setting_group(
    name = "all_configs",
    match_all = [":config1", ":config2", ":config3"]
)
```

This reduces the original example above to the following:

**OR:**
```python
select({
    ":any_config": [":desired_branch"],
    "//conditions:default": [":other_branch"],
})
```

**AND:**
```python
select({
    ":all_configs": [":desired_branch"],
    "//conditions:default": [":other_branch"],
})
```

Setting both `match_any` and `match_all` in the same `config_setting_group`
triggers an error. There's no technical obstacle to doing this, but this
proposal aims to add as little extra complexity as necessary to serve
known needs. If the need for this extra logic is subsequently demonstrated, this
can be added as a followup.

### Implementation

The following is added to
[`selects.bzl`](https://github.com/bazelbuild/bazel-skylib/blob/master/lib/selects.bzl). It
exploits the fact that
[`alias`](https://docs.bazel.build/versions/master/be/general.html#alias) rules
can use a `select` to determine which rules they resolve to.

```python
def config_setting_group(name, match_any = [], match_all = []):
    """ TODO: document.
    """
    empty1 = not bool(len(match_any))
    empty2 = not bool(len(match_all))
    if (empty1 and empty2) or (not empty1 and not empty2):
        fail('Either "match_any" or "match_all" must be set, but not both.')
    _check_duplicates(match_any)
    _check_duplicates(match_all)

    if ((len(match_any) == 1 and match_any[0] == "//conditions:default") or
        (len(match_all) == 1 and match_all[0] == "//conditions:default")):
        # If the only entry is "//conditions:default", the condition is
        # automatically true.
        _config_setting_always_true(name)
    elif not empty1:
        _config_setting_or_group(name, match_any)
    else:
        _config_setting_and_group(name, match_all)

def _check_duplicates(settings):
    """ Fails if any entry in settings appears more than once.
    """
    seen = {}
    for setting in settings:
        if setting in seen:
            fail(setting + " appears more than once. Duplicates not allowed.")
        seen[setting] = True

def _remove_default_condition(settings):
    """ Returns settings with "//conditions:default" entries filtered out.
    """
    new_settings = []
    for setting in settings:
        if settings != "//conditions:default":
            new_settings.append(setting)
    return new_settings

def _config_setting_or_group(name, settings):
    """ ORs multiple config_settings together (inclusively).

    The core idea is to create a sequential chain of alias targets where each is
    select-resolved as follows: If alias n matches config_setting n, the chain
    is true so it resolves to config_setting n. Else  it resolves to alias n+1
    (which checks config_setting n+1, and so on). If none of the config_settings
    match, the final alias resolves to one of them arbitrarily, which by
    definition doesn't match.
    """

    # "//conditions:default" is present, the whole chain is automatically true.
    if len(_remove_default_condition(settings)) < len(settings):
        _config_setting_always_true(name)
        return

    # One entry? Just alias directly to it.
    elif len(settings) == 1:
        native.alias(
            name = name,
            actual = settings[0],
        )
        return

    # First alias adopts the core name so user references start here.
    native.alias(
        name = name,
        actual = select({
            settings[0]: settings[0],
            "//conditions:default": name + "_2",
        }),
    )

    # Second through (n-2)nd aliases:
    for i in range(2, len(settings) - 1):
        cur_setting = settings[i - 1]
        native.alias(
            name = name + "_" + str(i),
            actual = select({
                cur_setting: cur_setting,
                "//conditions:default": name + "_" + str(i + 1),
            }),
        )

    # (n-1)st alias: if true it can resolve directly to the final config_setting
    # (which doesn't need an equivalent alias).
    native.alias(
        name = name + "_" + str(len(settings) - 1),
        actual = select({
            settings[-2]: settings[-2],
            "//conditions:default": settings[-1],
        }),
    )

def _config_setting_and_group(name, settings):
    """ ANDs multiple config_settings together.

    The core idea is to create a sequential chain of alias targets where each is
    select-resolved as follows: If alias n matches config_setting n, it resolves to
    alias n+1 (which evaluates config_setting n+1, and so on). Else it resolves to
    config_setting n, which doesn't match by definition. The only way to get a
    matching final result is if all config_settings match.
    """

    # "//conditions:default" is automatically true so doesn't need checking.
    settings = _remove_default_condition(settings)

    # One config_setting input? Just alias directly to it.
    if len(settings) == 1:
        native.alias(
            name = name,
            actual = settings[0],
        )
        return

    # First alias adopts the core name so user references start here.
    native.alias(
        name = name,
        actual = select({
            settings[0]: name + "_2",
            "//conditions:default": settings[0],
        }),
    )

    # Second through (n-2)nd aliases:
    for i in range(2, len(settings) - 1):
        cur_setting = settings[i - 1]
        native.alias(
            name = name + "_" + str(i),
            actual = select({
                cur_setting: name + "_" + str(i + 1),
                "//conditions:default": cur_setting,
            }),
        )

    # (n-1)st alias: if true it can resolve directly to the final config_setting
    # (which doesn't need an equivalent alias).
    native.alias(
        name = name + "_" + str(len(settings) - 1),
        actual = select({
            settings[-2]: settings[-1],
            "//conditions:default": settings[-2],
        }),
    )

def _config_setting_always_true(name):
    """ Returns a config_setting with the given name that's always true.

    This is achieved by constructing a two-entry OR chain where each
    config_setting takes opposite values of a boolean flag.
    """
    name_on = name + "_stamp_binary_on_check"
    name_off = name + "_stamp_binary_off_check"
    native.config_setting(
        name = name_on,
        values = {"stamp": True},
    )
    native.config_setting(
        name = name_off,
        values = {"stamp": False},
    )
    return _config_setting_or_group(name, [":" + name_on, ":" + name_off])
```

### Error Messages

This approach produces good, but not perfect, errors on `select` fails.

A failed `AND` evaluation of

```python
selects.config_setting_group(
    name = "all_configs",
    match_all = [":config1", ":config2", ":config3"]
)

select({
    ":all_configs": [":desired_branch"],
})
```

produces the error

```shell
ERROR: /home/me/workspace/BUILD:10:10:
Configurable attribute "cmd" doesn't match this configuration (would a default
condition help?).
Conditions checked:
 //:config2
```

This is pretty *good*: it reports the first condition in the `AND` chain that
doesn't match. This is precise, easily traceable, and actionable.

`OR` chaining isn't quite as clear. When the equivalent `select` on `:any_config`
fails, this produces

```shell
ERROR: /home/me/workspace/BUILD:10:10:
Configurable attribute "cmd" doesn't match this configuration (would a default
condition help?).
Conditions checked:
 //:config3
```

This only mentions the final condition. It makes no reference to any of the
other conditions that were checked. There's no obvious way to improve this. One
option is to create a new `config_setting` for the final chain link called
`:config1_or_2_or_3`. But this also seems confusing and not worth the mild
benefit.

## Alternatives

The main alternative approach would be to ad chaining support directly to
`config_setting`. This proposal rejects that approach for the following reasons:

1. We value keeping `config_setting` as simple as possible. This keeps the core
logic simple, clean, efficient, and easy to maintain. It also keeps it
accessible to new users who are learning the feature. Experience shows that more
complicated models invite more complicated uses. For example, `select` replaced
an older Google-only attribute on C++ rules that was modeled on regexes. When
`select` was introduced, it replaced every instance of that attribute with
simpler logic and no loss in functionality - a net-win for everybody. It turned
out that no practical uses of the feature required the full might of regexes.

1. Bazel in general is moving toward a simple, focused core with most rule
logic writtern in Starlark. This proposal continues that momentum.

1. Putting the logic in Starlark makes it easy for users to understand it, fork
it for project-specific needs, and contribute patches. Logic in Bazel core is a
huge barrier to wide community participation.

1. This proposal doesn't complicate `select` *at all* for users who don't need
these features.

1. Keeping the core logic simple helps maintain efficiency guarantees. Even the
current logic has contributed to Bazel freezes and OOMs with rogue
`select`s (which thankfully have since been bug-fixed). More powerful models
make it harder to reign these corner cases in.

## Followups

The following extensions can be added to this proposal on request:

* `config_setting_group` can be expanded to support both `match_any`
and `match_all` in the same instance. This would be interpreted as "*this group
matches if any setting in `match_any` matches and all settings in  `match_all`
match*".

* We can add a peer macro to
[`selects.with_or`](https://github.com/bazelbuild/bazel-skylib/blob/8cecf885c8bf4c51e82fd6b50b9dd68d2c98f757/lib/selects.bzl#L17)
called `selects.with_and`. This would provide `AND`-style embedded chaining
directly in `select`s without having to define any new rules. This could work by
implicitly calling `config_setting_group` with `match_all` and passing a
reference to that alias to the `select`.

## Testing

Since `select` users heavily depend on the correctness of `config_setting` to
control their build flow, thorough unit tests will be added to the `selects`
module's [test
script](https://github.com/bazelbuild/bazel-skylib/blob/master/tests/selects_tests.bzl).

## References

* [Configurable Build
Attributes](https://docs.bazel.build/versions/master/configurable-attributes.html)
* Original idea: ["Documentation/behavior mismatch with alias +
config_setting"](https://github.com/bazelbuild/bazel/issues/6449)
