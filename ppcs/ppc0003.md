# Issue a warning "-np better written as -p"

## Preamble

    Author:  
    Sponsor: Nicholas Clark <NWCLARK>
    ID:      0003
    Status:  Rejected

## Abstract

`perlrun` says *A **-p** overrides a **-n** switch.*

We warn for other overrides, such as variables declared twice. To be consistent we should warn for this one, if `-w` is on the command line.

## Motivation

We explicitly document that `-p` overrides `-n`. Calling the `perl` binary with both is not correct - the `-n` will be ignored. We could help users better by reporting their mistake to them, if they have opted into warnings.

## Rationale

For code, where what is written cannot make sense, we issue warnings. This is a similar case, just with command line flags

* Issuing a warning would make a programmer aware of the problem
* Issuing a warning would be consistent with our other use of warnings

## Specification

Invoking `perl` with all three of `-p`, `-w` and `-n` in any order or grouping should issue the warning

    -np better written as -p

## Backwards Compatibility

This is **hard** to assess.

We can search CPAN for representative use of Perl **code**. With the demise of Google codesearch, there isn't a good way to search for command-line use cases of `perl`. Is it viable to search Debian's source archives? Or the FreeBSD ports tree?

Issuing a warning **might** break existing users' code, and they would be grumpy, because it was working, it would still work without a trapped warning, and we have no intention of changing the behaviour

It might "break" existing code, where users view "you're making new noise" as breakage, but (of course) everything still works.

It might not make much difference - do we have any feel for how many scripts invoking `perl` as a command-line *better sed*/*better awk* actually use `-w` **at all**?

## Security Implications

It's unclear whether there is any (direct) security implication.

## Examples

I believe that it's exhaustively covered in the *Specification*.

## Future Scope

We don't intend to make this warning "fatal".

## Rejected Ideas

The PSC thanks the author for making a useful suggestion, but has decided that this change is not worth making.

The discussion was useful. We note

1) https://codesearch.debian.net/ lets one search for typical command-line invocations of `perl`. This is useful when considering changes to options or option parsing.
2) Command line options are processed as a result of all of
   1) Actual command line options
   2) Options found on a `#!` line
   3) Options in `PERL5OPT`

Hence "combinations" of options might have happened as a result of the internal unification of these, not because the programmer wrote something directly.

Using `-p` and `-n` is not causing one to be ignored.  "-p" is "-n and also more". The gain to not writing both is saving a single (or perhaps 2-3) keystrokes. If the user has written both, their program will work just as well as one with only one. Removing one would require **more** work. Also, we'd need a warning message sufficiently clear that the user knew, immediately, to go remove the "n", but also that no change was really **required**. The benefit is very close to nil.

This is **not** the same as the `@F[1]` warning. A simple test case doesn't suggest this:

    $ perl -wE 'our @A = (1,2,3); say @A[0]'
    Scalar value @A[0] better written as $A[0] at -e line 1.
    1

But there is a massively important one that's demonstrated by a different code snippet:

    $ perl -wE 'our @A; @A[0] = foo(); sub foo { warn wantarray ? "1\n" : "0\n" }'
    1
    $ perl -wE 'our @A; $A[0] = foo(); sub foo { warn wantarray ? "1\n" : "0\n" }'
    0

Using `@A[0]` creates a list context. Using `$A[0]` creates a scalar context. It's a setup for potentially deep confusion based on sigil variance, worth the warning.

## Open Issues

## Copyright

Copyright (C) 2021, Nicholas Clark

This document and code and documentation within it may be used, redistributed and/or modified under the same terms as Perl itself.
