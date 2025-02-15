(June 2021)

# Top line

80% of our feature requests are for changes to the language, and most seem to come from folks with no experience of core development. Whereas over the past 5 years most work on `toke.c` and `perly.y` is from just 5 people

    Karl, LeoNerd, Tony, Zefram, Dave

(with the first 3 far more prominent)

There's a massive disparity between how many ideas we have and how many people we have to even **mentor** others, let alone implement things.

To get out of this hole, we need an approach that acknowledges this and emphasises scaling up and out with what we have, instead of pretending that we're short on ideas and long on under-used talent.

# Meaning

Sure, we do not want to rule out "unsolicited" ideas, but we **have** to assume that these were unlikely to go far unless someone else with the right skills buys into them.

I really don't want to just create a "roadmap" of "aspirations" that blocks for years because no-one implements it. "Yes, we're going to add desugaring assignment..."

# So I want it to be clear

* How the person with the idea can contribute and help get a lot done
* How much work there really is
* That **your** idea needs to be applicable to other people ...
* ... and it needs at least 1 more person prepared to help implement it

# I want to avoid

* [The Perl 6 RFC failure](https://www.perl.com/pub/2000/11/perl6rfc.html/) of "no-one knew how to implement it"
* Throw an idea over the fence and assume that it will be picked up
* "Quantity over quality" of ideas
* A process that is too verbose to read
* Different people (repeating) the same feedback/what-if questions, because the answers to these are not easy to locate, resulting in noise and makework.


# Bugs, optimisations, build improvements etc

* can objectively and uncontroversially be triaged as accepted or rejected
* rarely conflict with each other
* once completed and tested can generally be closed and forgotten

# Feature requests are

* subjective
* involve trade offs that different people value differently
* have more states that "open" and "closed"
* and even rejected ideas should not be forgotten, because the reasoning for rejecting them is often interesting.

It's not helpful to place both in the same "queue" and process them as variants of the same thing, because they are not. They don't even need to be in the same repository - you don't need the possible future history of Perl in order to build it from source, or use it to write working programs.

We can strive to reach 0 bugs.

Unlike bugs, the functionality of released Perl doesn't suffer if we have infinite open features. We gain nothing from striving to reach 0 open feature requests.

# I'm really keen to

1. Split the bug queue from the **idea** queue
2. Distinguish between "someone's idea" and "idea we like and think is viable"
3. Be clear that there are no magic coding fairies (it should be the exception rather than the rule that we accept ideas without an idea of who is going to do it)
4. Be clear that there is a bunch of project management stuff that folks **can** help with to champion their feature - that's how they help us scale.  "oh, but I can't hack C so clearly I can't help here" is a misconception
5. Give equal prominence to ideas that didn't make it
6. And of assessments of why


# What are we trying to achieve

* A structured way for anyone to contribute workable ideas to improve Perl
* Permit folks without deep technical skills to be maximally helpful
* Spread workload to offload as much as possible from the few people who can actually wrangle the internals
* Distribute and decentralise where possible
* Record decisions taken along the way
* Create a record of what works/what doesn't to improve future contribution

Basically the *MVP* - Minimum Viable Process.
