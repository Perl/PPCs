# Make more elements of `%Config` relocatable

## Preamble

    Author:  Gianni Ceccarelli <dakkar@thenautilus.net>
    Sponsor:
    ID:      0037
    Status:  Draft

## Abstract

In addition to "relocatable `@INC`", provide relocatable
script-related paths.

## Motivation

By building a perl with `-Duserelocatableinc`, the "lib"-related paths
in `Config.pm` become relative to the location of the running `perl`
binary.

But some existing code
([`ExtUtils::Helpers`](https://metacpan.org/release/LEONT/ExtUtils-Helpers-0.028/source/lib/ExtUtils/Helpers/Unix.pm#L15),
[`ExtUtils::MakeMaker`](https://metacpan.org/release/BINGOS/ExtUtils-MakeMaker-7.78/source/lib/ExtUtils/MM_Unix.pm#L1314),
or [some tests inside
`Archive::Zip`](https://metacpan.org/release/PHRED/Archive-Zip-1.68/source/t/common.pm))
use `$Config{startperl}` or `$Config{perlpath}` which are not
relocatable. Trying to run that code on a relocated `perl` fails (by
which I mean, for example: build a relocatable perl, move its whole
tree somewhere else, try to install `Archive::Zip`, tests fail).

## Specification

Add another configuration option, `userelocatablebin` (or some better
name), that will cause these keys in `%Config` to be relocation-aware:

```
scriptdir
installbin
installprefix
installscript
perlpath
scriptdir
scriptdirexp
startperl
```

## Implementation notes and open questions

`installprefixexp` is already relocation-aware, why is `installprefix`
not? Would the same reason apply to `scriptdir` vs `scriptdirexp`?

The contents of `startperl` are not a path, and should be mangled
differently.

`bin` and `binexp` already start with `.../` on a relocatable build,
but they're not filtered through `relocate_inc()`… why?

Note that this change is not intended to magically fix the shebang
line (or equivalent) of any installed script. Setting
`-Dstartperl='/usr/bin/env -S perl'` may be a decent choice for most
UNIX-like systems, and could be documented together with
`userelocatablebin`.

## Backwards Compatibility

It's a new configuration option, nothing changes unless someone asks
for it.

## Security Implications

Same as the existing `-Duserelocatableinc`, which I don't think has
any.

## Copyright

Copyright (C) 2026, Gianni Ceccarelli.

This document and code and documentation within it may be used,
redistributed and/or modified under the same terms as Perl itself.
