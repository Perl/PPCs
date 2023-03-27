# built-in functions for checking the original type of a scalar

## Preamble

    Author:  Graham Knop <haarg@haarg.org>
    Sponsor:
    ID:      0017
    Status:  Accepted

## Abstract

This PPC proposes adding new builtin functions created_as_string and
created_as_number for checking the original form of non-scalar references.
Specifically, checking if a value was created as a string or a number. This
is primarily meant for use by serializers.

## Motivation

Perl itself is operator typed and is not generally meant to distinguish
between values like 1 and "1". However, many external systems do care about
these types, and many times these values need to be transferred through perl
without anything like a schema to enforce these types. This makes it important
for things like serializers to be able to track the "origin" type of a value.

Classically, perl scalars have no set type. They may be created as strings or
numbers, but based on their use they will internally store alternate forms of
the value. If these alternate forms are considered "accurate", the
corresponding flag will be set. This makes it impossible to know for certain
how some values were created, as they will have both string and number forms
that are flagged as "accurate".

In perl v5.36, this has changed. The POK flag is now only set for values that
are created as strings. Values that are created as strings may gain IOK or NOK
flags based on their use, but since numbers will never gain the POK flag, this
can still be used to detect numbers.

Exposing this information via builtin functions will allow pure perl code to
accurately serialize values.

## Rationale

While the flags on scalars can be checked via the B module, using it requires
an understanding of perl internals, and it is slow and uses a fairly large
amount of memory. Being able to interpret the flags to know string vs number
is also not entirely obvious.

The functions are intentionally using longer less convenient names, with the
hope that this will discourage their use for type checks within perl. Inside
perl, 1 and "1" are meant to be treated equivalently. It is only when
transferring data outside perl that these are meant to be used.

`created_as` is meant to only tell how a value was created, without implying
anything further about what the value "is". `created` in this case means any
modification of the value, as any modification of a value requires creating
the new, modified value.

## Specification

    created_as_string VALUE

`created_as_string` takes a single scalar argument.

It returns a boolean value representing if the value was originally created as
a string. It will return false for references, numbers, booleans, and undef.
Internally, this would be `SvPOK(sv) && !SvIsBOOL(sv)`.

    created_as_number VALUE

`created_as_number` takes a single scalar argument.

It returns a boolean value representing if the value was originally created as
a number. It will return false for references, strings, booleans, and undef.
Internally, this would be `SvNIOK(sv) && !SvPOK(sv) && !SvIsBOOL(sv)`.

## Backwards Compatibility

There should be no significant backwards compatibility concerns.
`created_as_string` and `created_as_number` will be imported only when
requested. Static analysis tools may need to be updated.

As these functions rely on the changes to how SV flags get set, it isn't
possible to accurately implement these on older perls. Existing serializers
use heuristics that will usually give accurate answers. One of these heuristics
could be used to provide a compatible polyfill.

## Security Implications

Nothing specific predicted.

## Examples

  use builtin qw(created_as_number created_as_string);

  my $value1 = "1";
  my $string1 = created_as_string $value1; # true
  my $number1 = created_as_number $value1; # false

  my $value2 = 1;
  my $string2 = created_as_string $value2; # false
  my $number2 = created_as_number $value2; # true

  my $value3 = "1";
  my $used_as_number3 = 0+$value3;
  my $string3 = created_as_string $value3; # true
  my $number3 = created_as_number $value3; # false

  my $value4 = 1;
  my $used_as_string4 = "$value4";
  my $string4 = created_as_string $value4; # false
  my $number4 = created_as_number $value4; # true

## Prototype Implementation

None.

## Future Scope

It is possible that in the future, the NOK or IOK flags will only be set for
values created as numbers. That would mean the implementation of these
functions may change, but should not impact the results returned.

Additionally, if the NOK and IOK flags were only set for values created as
integers or created as floats, it may be possible to provide additional
functions `created_as_integer` and `created_as_float` to distinguish these
types. Some serializers care about this distinction as well.

## Rejected Ideas

Various alternative names for these functions have been proposed.

`is_string` and `is_number` were the initial discussed functions, but these
names imply stronger typing than perl is really meant to have. They would more
strongly encourage using these functions for type checking within perl, rather
than the usually more appropriate `looks_like_number`.

`was_originally_string` and `was_originally_number` were also proposed, but
`originally` carries the implication that the values may no longer be those
things. The use of an adverb also feels wrong stylistically.

## Open Issues

None?

## Copyright

Copyright (C) 2022, Graham Knop.

This document and code and documentation within it may be used, redistributed
and/or modified under the same terms as Perl itself.
