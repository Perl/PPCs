# No Longer Require a True Value at the End of a Module

## Preamble

    Author:  Curtis "Ovid" Poe <curtis.poe@gmail.com>
    Sponsor:
    ID:      0018
    Status:  Draft

## Abstract

This PPC proposes a feature which, when used, eliminates the need to end a Perl
module with the conventional "1" or other true value.

## Motivation

Eliminate the need for a true value at the end of a Perl file.

## Rationale

There's no need to have a true value be hard-coded in our files that we
`use`. Further, newer programmers can get confused because sometimes code
_doesn't_ end with a true value but nonetheless compiles just fine because
_something_ in the code returned a true value and the code compiles as a
side-effect.

## Specification

First, a new `feature` is added:

```perl
use feature 'module_true';
```

Then, *whenever* a module is loaded with `require` (or an equivalent, like
`use`), the "croak if false" test is skipped if the `module_true` feature was
in effect at the last statement executed in the required module.

## Backwards Compatibility

There are no compatibility concerns I'm aware of because we're only suggesting
changing behaviour in the presence of a newly-added feature that is not
present in any existing code.

## Security Implications

None expected.

## Examples

Imagine this module:

```perl
package Demo1;
use feature 'module_true';

sub import {
  warn "You imported a module!\n";
}
```

When loaded by `require` or `use` anywhere in perl, this would import
successfully, despite the lack of a true value at the end.

This module shows an (almost certainly never useful) way to croak anyway:

```perl
package Demo2;
use feature 'module_true';

return 1 if $main::test_1;
return 0 if $main::test_2;

{
  no feature 'module_true';
  return 0 if $main::test_3;
}
```

In this example, the only case in which requiring Demo2 would fail is if
`$main::test_3` was true.  The previous `return 0 if $main::test_2` would still
be within the scope of the `module_true` feature, so the return value would be
ignored.  When `0` is returned outside the effect of `module_true`, though, the
old behavior of testing the return value is back in effect.

## Prototype Implementation

There is a prototype implementation at [true](https://metacpan.org/pod/true).

## Future Scope

Due to this being a named feature, this can eventually be the default behavior
when `use v5.XX;` is used.

## Rejected Ideas

It's been discussed that we should return the package name instead. This
supports:

```perl
my $obj = (require Some::Object::Class)->new;
```

However, per haarg:

> Changing the return value of require seems like a separate issue from what
> this PPC wants to address.
>
> If you wanted require to always return the same value, and for that value to
> come from the file, you need a new location to store these values. This
> would probably mean a new superglobal to go along with %INC. And it would
> usually be useless because most modules return 1. I don't think this is a
> very good idea.
>
> If you wanted require to always return the package name, it's a separate
> issue from this PPC because that means ignoring the return value from the
> file. It also presents a problem because require doesn't actually take
> package names.  It takes file path fragments. Foo::Bar is translated to
> "Foo/Bar.pm" at parse time. This would then need to be converted back to a
> package name, or do something else. I don't think this is a good idea
> either.
>
> Instead, it's probably best addressed with a builtin::load or similar
> routine that accepts a package as a string. This has been discussed in the
> past, and solves other problems. Module::Runtime has a use_module function
> that behaves like this, returning the package name.

Thus, we prefer simply returning `true` (or `1`).

## Copyright

Copyright (C) 2022, Curtis "Ovid" Poe

This document and code and documentation within it may be used, redistributed
and/or modified under the same terms as Perl itself.
