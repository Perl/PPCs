# Bootstrapping a PPC process

PPC is the acronym for "Perl Proposed Change".

80% of our feature requests are for changes to the language.

In the past 5 years most work on the parser and tokeniser came from just 5 people. All of them are busy, and none of them are employed by "Perl 5 Porters".

We can't change this mismatch quickly - the reality is that even good ideas might stall for lack of anyone to implement them, and even where folks are prepared to help implement their ideas, we will find it hard to mentor many simultaneously.

Hence we need a process that

* acknowledges this
* emphasises scaling what we have
* enables the originator of an idea to help us as much as possible
* summarises and records discussion and decisions, to iteratively improve
* prioritises proposals, to optimise the value we get from contributors
* clarifies who is meant to be pushing work forward at any given time


We'd like to record proposals to improve the language and their status as "Request For Comment" documents in their own repository under the Perl organisation on GitHub.


We have a [template](./template.md) for what an completed implemented PPC should end up as, but if all you have is an idea - don't worry, we'll help you get there.  We're still figuring this process out, so for now we're doing it as mail messages sent to p5p, not as "pull requests" to the PPC repository (or "issues" on the source repository). This way we can see if the process works as hoped, and fix the parts that don't.


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

Not every good idea belongs in the Perl core. Some are better implemented on CPAN. For some ideas, the PPC process is overkill. And the other way - for some issues or PRs, the reviewer is going to realise that it's more complex than it seemed, and needs to become a PPC.

## The Process

![a flowchart of the process described below](../images/flowchart.png)


### Pre-PPC

The PPC process starts with a formal proposal to add or change a language feature in Perl.  But you, the prospective author of a PPC, shouldn't start by writing that formal proposal.  Start by posting to p5p that you have an idea.  Explain what problem you're solving, how you think you can solve it, and what prior art you looked at.  Be clear and concise.  Make it easy for the rest of the list to see what you're suggesting without reading an enormous wall of text, but don't lose so much detail as to be meaningless.

You are taking the temperature of the list.  If there is a great outcry that this is a bad idea, or has been tried before, or was explicitly rejected before, you should probably stop.  No hard feelings!

Otherwise, you're ready to move on to the next step. You should post a follow-up, requesting that the PSC approve producing a draft.  If they do, move on to "Draft Proposal" below.  If not, they'll provide more feedback like "this is definitely impossible" or "you need to provide more information" or so on.

During this "Pre-PPC" phase, your proposal isn't in the PPC tracker.  It's not a PPC yet!

During this phase, you (the proposer) are responsible for moving things forward.  If you contact the PSC for approval to file a draft, and the PSC does not respond, it's you who should be keeping track of that.

### Draft Proposal

The PSC has agreed that you should write a formal draft proposal.  You get the [template document](./template.md) and fill it out.  You take its advice, thinking hard about what goes in each section.  Then you post it to p5p as an email with the subject "PROPOSAL:  my great idea".  Members of the list will reply with more questions and suggested amendments.  You should read them and amend the proposal to clarify your ideas or react to valid criticism.

During this phase, you (the proposer) are responsible for moving things
forward.

When you think your idea has been sufficiently scrutinized, and you have gotten all the feedback you're going to benefit from, post to perl5-porters, requesting the PSC accept the draft.  In reply, they will either request more discussion, reject the proposal, or enter it into the tracker with the status **Exploratory**.

### Exploratory Status

When a formal draft has been discussed sufficiently, submitted to the PSC, and is under consideration, it is entered into the proposal tracker, with the status **Exploratory**.  The PSC (or other deputized folk) will begin the process of vetting the idea.  They will review discussion about the idea, they will produce a list of questions not yet answered, and they will press existing core team members (or other experts) for input.

The PSC (or deputies) will eventually either:
 * move the document to **Implementing** status (see below) because they believe it is ready for implementation
 * move the document to **Rejected** status because they believe it should not be implemented.  This may come with advice on how to formulate an alternate proposal that has more chance of being accepted.  This isn't used for "it needs some edits", but for "it's fundamentally deeply flawed."
 * move the document to the **Expired** status because the original proposer (or other vital personnel) are not responsive

During this phase, the PSC is responsible for moving the proposal forward.

### Implementing

When a proposal is accepted, the PSC is stating that they'd like to see the idea implemented, and that they are likely to merge an implementation if it doesn't bring with it any unwelcome surprises or complications.

If no implementation has made progress for three months, the document moves to **Expired**.

If an implementation exposes serious problems that mean the PSC no longer believes the proposal can work, the document moves to **Rejected**.

If an implementation is delivered and it isn't clear that it's broken as designed, the document moves to **Testing** status.

During this phase, the proposer or other named implementor is responsible for moving the proposal forward.  The PSC will only review proposals in this status when work is delivered or when status updates cease.  The PSC will make a regular note of proposals in this status to perl5-porters.  The implementors of proposals in this status will post regular updates -- regular enough, at any rate, to avoid becoming Expired.

### Testing

Now there's an accepted proposal and an implementation.  At this point, it needs to have enough tests, showing how it will work in real world conditions.  The PSC will consult with the author, each other, perl5-porters, and other experts to produce a list of needed test cases.  The proposer, implementer, or other volunteers will provide the testing.  Once enough testing is done, then the document will be either…

 * marked **Accepted**, with the code merged as a new (probably experimental!) feature of perl
 * marked **Rejected**, because the quality or behavior of the feature or its implementation are not acceptable

If no progress is reported for three months, the document moves to **Expired**.

During the Testing phase, the PSC and the proposer (or implementor) will be working together and communicating regularly to keep track of what work remains to complete the testing phase.

## What needs a PPC? What can just be a PR?

There's no obvious answer, because there's no clear cut off, and there never will be, even when the process is "out of beta". For now we think we should use PPCs for

1. Language changes (feature changes to the parser, tokeniser)
2. Command line options
3. Adding/removing warnings (entries in `perldiag.pod`)
4. Significant changes to when an existing warning triggers

A case that came up recently was moving the reporting of an error from runtime to compile time (GH #18785). We think that this wouldn't warrant a PPC (just regular code review) because no correct code should be relying on when an error is reported. However, there is still a judgement call here, as it would **not** be correct for constant folding to report errors (such as divide by zero) as this might only happen on some platforms as a side effect of the values of constants, and those expressions were unreachable on those platforms.
