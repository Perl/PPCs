# A feature to automatically "yield true" at the end of a file

## Preamble

    Author:  Curtis "Ovid" Poe <curtis.poe@gmail.com>
    Sponsor:
    ID:      OVID-0018
    Status:  Draft

## Abstract

This RFC proposes a `yield_true` feature. When used, the current Perl _file_
containing `use feature 'yield_true';` will automatically return a true value
after successful compilation, eliminating the need for a "1" (or other true
value) at the end of the file.

## Motivation

Eliminate the need for a true value at the end of a Perl file.

## Rationale

There's no need to have a true value be hard-coded in our files that we
`use`. Further, newer programmers can get confused because sometimes code
_doesn't_ end with a true value but nonetheless compiles just fine because
_something_ in the code returned a true value and the code compiles as a
side-effect.

## Specification

    use feature 'yield_true';

Code using the above does not need to return a magic true value when compiled.

If the module explicitly returns a false value, module loading will fail as it
does now. If the module author wants the module to fail to load under certain
conditions, they should die with an appropriate error message rather than
returning false.

## Backwards Compatibility

There are no compatibility concerns I'm aware of.

## Security Implications

None expected.

## Examples

None. See above.

## Prototype Implementation

There is a prototype implementation at [true](https://metacpan.org/pod/true).

## Future Scope

Due to this being a named feature, this can eventually be the default behavior
when `use v5.XX;` is used.

## Rejected Ideas

It's been discussed that we should return the package name instead. This
supports:

    my $obj = (require Some::Object::Class)->new;

However, per haarg:

> Changing the return value of require seems like a separate issue from what
> this RFC wants to address.
>
> If you wanted require to always return the same value, and for that value to
> come from the file, you need a new location to store these values. This
> would probably mean a new superglobal to go along with %INC. And it would
> usually be useless because most modules return 1. I don't think this is a
> very good idea.
>
> If you wanted require to always return the package name, it's a separate
> issue from this RFC because that means ignoring the return value from the
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

## Open Issues

* Should returning a false value be ignored?
* Should any explicit return be an error?

## Copyright

Copyright (C) 2022, Curtis "Ovid" Poe

This document and code and documentation within it may be used, redistributed
and/or modified under the same terms as Perl itself.
