# Remove apostrophe ("Old package separator") as package separator

## Preamble

    Author:  Nicolás Mendoza <mendoza@pvv.ntnu.no>
    Sponsor:
    ID:      0015
    Status:  Draft

## Abstract

Remove support for ' (apostrophe) as package namespace separator.

## Motivation

* Removal of ambiguous syntax that complicates parsing and confuses users.

From perldoc perlmod
> The old package delimiter was a single quote, but double colon is now the preferred delimiter, in part because it's more readable to humans, and in part because it's more readable to emacs macros. It also makes C++ programmers feel like they know what's going on--as opposed to using the single quote as separator, which was there to make Ada programmers feel like they knew what was going on. Because the old-fashioned syntax is still supported for backwards compatibility, if you try to use a string like `"This is $owner's house"`, you'll be accessing `$owner::s`; that is, the `$s` variable in package owner, which is probably not what you meant. Use braces to disambiguate, as in `"This is ${owner}'s house"`.

## Rationale

* We have been warning against its usage for decades.
* It was there merely for easier adoption, but has been used very little.

## Alternatives

There are a few alternatives on how to disable this feature and changing the meaning of `'`

### String interpolation of identifiers:

* 1\. Treat `'` as text — `"$isn't" eq "${isn}'t"` and `"error: '$msg'" eq "error: '${msg}'"` 
* 1w. same with warning

* 2\. Treat `'` as part of identifier — `"$isn't" eq $isn't ne $isn::t` and `"error: '$msg'" ne "error: '${msg}'"`
* 2w. same with warning

* 3\. Syntax error — existing code supports warning, could die instead, see `toke.c` and look for `tick_warn`

* 4\. Current behaviour with warning — `"$isn't" eq "${isn::t}"` (warning) and `"error: '$msg'" eq "error: '${msg}'"` (no warning)

### Identifiers elsewhere (variables, function declarations etc.)

* a\. Treat `'` as part of identifier — `isn't()` would still work, but would need to be declared as `sub isn't { }` instead of `package isn; sub t {}`.

* b\. Syntax error

* c\. Current behaviour with warning

### Discussion

All of the above may be turned on or off using compiler flags, pragmas or extending warning::syntax

The current patch that I made some years ago is mostly like: `1 + b` but cold turkey removing as much as possible without any flags or switches.

I think `2 + a` would be an interesting approach to keep backwards-compatibility somewhat. 

Both `1` and `2` changes interpolation behaviour, but we have been warning about it since 5.28: https://github.com/Perl/perl5/commit/2cb35ee012cfe486aa75a422e7bb3cb18ff51336

`1w`, `2w`, `3` and `4` requires to keep around `tick_warn` in toke.c

`c` requires adding new code to warn correctly and keeping around ident parsing code: 

```
$ perl -wlE 'sub foo'"'"'bar { "lol"; }; print foo'"'"'bar();'
lol
```

## Specification

The feature should be rolled out in two steps, in two consecutive Perl stable releases

1. Add warning code to apostrophe usage in identifiers (outside of string interpolation) (alternative `4`) (as has been done for string interpolation already (alternative `c`)) [6]

2. Syntax error when using apostrophe as part of identifiers outside strings (alternative `b`), treat apostrophes inside strings as not a part of a variable when interpolating (alternative `1`) 

## Backwards Compatibility

This change will break backwards compatibility in two stable releases

### Known core / dual-life modules utilizing apostrophe as separator

The following issues are slowly being sorted out independently of this PPC. 

* ~~cpan/Term-Cap `isn't` in test [trivial to replace] https://github.com/jonathanstowe/Term-Cap/pull/13~~ [fixed]
* ~~dist/PathTools `isn't` in test [trivial to replace] https://github.com/Perl/perl5/pull/19865~~ [fixed]
* ~~cpan/autodie test in Klingon [not THAT trivial to replace, don't know Klingon] https://github.com/pjf/autodie/issues/115̃~~

The following issues might need to synchronize with the perl releases by checking versions or feature flags 

* ~~cpan/Test-Simple `isn::t` [would work wth 2 + b with small change, or state compatibility in docs]~~ [fixed] https://github.com/Test-More/test-more/commit/d619c1b0486422ac86b6c870241bf6138a041a8f
* cpan/Scalar-List-Utils `set_subname` implementation [need to check perl version in .xs code etc]

## References

* [1] Old package separator syntax #16270 (rt.perl.org#132485) https://github.com/Perl/perl5/issues/16270 
* [2] Discussion from 2009 at perl5-porters: https://markmail.org/message/gaux5xx5jlop3vmk
* [3] Discussion from 2017 at perl5-porters: https://markmail.org/message/ffmc2k3xafhwajys
* [4] Discussion from 2021 at perl5-porters: https://markmail.org/message/hms4gy4okkgvnk23
* [5] Simple grep of `isn't` usage on CPAN: `https://grep.metacpan.org/search?q=%5E%5Cs*isn%27t%7Cisn%27t%5Cs*%5B%28%24%40%25%5D&qd=&qft=*.t` (Note that there quite a few false positives, so I'd guess more like 30 pkgs)
* [6] Father Chrysostomos added a warning when used apostrophes are used inside string within a variable name: https://github.com/Perl/perl5/commit/2cb35ee012cfe486aa75a422e7bb3cb18ff51336

## Conditions to test

* from @epa https://github.com/Perl/perl5/issues/16270#issuecomment-544092952

```
…

Moreover, the need to parse it as a package separator sometimes affects its use as a string delimiter.

  sub foo { say $_[0] }
  foo"hi"; # works
  foo'hi'; # doesn't work

  foox"hi"; # String found where operator expected
  foo'"hi"; # Bad name after foo' (huh?)

It's not that hot as a synonym for :​: either​:

  sub foo' {} # Illegal declaration of subroutine main​::foo
  sub foo​:: {} # OK

  sub 'foo { say $_[0] }
  'foo 5; # Can't find string terminator "'"
  :​:foo 5; # OK
```

## Copyright

Copyright (C) 2022, Nicolas Mendoza

This document and code and documentation within it may be used, redistributed and/or modified under the same terms as Perl itself.
