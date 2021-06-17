# How do we scale this up so that it doesn't need so much central attention?

The initial process assumes that

* many people want to submit ideas
* most of them don't understand what it takes to get an idea implemented
* there are a lot of duplicate ideas
* we don't have a clear way for them to know "what ideas are viable"
* we don't know if the process will work

so there's a deliberately a lot of hand holding and centralisation, so that we can steer folks in the right direction, or say no early, before they do too much work.

## If the process works, I'd like to progress to

* people submit draft RFCs, not just ideas
* they've actually consulted other people *before* submitting
* the person submitting the RFC knows that it they don't help, it dies
* there's a clearer set of requirements needed to progress

## What does the Author do/Where do RFCs live?

The Author is the champion for the RFC. Their motivation and enthusiasm to have the feature successfully implemented and shipped drives the process.  Their task is to eliminate each "lowest hanging excuse" in turn to get to shipping code. They are responsible for

* seeking input
* shepherding discussion to some sort of consensus (or appealing to the PSC to resolve an impasse)
* ensuring all sections in the RFC are complete (working with implementers and others as necessary)

RFCs live in version control. Anyone can create an "Exploratory" RFC, self-assign it an ID, and start to flesh it out. "Draft" status gets you an official RFC ID, and from that point onwards the RFC and history is mirrored/merged into the official repository.

To make this workable, the RFC should be in a source code repository that the *Author* can edit directly (GitHub, GitLab, Bitbucket, self-hosted git, etc). The minimal workable requirements are that

1. it gives a well-known stable URL for the rendered current version
2. it can tag (or identify) specific previous revisions
3. history can be mirrored into the official repo, and merged on status change
4. discussion can be archived once RFC is "Accepted" and "Implemented"

Hence GitHub PRs might not be the best forum for discussion, unless they can be archived. Basically we want to avoid the situation were we have a feature live, and some third party can delete the historical discussion related to it.

As we get better at this, I think that the status transitions should aim for these minimum requirements


## Exploratory

* *Author* self-assigns RFC ID
* *Author* seeks input/feedback

## Draft

* MUST have *Sponsor* (on the core team, or PSC can delegate externally)
* MUST have (at least minimal) Motivation, Rational and Examples
* SHOULD have (at least a minimal) Specification
* Gets an official RFC ID

Means "we think this idea is worth exploring"

## Provisional

* MUST have viably complete Motivation

Means "we think this idea is worth implementing"

## Accepted

* MUST have viably complete Motivation, Rational, Specification, Examples
* MUST have Prototype Implementation/Proof Of Concept/Specific Plan

Means "we think this plan looks viable"

## Implemented

* MUST have no Open Issues
* MUST actually have an implementation!

Means "it's good to merge - we think we can support it in the future"

## Shipped

* In a stable release, subject to "experimental features" process

## Stable

* In a stable release, would need to follow "deprecation" process to remove



where (at least) "Provisional" and "Shipped" can be skipped.

## RFC IDs, and how to self-assign

* Official RFC IDs are 4 digits
* Self-assigned IDs should be `CPANID-#### or` `githubid-####`

These are the obvious likely popular two, and done case sensitively will not clash. These two should be sufficient for a workable globally unique system.
