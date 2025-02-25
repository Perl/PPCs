# Meta-operator Flags to Modify Behaviour of Operators

## Preamble

    Author:  Paul Evans <PEVANS>
    Sponsor:
    ID:      0031
    Status:  Proposed

## Abstract

Defines a new syntax for adding behaviour-modifying flags to operators, in order to extend the available set of behaviours.

## Motivation

This idea came out of discussions around the proposal to add new `equ` and `===` operators, which are aware of the special nature of the `undef` value. The idea is that rather than adding just two new special-purpose operators, a far more flexible and extensible idea may be to allow syntax for putting flags on the existing operators.

The central idea here is that existing operators can be modified by a set of single-letter suffix flags, that could be considered slightly similar to the concept of regexp pattern flags.

## Rationale

(explain why the (following) proposed solution will solve it)

## Specification

Following the name of a regular value-returning infix operator, an optional set of flag letters can be supplied after a separating colon:

```perl
$x eq:u $y
```

In terms of syntax, these are flags that alter the runtime behaviour of the operator, and do not change the way it parses at compile-time. The particular behaviour being changed would depend on the operator and the flag applied, but in order to make it easy to learn and use, some consistency should be applied to the choice of flags.

The `:u` flag alters the behaviour of a scalar-comparing operator (`eq`, `ne`, `==`, `!=`) in the same way described in the `equ` proposal; namely, that these operators now consider that `undef` is equal to another `undef` but unequal to any defined value, even the empty string or number zero.

```perl
$x eq:u $y

# Equivalent to  (!defined $x and !defined $y) or 
#                    (defined $x and defined $y and $x eq $y)

$x ==:u $y

# Equivalent to  (!defined $x and !defined $y) or 
#                    (defined $x and defined $y and $x == $y)
```

The `ne` and `!=` operators can likewise be modified to produce the negation of these ones.

```perl
$x ne:u $y

# Equivalent to  not( $x eq:u $y )
#          i.e.  (defined $x xor defined $y) or
#                    (defined $x and defined $y and $x ne $y)

$x !=:u $y

# Equivalent to  not( $x ==:u $y )
#          i.e.  (defined $x xor defined $y) or
#                    (defined $x and defined $y and $x != $y)
```

Another flag, `:i`, modifies the string comparison operator making it case-insensitive, acting as if its operands were first passed through the `fc()` function:

```perl
$x eq:i $y

# Equivalent to  fc($x) eq fc($y)
```

There is no equivalent on a numerical comparison; attempting to apply the flag here would result in a compile-time error.

Finally, note that these two flags can be combined, and in that case the order does not matter.

```perl
$x eq:ui $y
$x eq:iu $y

# Equivalent to  (!defined $x and !defined $y) or 
#                    (defined $x and defined $y and fc($x) eq fc($y))
```

## Backwards Compatibility

As these notations are currently syntax errors in existing versions of Perl, it is not required that they be feature-guarded.

In particular, there is no potential that a colon-prefixed identifier name is confused with a statement label, because a statement label must appear at the beginning of a statement, whereas an infix operator must appear after its left-hand operand:

```perl
use v5.xx;

eq:u $y;     # A call to the u($x) function, in a statement labeled as 'eq:'

$x eq:u $y;  # An expression using the eq operator modified with :u
```

Similarly, there is no ambiguity with the ternary conditional operator pair `? :`, because the colon for these flags only appears when the right-hand operand to an infix operator is expected: 

```perl
use v5.xx;

my $result = $stringy ? $x eq:u $y : $x ==:u $y;
#  Parser always knows these ^------------^ are meta-operator flags
```

Finally, there is no ambiguity with attribute syntax - even when considering extension
proposals that would extend the attribute syntax to more sites - again because of the non-overlap between applicable situations. In fact it could be argued that these flags are a similar and related syntax; appearing to apply some sort of adverb-like "attribute" to an operator. Where full attributes have identifiers as names and can take optional values, these flags are bundled into single-letter names without options.

## Security Implications

No new issues are anticipated.

## Examples

A few examples are given in the specification above.

## Prototype Implementation

It is currently not thought possible to prototype this exact syntax as a CPAN module, because it requires extending the parser in ways that neither the `PL_keyword_plugin` nor `PL_infix_plugin` extension mechanisms can achieve.

If it were considered necessary to prototype these on CPAN in some manner, it would be possible to define some new operator names, such as `eqx` for "extensible `eq`", which would then allow such suffix flag notation on them. While it would then be possible to experiment with possible behaviours of new flag letters, it would remove most of the advantage of the idea in that it uses (and applies to) existing operator names, rather than inventing new ones. It is therefore unlikely to be worth performing such an experiment.

## Future Scope

As this proposal introduces the idea of adding flags named by letters to existing Perl operators, there is clearly much more potential scope to define other modifier letters to these or other operators.

However, care should definitely be taken to limit these new definitions to genuinely useful combinations that could not easily be achieved by other means. With over 30 infix operators and 52 single-letter flags available, the number of possible ideas far exceeds the number of combinations that would actually be useful in real-world situations, and sufficiently motivating to justify adding more things for users to learn and recognise.

In particular, care should be taken that any new flag proposals do not attempt to reuse existing flag letters to have other meanings. For example, if the `:i` flag is added to other operators its meaning should be somehow compatible with the "ignore-case" of `eq`, rather than serve some other unrelated purpose.

## Rejected Ideas

### Using the `/` symbol as a flag separator

It may look similar to the regexp pattern flags and suggest a similar purpose, but it becomes more of a problem to suffix attach these onto existing numerical operators. While not suggested by this document, consider the hypothetical case of adding a flag to the division operator such as `z` to alter how it behaves on division-by-zero. The `:` symbol allows this as `/:z` whereas the `/` symbol leads to the problematic `//z` operator. There are no existing infix operators that use the `:` symbol.

## Open Issues

### Is this use of the `:` symbol still too much?

Considered alone in this proposal, the syntax is unambiguous and makes sense with the existing Perl syntax. In particular, the similarity with attributes is noted. However, when considering such other possible ideas as the `in:OP` hyper-operator, or the `match/case` syntax to replace the problematic `given/when` and smartmatch, there seems to be an explosion in the possible meanings of the colon character:

```perl
if( $str in:eq:u @possible_matches ) { ... }
#          ^--^  these two colons mean conceptually different things
```

```perl
match( $value : eq:u ) {
#             ^---^  these two colons also mean conceptually different things
    case(undef) { say "It was undefined" }
    case("")    { say "It was empty string" }
    default     { say "It was a non-empty string" }
}
```

In each case here the Perl parser (or another other static analysis tooling, such as syntax highlighters) should have no trouble understanding the various meanings. However, it may cause some confusion to human readers as to what the different uses all mean. I have tried to be consistent with the use of whitespace before or after the colon symbol to hint at its various different meanings in the examples above. These are currently purely informational hints to human readers, and not considered significant by the Perl parser.

There aren't many other viable choices of symbol, at least not while remaining both within the character set provided by ASCII, and not being sensitive to the presence of whitespace. This may be the inevitable pressure of trying to add more features to such an operator-rich language as Perl, while attempting to stick to ASCII and whitespace-agnostic syntax. At some point we may have to accept that we cannot add new operator-based syntax without either accepting the use of non-ASCII Unicode operators, or using presence-of-whitespace hints to disambiguate different cases.

## Copyright

Copyright (C) 2024, Paul Evans.

This document and code and documentation within it may be used, redistributed and/or modified under the same terms as Perl itself.
