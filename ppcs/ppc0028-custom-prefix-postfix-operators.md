# Custom Pre- and Post-fix Operators

## Preamble

    Author:  Paul Evans <PEVANS>
    Sponsor:
    ID:      0028
    Status:  Exploratory

## Abstract

Expand the current `PL_infix_plugin` and mechanism and `XS::Parse::Infix` to support prefix and postfix operators as well.

## Motivation

Perl version 5.38 added a new internal mechanism, named `PL_infix_plugin`, which allows external modules to assist the parser to define new infix operators, either named like identifiers, or with sequences of non-identifier characters. This mechanism is exposed to XS module authors via the `XS::Parse::Infix` module, allowing new operator behaviours to be created. There are several on CPAN using this mechanism, mostly named within the `Syntax::Operator::*` space.

While originally intended to allow CPAN modules to experiment with new kinds of comparison or matching operators, such as `equ`, `eqr` and `is`, the mechanism has also been used for other cases such as infix zip and mesh operations. It may further be the case that other new kinds of operators would be useful, if Perl allowed prefix and postfix operators, in addition to the current infix kind.

Additionally it may also be useful to support nullary operators (i.e. operators that take no operand), as a minor extension of the idea. There is limited utility here in such ideas as being able to support a mathematical "pi" or infinity constant using Unicode symbols.

## Rationale

(explain why the (following) proposed solution will solve it)

## Specification

As most of the existing infrastructure is named specifically to do with infix operators, it would be relatively easy to add a new mechanism that can cover more shapes of operator.

The existing components are:

```
PL_infix_plugin

enum Perl_custom_infix_precedence {
    INFIX_PREC_LOW,
    INFIX_PREC_LOGICAL_OR_LOW,
    ...
};

struct Perl_custom_infix {
    OP (*build_op)(pTHX_ SV **opdata, OP *lhs, OP *rhs, struct Perl_custom_infix *);
}
```

They could be renamed to

```
PL_operator_plugin

enum Perl_custom_operator_precedence {
    INFIX_PREC_LOW,
    ...
    PREFIX_LOW,
    ...
    POSTFIX_LOW,
    ...
};

struct Perl_custom_operator {
    OP (*build_op)(pTHX_ SV **opdata, OP *lhs, OP *rhs, struct Perl_custom_operator *);
    /* `rhs` is ignored for non-infix operators */
}
```

There is however a possiblity of a conceptual clash with the other "custom operators" idea, which was added in Perl 5.14 and allows new kinds of opcodes in generated optrees whose behaviour is provided by XS functions. Those are not "operators" in the syntax sense of the word.

Likewise, the `XS::Parse::Infix` module would get renamed to `XS::Parse::Operator`, and allow for expanded API to support these new operator shapes.

## Backwards Compatibility

This proposal only intends to rename a core mechanism used by a special-purpose CPAN module, and does not directly alter any user-visible parts. That CPAN module (`XS::Parse::Infix`) would to be changed to support the new name and behaviour. A wrapper can be provided under its original name to support existing operator modules.

## Security Implications

This proposal adds a minor extension to an existing parser mechanism that is only invoked during compiletime of program source. It is unlikely that any new issues are introduced by such an extension.

## Examples

As this proposal intends to add a core interpreter mechanism for use by a module that is itself simply used by modules that provide syntax, it is hard to directly write any examples of its use. More illustrative may be to consider the kinds of new operator modules that such a mechanism would permit to be written.

It is easily possible to come up with trivial "syntax neatening" of existing operators as mathematical notation, such as squaring and square-roots.

```perl
use Syntax::Operator::Square;     ## provides postfix ²
use Syntax::Operator::SquareRoot; ## provides prefix √

my $hyp = √($x² + $y²);

# equivalent to
my $hyp = sqrt($x**2 + $y**2);
```

For another example, consider the use of a summation operator:

```perl
use Syntax::Operator::Sum;

my @numbers = ...;
my $total = ∑ @numbers;
```

It is admittedly hard to see what this prefix operator notation gains over the far more traditional and widely-supported idea of simply having a `sum` function - such as that provided by `List::Util` - that operates on the array.

I must admit, my imagination is somewhat lacking on some particularly motivating examples. It may be the case that there are genuinely-useful ideas for new operators that could be made that cannot currently be provided in existing syntax, but as yet I have failed to actually think of one.

## Prototype Implementation

None is currently possible as this would need to be supported by the actual Perl parser as implemented in `perly.y`, `toke.c` and related files. If this proposal is accepted as exploratory, a prototype implementation can be directly implemented as a branch of the core perl interpreter.

## Future Scope

* The `XS::Parse::Infix` module, or its more generically-named successor, could be considered for eventual inclusion in core perl directly, making it available to authors without needing to fetch it from CPAN.

* Other more complex operator shapes - circumfix notation around an expression, or more interleaved shapes of syntax and operands. These would need to be considered on a per-usecase basis.

## Rejected Ideas

## Open Issues

The entire thing? ;) As hinted above in the examples section, it isn't immediately clear that this mechanism is even required at all, which is why in my original implementation I restricted it to only infix operators. If there are specific motivating ideas of prefix or postfix operators that cannot be adequately expressed in the existing function-call notation then these would be useful to consider.

## Copyright

Copyright (C) 2024, Paul Evans.

This document and code and documentation within it may be used, redistributed and/or modified under the same terms as Perl itself.
