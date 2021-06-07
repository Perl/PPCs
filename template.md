# Preamble

    Author:  A. U. Thor <author@example.com>
    Sponsor:
    ID:      ADOPTME-2038
    Status:  Exploratory

The preamble should be RFC-822 like headers.
Author, Sponsor and similar should be valid e-mail addresses, or CPAN IDs.

# Abstract

* 100 to 200 words summarising the entire RFC.
* The most important section is **Motivation**.

# Motivation

* What problem are we trying to solve?
* Why doesn't the existing syntax/functionality cover it?

# Rationale

* Why does the new syntax/functionality solve the problem described above?
* Why choose this solution, and reject others?

# Specification

Perl doesn't have a formal specification. Effectively it's defined by the documentation and the regression tests. Likely the best way to specify a new feature **is** to write the documentation for it.

You don't need to know the internals to do this - hence the *Author* can contribute directly by working on these, with the *Sponsor* acting as a mentor and guide as needed.

# Backwards Compatibility

If proposing syntax changes, think in terms of "can this be detected by"/"misunderstood by":

* Static tooling inspecting source code
* The Perl interpreter at compile time
* Only as a runtime error
* Subtle runtime behaviour changes that can't be warned about and break things

and how does this affect things like

* [`B::Deparse`](https://metacpan.org/pod/B::Deparse)
* [`Devel::Cover`](https://metacpan.org/pod/Devel::Cover)
* [`Devel::NYTProf`](https://metacpan.org/pod/Devel::NYTProf)
* [`PPI`](https://metacpan.org/pod/PPI) (hence [`Perl::Critic`](https://metacpan.org/pod/Perl::Critic) etc)

Also, is it possible to emulate this for earlier Perl versions (or at least, a useful and correct subset), even if slow? And if **not**, what sort of API or functionality is missing that if added would make similar future "polyfill"s possible?

# Security Implications

CVEs are not fun. Try to foresee problems.

# Examples

PEPs have this as "How to Teach This". That's a valid goal, but there are different audiences (from newcomers, to experienced Perl programmers unfamiliar with your plan).

Most of us are not experienced teachers, but many folks reading your RFC are experienced programmers. So probably the best way to demonstrate the benefits of your proposal is to take some existing code in the core or on CPAN (and not your own code) and show how using your new feature can improve it (easier to read, less buggy, etc)

# Prototype Implementation

Is there something that shows the idea is feasible, and lets other people
play with it? Such as

* A module on CPAN
* A source filter
* Hack the core C code - fails tests, but lets folks play

# Rejected Ideas

Why this solution/this syntax was better than the obvious alternatives.

We've seen before that there will be **F**requently **A**sked **Q**uestions.
eg *Why not have different behaviour in void context?*

Likely the answer is in the previous discussion **somewhere**, but most people won't stop to read all the comments. It needs to be easy to find, and updated as it becomes clear which questions are common.

Hence it **needs** to be in the RFC itself. Without this, the RFC process as a whole won't scale.

# Open Issues

Use this to summarise any points that are still to be resolved.

# Copyright

Copyright (C) 2038, A.U. Thor.

This document and code and documentation within it may be used, redistributed and/or modified under the same terms as Perl itself.
