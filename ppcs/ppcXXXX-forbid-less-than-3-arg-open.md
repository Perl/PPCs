# Forbid `open` with less than three arguments

## Preamble

    Author:  Philippe Bruhat (BooK) <philippe@bruhat.net>
    Sponsor:
    ID:      
    Status:  Draft

## Abstract

Add a new unfeature to the `feature` module, to disable
two-argument `open`.

One-argument and two-argument `open` are considered "legacy usage", and
are insecure, as they pick the open mode from the provided filename. A
filename starting or ending with a `|` can lead to a Remote Code
Execution issue when fed to a one-argument or two-argument `open`.

The feature name would be `two_arg_open`, and typical usage would be:

```perl
no feature 'two_arg_open';
```

It would make using two-argument `open` a fatal error at compile time.

The most common use case should be in a feature bundle, as part of a
`use VERSION` statement.

## Motivation

The main reason to want to disable two-argument open is security. As
stated above, a filename starting or ending with a `|` will shell out to
an external command, leading to potential remote code execution.

The `<<>>` operator added in Perl 5.22 was mitigating the issue in
one-liners and code using the diamond operator by turning the
implicit `open` over `@ARGV` values into a three-argument open (using
the `<` mode).

The `bareword_filehandles` feature already disables one-argument `open`,
since it depends on using a bareword filehandle and using a package
variable with the same name. It is part of the `":5.38` feature bundle.

This feature would be the last nail in the coffin of "less tha  three-argument `open`".

## Rationale

`open` is one of the most complex builtin in Perl, so much so that it
has its own tutotrial, `perlopentut`.

In perlfunc, it has the longuest list of possible ways to call it, at 5
entries (on par with`use` and `no`).

    open FILEHANDLE,MODE,EXPR
    open FILEHANDLE,MODE,EXPR,LIST
    open FILEHANDLE,MODE,REFERENCE
    open FILEHANDLE,EXPR
    open FILEHANDLE

The last two items in the list are insecure and documented as
"legacy usage", with the documentation saying:

> New code should favor the three-argument form of "open"
> over this older form. Declaring the mode and the
> filename as two distinct arguments avoids any confusion
> between the two.

The documentation also euphemistically claims that:

> It is safe to use the two-argument form of `open` if the filename
> argument is a known literal.

when it should actually say that the two-argument form of `open` is
insecure in any other case.

Since one-argument `open` is already forbidden by disabling
`bareword_filehandles`, the obvious name for the feature is
`two_arg_open` (or maybe `two_argument_open`, given the recent
`apostrophe_as_package_separator` has shown name length is not really
a concern for unfeature names).

When the need for "magic open" arise, it will always be possible to write:

```perl
use feature 'two_arg_open';
```

## Backwards Compatibility

This is backwards compatible, by virtue of being feature-gated.

## Security Implications

This feature, once disabled in a feature bundle, will *increase* security.

## Examples

Compare:

```perl
# works, and a security issue if $file is 'rm -fr ~ |'
open my $fh, $file
  or do "Can't open $file: $!";
```

with:

```perl
# die at compilation time: no security risk
no feature 'two_arg_open';
open my $fh, $file
  or do "Can't open $file: $!";
```

## Prototype Implementation

None yet.

## Future Scope

Since the null filehandle `<>` (diamond operator) uses the two-argument
form of `open`, it is also insecure by default.

Given that `<<>>` doesn't seem to have taken on, it could be considered,
under  `no feature 'two_arg_open'`, to either:
* implicitly turn `<>` into `<<>>`,
* warn about `<>` being insecure, and `<<>>` being preferred.

Again, enabling the feature would allow to opt in the insecure behaviour.

## Rejected Ideas

None.

## Open Issues

None.

## Reference

The long way towards a safer `open`:

* Three-argument open was introduced in March 2000, with Perl 5.6.0
  <https://perldoc.perl.org/5.6.0/perldelta#open()-with-more-than-two-arguments>
* `<<>>` was introduced in June 2015, with Perl 5.22.0
  <https://perldoc.perl.org/5.22.0/perldelta#New-double-diamond-operator>
* The `bareword_filehandle` unfeature was introduced in May 2021, with
  Perl 5.34.0
  <https://perldoc.perl.org/feature#The-'bareword_filehandles'-feature>

The warnings have been piling up for quite some time:

* [Perl::Critic::Policy::InputOutput::ProhibitTwoArgOpen](https://metacpan.org/pod/Perl::Critic::Policy::InputOutput::ProhibitTwoArgOpen) (2005)
* [perl open() injection prevention](https://stackoverflow.com/questions/26614348/perl-open-injection-prevention) (2015)
* [IDS31-PL. Do not use the two-argument form of open()](https://cmu-sei.github.io/secure-coding-standards/sei-cert-perl-coding-standard/rules/input-validation-and-data-sanitization-ids/ids31-pl/) (2021)
* [Are you still using the 2-argument open?](https://security.metacpan.org/2025/06/06/two-arg-open.html) (2025)
* [CVE-2026-11526: GD versions before 2.86 for Perl allow OS command injection and file overwrite via a 2-arg open() of filename arguments in _make_filehandle](https://lists.security.metacpan.org/cve-announce/msg/41004664/) (2026)

A recent RCE caused by two-argument `open`:

* [The 23-Year-Old Flaw at the Heart of CPAN: An RCE in the Perl Infrastructure](https://irsl.medium.com/the-23-year-old-flaw-at-the-heart-of-cpan-an-rce-in-the-perl-infrastructure-fba07a773c6f)
* [Arbitrary code execution by uploading a crafted distribution tarball](https://github.com/andk/pause/security/advisories/GHSA-p262-w3xr-phm4)

## Copyright

Copyright (C) 2026, Philippe Bruhat (BooK).

This document and code and documentation within it may be used,
redistributed and/or modified under the same terms as Perl itself.
