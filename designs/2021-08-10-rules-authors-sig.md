---
created: 2021-08-10
last updated: 2021-08-19
status: To be reviewed
reviewers:
  - philwo [?]
  - sventiffe
  - hicksjoseph
title: Rules Authors SIG
authors:
  - [Alex Eagle](http://github.com/alexeagle), Aspect
  - [Daniel Wagner-Hall](http://github.com/illicitonion), Apple
  - [Helen Altshuler](https://github.com/helenalt), EngFlow
  - [Keith Smiley](http://github.com/keith), Lyft
---


# Abstract

We propose to form a Special Interest Group (SIG) among authors of commonly used Bazel rulesets.

> The term "rulesets" is used in this document to refer to all
> Bazel extensions, including Editor Plugins and Starlark shared libraries.

The intent is for these engineers to share technical approaches for solving common problems,
to have a single coherent voice for interacting with the core Bazel team,
and to provide a more consistent experience for Bazel end-users.

Note that Bazel does not have a general, repeatable process for forming SIGs like Tensorflow and Kubernetes have.
Our intent is not necessarily to institute such a process, though it is possible that other groups such as Remote Execution might want to follow our lead to form additional SIGs.

# Background

Currently, Bazel core, which is the bazelbuild/bazel repository, is governed by Google.

Bazel rulesets, on the other hand, have a variety of governance.
Many of them have been handed off to community maintainers,
who are mostly working in isolation from each other.

There are many common problems for rulesets however. Some examples:

- providing a consistent experience for users
- versioning, semver, and changelogs
- support matrix (e.g. supporting Bazel LTS vs rolling releases)
- release engineering, e.g. artifacts that don't expose design-time deps
- dealing with Workspace dependencies, often on other rulesets
- high-level documentation linked from bazel.build
- API documentation generation and hosting
- getting fixes into dependent layers such as bazel-skylib or bazel itself
- triaging the flood of user issues and pull requests
- funding work done by contributors

The Rules Authors SIG provides a forum for discussion of cross-cutting concerns such as these.

It also streamlines communication between rulesets authors and the Bazel core team, allowing the latter to prioritize work based on the consensus among authors.
For example, the SIG might help to curate a list of upstream PRs and issues that are of high value across rulesets.

Ultimately, the SIG may determine its own scope beyond its charter.

# Proposal

## Charter

The SIG facilitates the work of maintaining Bazel's rulesets by bringing community authors together to solve shared problems.

Participants must abide by a Code of Conduct.
> Note that the Bazel repo itself does not have one, so we'll have to search for something appropriate.
> Possibly https://conf.bazel.build/2018/coc can be a starting point.

## Communication

The SIG should communicate in the open, ensure other SIGs and community members can find notes of meetings, discussions, designs, and decisions, and periodically communicate a high-level summary of the SIG's work to the community.

The [bazel-dev](https://groups.google.com/g/bazel-dev) mailing list seems like the
best place to post this communication.

## Membership

A trade-off must be made between excluding valuable input, and having too many people involved for meetings to be productive.
We don't want SIG meetings to become a forum for users to ask for support or escalate issues or Pull Requests.

Any member joining the SIG should commit to active participation to
support the goals of the group.

We propose that the group be initially formed with a representative from each of the [recommended] rulesets.

> We could consider some metric of popularity as well, as some
> recommended rulesets don't seem actively maintained.
> However this ought to be based on adoption numbers, and we don't have these metrics.
> We could use GitHub stars/forks count if we want to limit the initial
> participants, however in the spirit of inclusivity this proposal
> doesn't make any such limitation.

Clearly Google teams have a stake in many rulesets, so the Tech Lead/Manager of
the core Bazel team may propose liasons from those teams to 
observe or participate as much as the policy determined by their management allows.

In addition to the OSS maintainers, the SIG needs curated input from the user community to inform prioritization.
We also want to remain close to resources at companies who can afford their engineers time or financial support for OSS.
For this reason the initial group may include a representative from each of the Bazel [experts].

[recommended]: https://docs.bazel.build/versions/4.1.0/rules.html#recommended-rules
[experts]: https://bazel.build/experts.html

As soon as this initial group meets, they can decide what criteria should
exist for determining additional membership.

### Leadership

We will need someone to do the grunt work of organizing and leading meetings, ensuring we take notes, etc.

Proposal: at the first meeting the group nominates a primary and a secondary Chair, with a six-month rotation before handing off to other members.

## Initial meeting

Given the group of formative members described above, they will first have some
organizational meetings to bootstrap the group, mostly to determine the form and
agenda of the first SIG meeting.

We propose that the first SIG meeting take place the day after BazelCon 2021
(Friday, November 19), at a time permitting most members to attend in their
timezone (likely evening CET/morning US Pacific).
Many participants will be in "conference mode" already that week
so this minimizes churn, and waiting until after the conference means our brains
are primed from the talks and customer interactions.

## Possible work products

This section is speculative, but should help to illustrate what kinds of specific activities we think the SIG might do.

1. Synthesize customer input to cross-prioritize work across rulesets, so that authors can shift their effort where it is most needed
1. Tweak and share settings for GitHub actions/bots like [Stale](https://github.com/marketplace/actions/close-stale-issues)
1. Solve a dev-infra shaped problem across rulesets, for example we introduced a pre-commit hook to most rulesets to trigger buildifier formatting before sending a PR, using https://github.com/keith/pre-commit-buildifier
1. Maintain a prioritized list of issues and PRs on upstream repos such as bazelbuild/bazel that the rules authors would most like to see fixed/merged
1. Make a decision about rules APIs so that there is more consistency for users, such as naming attributes similiarly
1. Create a rules authoring style guide
1. Allocate funding from a source like an OpenCollective to contributors (like Chrome does: https://opencollective.com/chrome)

## Decision-making

We propose to use the [Rough Consensus] model from IETF

> Working groups make decisions through a "rough consensus" process. IETF consensus does not require that all participants agree although this is, of course, preferred. In general, the dominant view of the working group shall prevail. (However, "dominance" is not to be determined on the basis of volume or persistence, but rather a more general sense of agreement). Consensus can be determined by a show of hands, humming, or any other means on which the WG agrees (by rough consensus, of course). Note that 51% of the working group does not qualify as "rough consensus" and 99% is better than rough. It is up to the Chair to determine if rough consensus has been reached (IETF Working Group Guidelines and Procedures).

[Rough Consensus]: https://en.wikipedia.org/wiki/Rough_consensus

# Models

Here are things we looked at as inspiration for this proposal.

## Kubernetes SIGs

https://github.com/kubernetes/community/blob/master/governance.md#sigs

## Tensorflow SIGs

https://www.tensorflow.org/community/sig_playbook
