# `${^ENGLISH_NAME}` aliases for punctuation variables

## Preamble

    Author:  Graham Knop <haarg@haarg.org>
    ID:      0014
    Status:  Proposed

## Abstract

Provide aliases for all punctuation variables with the form
`${^ENGLISH_NAME}`, with the english name taken from the aliases currently
provided by `English.pm`.

## Motivation

Punctuation variables are not very friendly to use. Many people consider them
ugly, and they require you to memorize all of the relevant variables. It would
be valuable to provide better names for these variables as part of the core
language. `English.pm` does exist as an attempt at this but brings its own
issues. The English names look like `$ENGLISH_NAME`, which is syntactically
the same as a package variable that a user might define themselves. While not
exactly rare, use of `English.pm` is not particularly common either. Many
seasoned Perl programmers would not know the English names for many commonly
used punctuation variables without consulting a reference.

## Rationale

Some of the newer superglobals are spelled like `${^ENGLISH_NAME}`. Providing
aliases to all of the punctuation variables would allow them to be more self
documenting, like the `English.pm` names, but without the being syntactically
identical to a normal package variable. Being spelled like `${^ENGLISH_NAME}`
makes it obvious that this is a special variable. This makes the magic
behavior more obvious, and helps point to the correct reference material.
Since there are already several "punctuation" variables with this spelling, it
should be familiar to Perl programmers. It also means the variables are always
available without needing to load an additional module.

The names used by `English.pm` could mostly be re-used with the new spelling.
This should help any Perl programmer that has preferred `English.pm` to
understand the new form.

## Specification

For each variable listed in perlvar which includes an `English.pm` name like
`$OS_ERROR`, provide an alias named `${^OS_ERROR}` as part of the core
language.

## Backwards Compatibility

There won't be any conflict regarding syntax, as this is already valid syntax,
and already referring to super-globals. This does present an issue, as
variables with this syntax are exempt from strict. Attempting to use these
variables will fail silently when run on older perls, or when misspelling the
variable.

A shim could be created to allow using these variables on older perls. It may
not be able to be 100% compatible.

## Security Implications

Many of the punctuation variables relate to system state or errors.
Misspelling a variable like `${^EVAL_ERROR}` (aka `$@`) could lead to an error
being ignored rather than handled, which could lead to security concerns.

## Examples

```perl
local $_;
local ${^ARG};

my $pid = $$;
my $pid = ${^PID};

my $error = $@;
my $error = ${^EVAL_ERROR};

my %signals = %SIG;
my %signals = %{^SIG};
```

## Prototype Implementation

An initial attempt at a backwards compatibility shim:
https://github.com/haarg/English-Globals/blob/master/lib/English/Globals.pm

A more complete implementation written in XS:
https://metacpan.org/pod/English::Name

Another implementation of the same idea from 2017, in pure perl:
https://metacpan.org/pod/English::Control

## Future Scope

Most punctuation characters are already used for special variables, so a typo
can't really be caught. With english names, there is a greater risk of typos,
but also a greater possibility of catching them. Possibly variables with this
format could have strict applied to them.

Providing these aliases could make it possible to deprecate and remove some
lesser-used punctuation variables, freeing them up for use as syntax rather
than variables. This was previously done with the `$*` variable.

## Rejected Ideas

`English.pm` serves as a previous example of trying to make punctuation
variable use more friendly, but has various caveats as covered in the
"Motivation" section.

Spelling these variables like `$^ENGLISH_NAME` is not possible because
variables of the form `$^E` exist. When interpolating, `"$^OS_ERROR"` is
parsed as `"${^O}S_ERROR"`. If this brace-less form was still desired, this
PPC would still be a prerequisite. When interpolating, a delimited form is
required to allow word characters to follow the variable. This delimited form
would be the same `${^ENGLISH_NAME}` form proposed by this PPC.

`$*ENGLISH_NAME` has been proposed, but conflicts with glob dereferencing.
Outside string interpolation, that variable syntax is currently invalid and
thus could be used. But when interpolating, a delimited form is needed. This
would be `${*ENGLISH_NAME}`, but that syntax is already valid, as a scalar
dereference of a glob. Trying to use `$*{ENGLISH_NAME}` is also a problem
because it is currently valid syntax, refering to an element in the `%*` hash.

## Open Issues

  - None at this time

## Copyright

Copyright (C) 2022, Graham Knop.

This document and code and documentation within it may be used, redistributed and/or modified under the same terms as Perl itself.
