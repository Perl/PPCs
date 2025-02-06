# `undef`-aware Equality Operators

## Preamble

    Author:  Paul Evans <PEVANS>
    Sponsor:
    ID:      0030
    Status:  Proposed

## Abstract

Adds new infix comparison operators that are aware of the special nature of the `undef` value, comparing it as distinct from the number zero or the empty string.

## Motivation

Perl has two sets of comparison operators; one set that considers the stringy nature of values, and another that considers numerical values. When comparing against `undef`, the stringy operators treat undef as an empty string, and the numerical operators treat it as zero. In both cases, a warning is produced:

```
Use of uninitialized value in string eq at ...
```

Sometimes it is useful to consider that `undef` is in fact a different value, distinct from any defined string or number - even empty or zero. Currently in Perl, to check if a given value (which may be `undef`) is equal to a given string value, the test must make sure to check for definedness:

```perl
if( defined $x and $x eq $y ) { ... }
```

Furthermore, if the value `$y` can also be undefined, it may be desired that in that case it matches if `$x` is also undefined. In that situation, the two cases must be considered individually:

```perl
if( (!defined $x and !defined $y) or
    (defined $x and defined $y and $x eq $y) ) { ... }
```

Countless bugs across countless modules have been caused by failing to check in such a way and using code such as simply `$x eq $y` in these circumstance.

By providing new comparison operators that have different behaviour with undefined values, Perl can provide more convenient choice of behaviours for authors to use.

## Rationale

(explain why the (following) proposed solution will solve it)

## Specification

A new operator, named `equ`, which has the same syntax as regular string equality `eq`, but semantics identical to that given above:

```perl
$x equ $y

# Equivalent to  (!defined $x and !defined $y) or 
#                    (defined $x and defined $y and $x eq $y)
```

In particular, given two `undef` values, this operator will yield true. Given one `undef` and one defined value, it yields false, or given two defined values it will yield the same answer that `eq` would. In no circumstance will it provoke a warning of undefined values.

Likewise, a new operator named `===`, which provides the numerical counterpart:

```perl
$x === $y

# Equivalent to  (!defined $x and !defined $y) or 
#                    (defined $x and defined $y and $x == $y)
```

Note that while the `===` operator will not provoke warnings about undefined values, it could still warn about strings that do not look like numbers.

## Backwards Compatibility

As these infix operators are currently syntax errors in existing versions of Perl, it is not required that they be feature-guarded.

However, it may still be considered useful to add a feature guard in order to provide some compatibility with existing code which uses the `Syntax::Operator::Equ` CPAN module. Likely this would be named `equ`:

```perl
use feature 'equ';
```

Once the feature becomes stable it is likely this would be included in an appropriate `use VERSION` bundle, so users would not be expected to request it specifically in the long-term.

## Security Implications

No new issues are anticipated.

## Examples

## Prototype Implementation

This operator is already implemented using the pluggable infix operator support of Perl version 5.38, in CPAN module [`Syntax::Operator::Equ`](https://metacpan.org/pod/Syntax::Operator::Equ).

## Future Scope

Considering further the possible idea to provide a `match/case` syntax inspired by [`Syntax::Keyword::Match`](https://metacpan.org/pod/Syntax::Keyword::Match), this operator would be a useful addition in combination with that as well, allowing dispatch on a set of fixed values as well as `undef`:

```perl
match( $x : equ ) {
    case(undef) { say "The value in x is undefined" }
    case("")    { say "The value in x is empty string" }
    case("ABC") { say "The value in x is ABC" }
    ...
}
```

## Rejected Ideas

* It is not possible to unambiguously provide extensions of the ordering operators `cmp` and `<=>` in a similar way, because this requires a decision to be made as to the sorting order of undefined values, compared to any defined string or number. While it may be natural to consider that undef sorts before an empty string, there is no clear choice on where undef would sort compared to (signed) numbers. Would it be before any number, even negative infinity? This would place it far away from zero.

* Likewise, because it is not possible to provide an undef-aware version of `cmp` or `<=>`, the ordering comparison operators of `le`, `lt` and so on are also not provided.

## Open Issues

* Should we also provide negated versions of these operators? While much rarer in practice, it may be useful to provide a "not equ", perhaps spelled `nequ` or `neu`; and likewise `!===` or `!==` for the numerical version. These do not suffer the sorting order problem outlined above for more general comparisons.

* As an entirely alternate proposal, should we instead find ways to apply behaviour-modifying flags to the existing operators? That is, rather than adding a new `equ` and `===` could we instead consider some syntax such as `eq:u` and `==:u` as a modifier flag, similar to the flags on regexp patterns, as a way to modify operators? This would be extensible in a more general way to more operators, while also allowing more flexible flags in future, such as for instance a case-ignoring string comparison to be spelled `eq:i`. This alternate proposal is be the subject of an [alternate PPC document](ppc0031-metaoperator-flags.md).

* How to pronounce the name of this new operator? I suggest "ee-koo", avoiding the "you" part of the sound.

## Copyright

Copyright (C) 2024, Paul Evans.

This document and code and documentation within it may be used, redistributed and/or modified under the same terms as Perl itself.
