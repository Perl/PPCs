# Named Parameters in Signatures

## Preamble

    Author:  Paul Evans <PEVANS>
    Sponsor:
    ID:      TODO
    Status:  Exploratory

## Abstract

Adds the ability for subroutine signatures to process named arguments in the form of name-value pairs passed by the caller, in a fashion familiar to existing uses of assignment into a hash.

## Motivation

Perl 5.20 added "subroutine signatures", native syntax for more succinctly expressing the common patterns of processing arguments inbound into a subroutine by unpacking the `@_` array. Rather than writing common styles of code directly, a more compact notation, in a style immediately familiar to users of many other languages, allows Perl to create the required behaviour from that specification.

```perl
sub f ($x, $y, $z = 123) { ... }

# instead of
sub f {
    die "Too many arguments" if @_ > 3;
    die "Too few arguments" if @_ < 2;
    my ($x, $y, $z) = @_;
    $z = 123 if @_ < 3;
    ...
}
```

One common form of argument processing involves passing an even-sized list of key/value pairs and assigning that into a hash within the subroutine, so that specifically named parameters can be extracted from it. There is currently no support from subroutine signature syntax to assist authors in providing such behaviours.

## Rationale

(explain why the (following) proposed solution will solve it)

## Specification

A new kind of element may be present in a subroutine signature, which consumes a named argument from the caller. These elements are written with a leading colon prefix (`:$name`), indicating that it is named rather than positional. The name of each parameter is implied by the name of the lexical into which it is assigned, minus the leading `$` sigil.

Each element provides a new lexical variable that is visible during the body of the function, in the same manner as positional ones.

The value of a named parameter is taken from the argument values passed by the caller, in a manner familiar to existing uses of hash assignment. The caller should pass an even-sized name-value pair list. The values corresponding to names of parameters will be assigned into the variables. The order in which the values are passed by the caller is not significant.

Furthemore, all of the new behaviour is performed within the body of the invoked subroutine entirely by inspecting the values of the arguments that were passed. The subroutine is not made aware of how those values came to be passed in - whether from literal name-value syntax, a hash or array variable expansion, or any other expression yielding such a list of argument name and value pairs.

```perl
sub make_colour ( :$red, :$green, :$blue ) { ... }

make_colour( red => 1.0, blue => 0.5, green => 0.2 );
# The body of the function will be invoked with
#   $red   = 1.0
#   $green = 0.2
#   $blue  = 0.5
```

As with positional parameters, a named parameter without a defaulting expression is mandatory, and an error will be raised as an exception if the caller fails to pass a corresponding value. A defaulting expression may be specified using any of the operators available to positional parameters - `=`, `//=` or `||=`.

```perl
sub make_colour ( :$red = 0, :$green = 0, :$blue = 0 ) { ... }

make_colour( red => 1.0, blue => 0.5 );
# The body of the function will be invoked with
#   $red   = 1.0
#   $green = 0
#   $blue  = 0.5
```

A subroutine is permitted to use a combination of *mandatory* positional and named parameters in its definition, provided that all named parameters appear after all the positional ones. Any named parameters may be optional; there are no ordering constraints here.

If a subroutine uses named parameters then it may optionally use a slurpy hash argument as its final element. In this case, the hash will receive all *other* name-value pairs passed by the caller, apart from those consumed by named parameters. If the subroutine does not use a slurpy argument, then it will be an error raised as an exception for there to be any remaining name-value pairs after processing.

While all of the above has been specified in terms of subroutines, every point should also apply equally to the methods provided by the `class` feature, after first processing the implied invocant `$self` parameter.

## Backwards Compatibility

At the site of a subroutine's definition, this specification only uses new syntax in the form of the leading colon prefix on parameter names. Such syntax was formerly invalid in previous versions of perl. Thus there is no danger of previously-valid code being misinterpreted.

All of the behaviour provided by this new syntax is compatible with and analogous to any existing code that could have been written prior, perhaps by direct assignment into a hash. There are no visible differences in the external interface to a subroutine using such syntax, and it remains callable in exactly the same way. Functions provided by modules could be upgraded to use the new syntax without any impact on existing code that invokes them.

## Security Implications

There are no anticipated security concerns with expanding the way that subroutines process parameters in this fashion.

## Examples

As the intention of this syntax addition is to make existing code practice more consise and simple to write, it would be illustrative to compare pairs of functions written in the newly proposed vs. the existing style.

TODO

## Prototype Implementation

The [`XS::Parse::Sublike`](https://metacpan.org/pod/XS::Parse::Sublike) module contains parsing to allow third-party syntax modules to parse subroutine-like constructions, and includes a parser for named parameters already having this syntax. These are also made available to regular subroutines via the `extended` keyword provided by [`Sublike::Extended`](https://metacpan.org/pod/Sublike::Extended).

## Future Scope

* If the Metaprogramming API (PPC0022) gains introspection abilities to enquire about subroutine signature parameters, further consideration will need to be made in that API on how to represent the extra kinds of parameters added by this specification.

## Rejected Ideas

* This specification intentionally makes no changes to the call-site of any subroutines using named parameters.

## Open Issues

* How to specify a named parameter whose name is anything other than the name of the lexical variable into which its value is assigned? This is more of note when considering that traditional argument handling techniques involving assignment into hashes can handle more styles of name that would be invalid for lexical variables - for example, names including hyphens.

## Copyright

Copyright (C) 2024, Paul Evans.

This document and code and documentation within it may be used, redistributed and/or modified under the same terms as Perl itself.
