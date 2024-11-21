# Perl 5 is Perl

## Preamble

    Author:  Aristotle Pagaltzis <pagaltzis@gmx.de>
    ID:      ARISTOTLE
    Status:  Draft

## Abstract

There is no Perl&#160;6 looming any more, so the only thing “Perl” now refers to is Perl&#160;5. That has also forever been the language that people mean when they say just “Perl” rather than specifying “Perl&#160;5”. There were earlier versions of Perl, but they are ancient history. All the history that created a distinction between “Perl” and “Perl&#160;5” has fallen away while nobody was paying conscious attention.

The interpreter already identifies itself as “perl 5, version 40, subversion 0”. We should just drop the 5 and make it official.

## Motivation

Perl is in need of a future.

30 years ago on October 17, 1994, it turned 5.0.

In its early years Perl went through versions rapidly. 1.0 came out in December 1987, less than 7 years earlier. Up to Perl&#160;4 (itself just a release of Perl&#160;3 re-badged for the convenience of the first Camel book), Perl had been a different, distinct, earlier language, in the sense that code written in it would be structured differently. This early Perl was not well suited to programming in the large.

Perl&#160;5 changed that. It expanded the power of the language drastically and enabled the creation of CPAN. In so doing it also broke compatibility in some small ways, famously breaking some older Perl poetry.

In 2000, some 6 years later – almost as long as it had taken to reach 5.0 –, the need for rejuvenation led to [the inception](https://www.nntp.perl.org/group/perl.packrats/;msgid=20020729060339.GF11511%40chaos.wustl.edu) of Perl&#160;6. It is easy to see how it would have seemed obvious that Perl&#160;6 would repeat the precedent set by Perl&#160;5, just on a larger scale. Under that understanding, Perl&#160;5 was the maintenance track, the stable version, while Perl&#160;6 was the version under development. That was the future of Perl.

The intent was to take the version bump as an opportunity to break compatibility as widely as needed, just once, to fix deeper problems and regularize the language so as to allow it to be defined in terms of itself, laying a solid foundation that would be able to evolve without further fundamental breakage. This was to be the community’s rewrite of Perl.

It would not have been apparent at the time that this would iteratively lead to an entirely distinct language. But that’s what it did. Going back to Larry’s 2015 Tolkien talk, one might say that Perl had been Larry’s vision of Unix, and Perl&#160;6 was Larry’s vision of Perl. It is obviously a very closely related language, but in many ways also deeply different.

It is not tautological that this is equally true in the other direction. Perl&#160;5 suffered significantly from attempts in the 5.10 era to adopt early designs of parts of Perl&#160;6 verbatim. Even all the way back then, the languages were too far apart for this to work. As a result, many of those features had to be (and some are even today still being) laboriously excised again from the language.

In all this meantime it became clear that not everyone was going to adopt the new language, and that the ecosystem of the existing language was going to stick around. There would continue to be interest in Perl&#160;5 evolving for the foreseeable future, even as Perl&#160;6 came into its own. Thus Perl became a family. And that was now its future.

Both conceptions of Perl’s future posed problems for the outside perception of Perl&#160;5. Even in the language family framework, though Perl&#160;5 was no longer the eventually-obsolete version, it appeared as the older, wartier, outmoded flavor. (In fact, both conceptions of Perl’s future equally posed problems for the outside perception of Perl&#160;6.)

The community too, mirroring the versions-then-languages themselves, bifurcated, then separated, becoming two somewhat overlapping but distinct communities.

And then, in October 2019, [Perl&#160;6 changed names](https://github.com/Raku/problem-solving/pull/89#pullrequestreview-300789072). Suddenly, it was no longer the future of Perl, in any shape, not even as the more advanced flavor.

But Perl&#160;5 is bound to being Perl. It has a present, and a 30-year-long past as Perl&#160;5, but is now again in want of a future, and needs a large correction of its outside perception.

This void has made itself felt already. A first attempt was made to define and release a Perl&#160;7. It did not succeed – which only made Perl’s perception problem worse. And the matter has not gone away. Perl needs a future, and attempts to create one will be made until one of them succeeds.

## Rationale

Just as Raku has given up on the notion that it was going to be Perl&#160;6, the successor to Perl&#160;5, so Perl needs to give up on the notion of being Perl&#160;5, the predecessor to Perl&#160;6, and claim the name Perl back for itself.

We have an opportunity then to look back and recognize that Perl never did stand still. It has continued to change its present, one moment at a time, unwinding more of its story in the process. It has unassumingly been heading into its own future all along, merely obscured in this by the drawn-out separation from Raku and the necessity of clinging to being Perl&#160;5 in the meantime. From a pre-5.0 perspective, there was no reason not to keep increasing the major version number.

And so that is the proposal. The next release will not be Perl&#160;5.42 but simply Perl&#160;42 – which is none other than what it really was anyway. There is no difference between them except in our self-understanding. What this is is a reclaiming of Perl’s history.

The pace of changes did decrease after 5.0, relative to before, but the reason was the much greater potential of its new facilities, and the realization of that potential through CPAN, which allowed the language to fill new niches without necessitating changes to its core. The reason the transition from Perl&#160;5 to Perl&#160;6 failed to materialize when the one from Perl&#160;4 to Perl&#160;5 succeeded so thoroughly was that despite truly being a new and more powerful language, Perl&#160;5 was almost entirely a superset of Perl&#160;4. Subsequent attempts have borne this out: [Kurila](https://github.com/ggoossen/kurila), [Moe](https://github.com/MoeOrganization/moe), and despite its much smaller ambitions, Perl&#160;7 too, all ultimately failed to gain traction. But meanwhile, new internal APIs have come along and made it possible to extend the language in previously impossible ways – on CPAN. And eventually we even learned how to undo old mistakes: by naming them with feature flags to be turned off under future feature bundles.

That is the future. Perl will not have a successor. It will not break with its past. It will keep moving forward the same way it always has: sometimes with substantial changes; often with minor ones; occasionally with breaking changes, but always deliberately and circumspectly.

Relations between Perl and Raku may (and should, and hopefully will) adjust to each other’s new self-understanding. In time, the family should rebalance into a new equilibrium, as family systems do. Perl and Raku can and should stay in touch and take inspiration from each other, even if each needs its features designed for cohesion with its own overall design.

This type of re-versioning has some precedents:

* Java 1.4 was followed by Java 5. The technical details of that transition differ from what is proposed here, but even there we have a parallel between how Java 5 was mostly still 1.5 internally and how this proposal deals with XS code. Lessons can be had from the unlikeliest of sources.
* For some readers, the relation between Solaris 7 to SunOS 5.7 and onward may come to mind (although the relationship between Solaris and SunOS in earlier versions is messier).
* Having undergone this change, our versioning cadence will match the one followed by Node.js, with a new odd and even version every year, denoting a new development and stable release. Javascript was influenced more than a little by Perl, so this is not such unlikely company as may today seem.

Version 42 specifically is a particularly compelling point in time at which to do this, and not just for the Douglas Adams reference which will be evident even to Perl outsiders: there are further points of significance within Perl culture too, particularly when it comes to versions. 42 happens to be 6 &times; 7, and in ASCII, 42 (and thus the v-string `v42`) happens to be `*`. From a technical perspective, it is also opportune to undergo this transition well before version 48, at which point the fact that `v48 eq '0'` (and `v50 eq '2'` etc.) might cause unanticipated heartburn – so it is useful to have time to have any potential fallout sorted out well ahead of that.

## Specification

In the next release of Perl, `perl -v` will not say “Perl 5 version 42 subversion 0” but simply “Perl version 42.0”.

The value of `$]` will be simply 42 and `$^V` will be equal to `v42.0`. In future, the version of Perl will only have a major and minor component. 42.1 will have `$]` as 42.001 and `$^V` as `v42.1`.

For XS code, nothing changes at all. `PERL_REVISION` continues to exist and its value continues to be 5, forever. Only `PERL_VERSION` and `PERL_SUBVERSION` are relevant going forward.

A number of new environment variables are introduced which supersede existing environment variables if present:

* `PERL5LIB` → `PERL_LIB` (but cf. [Open Issues](#open-issues) regarding this one)
* `PERL5OPT` → `PERL_OPT`
* `PERL5DB` → `PERL_DB`
* `PERL5DB_THREADED` → `PERL_DB_THREADED`
* `PERL5SHELL` → `PERL_SHELL`

As a result, all Perl environment variables except `PERLIO` now follow the pattern `PERL_*`.

Finally, we edit the documentation to no longer advertise the use of a v-string on the `use VERSION` line. (Cf. [Examples](#examples))

PAUSE and MetaCPAN must be made aware of the new versioning cadence.

At some point, `INSTALL_BASE` should be discouraged (though remain supported indefinitely) and supplanted by a new option which does not contain the hard-coded `perl5` segment. It should then also be made to install to a path containing the full perl version, to solve problems with switching between versions. Ideally this will be in 42.0 but it can follow later.

## Backwards Compatibility

This proposal introduces no new syntax or semantics.

Adjustments will be required in code which parses the output of `perl -v`/`perl -V` and Perl code which expects `$]` and `$^V` to always be of a 5.x form, or always have three components.

XS code is almost entirely unaffected because the `PERL_REVISION` `PERL_VERSION` `PERL_SUBVERSION` symbols continue to exist and continue to change values exactly as they have been doing all along. Only XS code which tries to reconstruct the values of `$]` or `$^V` (or some facsimile of them) from those symbols will now be incorrect and will have to be adjusted.

Code that unsets `PERL5*` environment variables will technically become broken by this change and will have to be fixed to also unset `PERL_*`. But this breakage is not going to manifest widely in practice until sometime down the line; also, it is probably almost entirely down to `PERL5OPT` and `PERL5LIB`, as well as likely mainly being down to parts of the ecosystem which are visible to us on CPAN and can therefore be updated.

Code that cares about the paths to installed modules will have to be made aware of the new variant of `INSTALL_BASE` and its resulting directory structure, once that comes into play.

## Security Implications

There is a very marginal possibility that  code which does not expect a Perl revision other than 5 might present an opportunity for some kind of exploitation.

## Examples

`use 42;`

Note that a v-string is not used. A v-string is how you can write `use v5.40` rather than the less readable `use 5.040`, where the important part is noisier – namely the version, 40, rather than the more prominent revision, 5. Without the revision, the version becomes the integer part and thus no longer requires padding. The subversion becomes the fractional part behind the first dot, with no need for a second dot – however, specifying a subversion on the `use VERSION` line no longer comes up anyway, because feature bundles are not subversion-specific and Perl point releases have long since lost the longevity and prominence they had in the 5.6/5.8/5.10 era. (If you were to need to specify a subversion, then without a v-string you would still have to write e.g. `use 42.001`, which is not as nice as `use v42.1` – but this is now academic.) Thus the need for a v-string goes away in practice. This also means Perl novices are no longer immediately confronted with them.

## Prototype Implementation

None.

## Future Scope

Hopefully none. This proposal is a recognition that what we have long been doing has already been what Perl needed, and it was the language siblings relationship that needed to sort itself out and settle in. Perl will continue to be Perl even as the language grows and changes with the times.

## Rejected Ideas

There are some alternative approaches we could take:

1. Continue with Perl&#160;5.x indefinitely

   We might decide on this option because we aren't going to change the language substantially, and there are technical complications in changing the version. But this would not address the perception issue, and the non-change nature of this approach leaves the version number issue open to renewed future challenge.

2. Move to Perl&#160;7 immediately

   We might decide on this option because we are continuing to evolve the language anyway, so now that Perl&#160;6 is no longer in play, we are free to just bump the major version, and there are plenty of differences to Perl&#160;5.0 to justify a new major version. This is a viable option, but the technical costs (mainly due to XS version checking) are somewhat significant, and we take them on in exchange for nothing in particular. The outside-world optics of this would be odd, with the version finally moving beyond 5 after almost 30 years, but for no apparent reason. (Even when Perl&#160;3 was rebadged as Perl&#160;4, it was because of the Camel book.) And internally, this lack of apparent reason also leaves us with an entirely open question regarding how we should decide when to bump the version again, to 8 and beyond, putting us back in the same position regarding the version some time down the line as we are today.

3. Move to Perl&#160;7 based on some headline feature (such as a new object system)

   This has the same technical costs as moving to Perl&#160;7 generally, but the outside-world optics of the move are less strange. However, the feature then needs to be big enough to justify a bump in this way, lest it be perceived as a PR stunt for repackaging mostly-same-old as a major new version. And it does not set a precedent for when the version is to be bumped, opening up a future debate about whether changes are big enough to warrant a bump, and creating indirect pressure to introduce big headlining features every so often in order to justify a bump to avoid a renewed perception of stagnation. In the same vein, as already experienced before, with this type of version bump, there will be strong temptation to including breaking changes within its scope, every single time a new one happens – not just because of the seeming opportunity to clean house, but partly also because a perceived need to break certain things more easily makes the case for bumping the version.

Moving to version 42 combines (hopefully the best) aspects of all of these options. It retains a lot of the technical non-cost of option 1, by being essentially indistinguishable from option 1 at the XS level. Outwardly it is closest to option 2, in that the version changes without being motivated by a particular headlining feature, but unlike option 2, it does address the optics of changing the version. It just does so in a very different way from option 3: rather than justifying the bump by making big changes to the language, it highlights the fact that important changes have already been happening and the language has already been evolving, all along.

## Open Issues

1. The environment variable superseding leaves us with an awkward priority cascade for `PERL_LIB` > `PERL5LIB` > `PERLLIB`. However, the last one of these is already mainly historical.

   Maybe we want to reuse `PERLLIB` by flipping the priorities instead of introducing `PERL_LIB`. In that case maybe the counterpart to `PERL5OPT` should be named `PERLOPT`, not `PERL_OPT`, given that these two variables are somewhat complementary. But this would miss the opportunity to tidy things up into “everything is named `PERL_*` (except `PERLIO`)”.

   Maybe we want to just remove `PERLLIB`.

   Maybe we just shrug and accept the awkwardness.

1. Something needs to be done about version.pm presuming three components to a version number:
   ````
   perl -E 'say version->parse( 42.0 )->normal'
   v42.0.0
   ````

1. What do we do about the [perl5.git](https://github.com/Perl/perl5) repository name?

## History / Acknowledgements

It all began as [a tongue-in-cheek remark on Reddit](https://www.reddit.com/r/perl/comments/1f9bbhy/other_than_raku_are_there_any_serious_plans_for_a/llvsfjc/?context=3). The details of the [Specification](#specification) owe much to Graham Knop and the [Rationale](#rationale) to Philippe Bruhat, who as co-members of the PSC both contributed to thoughts across the entire document and eventually proofread it. After weeks of deliberation, the Motivation section finally crystallized during a cold early morning walk on Nov 13, 2024 and turned into writing on a riverside bench overlooking the Rhine.

## Copyright

Copyright (C) 2024, Aristotle Pagaltzis

This document and code and documentation within it may be used, redistributed and/or modified under the same terms as Perl itself.
