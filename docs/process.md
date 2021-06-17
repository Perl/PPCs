# Bootstrapping an RFC process

80% of our feature requests are for changes to the language.

In the past 5 years most work on the parser and tokeniser came from just 5 people. All of them are busy, and none of them are employed by "Perl 5 Porters".

We can't change this mismatch quickly - the reality is that even good ideas might stall for lack of anyone to implement them, and even where folks are prepared to help implement their ideas, we will find it hard to mentor many simultaneously.

Hence we need a process that

* acknowledges this
* emphasises scaling what we have
* enables the originator of an idea to help us as much as possible
* summarises and records discussion and decisions, to iteratively improve
* prioritises proposals, to optimise the value we get from contributors


We'd like to record proposals to improve the language and their status as "Request For Comment" documents in their own repository under the Perl organisation on GitHub.


We have a [template](template.md) for what an completed implemented RFC should end up as, but if all you have is an idea - don't worry, we'll help you get there.  We're still figuring this process out, so for now we're doing it as mail messages sent to p5p, not as "pull requests" to the RFC repository (or "issues" on the source repository). This way we can see if the process works as hoped, and fix the parts that don't.


## What makes a good idea?

Strictly speaking, as Perl is a Turing complete language, no changes to Perl are **necessary** because any task that can be implemented at all can be implemented with or without a proposed change. Arguing against a new feature because it is already possible is tautological. New features are suggested because they might make the language *better*. Better is subjective - we can't avoid different people weighing trade offs differently. Perl tries to make *easy things easy and hard things possible*, and is fine with *There Is More Than One Way To Do It*.

All changes have costs and benefits. Benefits of a new approach could be that it

* is less verbose than existing syntax
* has fewer ways to make mistakes
* is more efficient internally
* opens up new ways to express problems

Costs are

* yet another way to do it - Perl becomes incrementally harder to learn, remember and read
* existing tooling has to adapt to cope
* linearly more implementation to maintain
* exponentially more combinations of features to define and debug

Not every good idea belongs in the Perl core. Some are better implemented on CPAN. For some ideas, the RFC process is overkill. And the other way - for some issues or PRs, the reviewer is going to realise that it's more complex than it seemed, and needs to become an RFC.

## The straight through process is

                                 idea
                                   |
                                   v
                                mail p5p
                                   |
                  better           |         rejected
                    on     <-------?------->   with
                   CPAN            |         reasoning
                                   v
                               Draft RFC
                "we think this idea is worth exploring"
                (Help us figure out how this will work)
                                   |
                                   v
                             Provisional RFC
              "we think this idea is worth implementing"
              (We have a firm idea of what we want to do)
                                   |
                                   v
                               Accepted RFC
                  "we think this plan looks viable"
               (There is a sane plan for how to do it)
                                   |
                                   v
                             Implemented RFC
                   "docs, tests and implementation"
                (And we can support it in the future)
                                   |
                                   v
                                Shipped
    In a stable release, subject to "experimental features" process
                                   |
                                   v
                                 Stable
    In a stable release, now subject to normal "deprecation" rules



If there are better names, we should change them. The intent is the important part. Also RFCs might still fail at any point (before "Stable") and hence become "Rejected". RFCs might also be "Withdrawn" by their Author(s), or "Superseded" by a newer RFC, so these states and transitions exist.

Part of this discussion to get from "Draft" to "Accepted" should include whether a feature guard is needed, concerns on CPAN breakage, security etc, helping to fill out the "Backwards Compatibility" and "Security Implications" sections. This is the point where a subject-matter expert may raise concerns about the proposal, and may effectively veto it. For example, if you propose a change related to Unicode, and Karl says "it's a really bad idea for the following reasons", then it's not likely to progress.  Similarly, as the discussion progresses, it may become clear to everyone that the idea should be rejected. We might figure out that the idea is better implemented on CPAN, that something we thought was better on CPAN should return as an RFC. (eg try/catch and Moose/Moo leading to Cor).

Any RFC (before merging) can be marked "Deferred" if work has paused, or if they have no-one implementing them. RFCs have at least on *Author*, who acts as champion for the idea, and ideally writes documentation and tests. "Accepted" RFCs should have a core team member as a *Sponsor*, who acts as mentor and point of contact. If the *Author* can't implement their idea alone, and no-one else volunteers, then the PSC will try to find someone to implement an "Accepted" RFC, but this may not be possible, and the RFC will stall.

The PSC approves the transitions Draft => Provisional => Accepted
but will actively seek opinion from people familiar with the subject.

The transition from Accepted => Implemented is made by merging the PR that implements the RFC. At least one reviewer should be neither the *Author* nor the *Sponsor*. (Even if all that they can say is that the other two are **the** subject experts and that it all looks good.)

There is an ongoing discussion how how we decide whether to merge a PR, with the latest proposal being:

* After an appropriate period, if there are not strong disagreements, and the PSC haven't rejected it, a committer will merge the PR.
* Trivial or obviously-correct changes may be committed directly. I.e., the appropriate length is sometimes zero.
* If objections are raised, they need to be addressed (meaning "clearly replied to", and for an RFC the reasoning recorded in "Rejected Ideas")
* If the objection is from a subject matter expert, you need to come to an agreement.
* If the PR stalls, ping the PSC for adjudication. We expect this will be quite rare either you should have gone through the proposal process, or it's straightforward.
* PRs from non-committers are expected to have more scrutiny.

What's an appropriate period?

* For "I think this is right but want to give a chance for comments", a couple of days is probably fine.
* For "this is a significant change that I could really use feedback on," a week or more is probably best, and the PR should probably be flagged to the list as wanting attention.
* The later in the release cycle, the stricter we should be with ourselves.

Don't panic: changes can be reverted.


## What needs an RFC? What can just be a PR?

There's no obvious answer, because there's no clear cut off, and there never will be, even when the process is "out of beta". For now we think we should use RFCs for

1. Language changes (feature changes to the parser, tokeniser)
2. Command line options
3. Adding/removing warnings (entries in `perldiag.pod`)
4. Significant changes to when an existing warning triggers

A case that came up recently was moving the reporting of an error from runtime to compile time (GH #18785). We think that this wouldn't warrant an RFC (just regular code review) because no correct code should be relying on when an error is reported. However, there is still a judgement call here, as it would **not** be correct for constant folding to report errors (such as divide by zero) as this might only happen on some platforms as a side effect of the values of constants, and those expressions were unreachable on those platforms.
