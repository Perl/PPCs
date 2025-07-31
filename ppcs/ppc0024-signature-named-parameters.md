# Named Parameters in Signatures

## Preamble

    Author:  Paul Evans <PEVANS>
    Sponsor:
    ID:      0024
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

Since it is a relatively common pattern in callsites in existing code to rely on the semantics of assignment of name-value pair lists into hashes, the behaviour on encountering duplicate key names needs to be preserved. This is that duplicated key names do not raise an error or a warning, and simply accept the last value associated with that name. This allows callers to collect values from multiple sources with different orders of priority to override them; for example using a hash of values combined with individual elements:

```perl
sub func ( :$abc, :$xyz, ... ) { ... }

func(
    abc => 123,
    %args,
    xyz => 789,
);
```

In this example, the given `abc` value will take effect unless overridden by a later value in `%args`, but the given `xyz` value will replace an earlier one given in `%args`. Neither situation will result in a warning or error.

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

Since defaulting expressions are full expressions and not necessarily simple constant values, the time at which they are evaluated is significant. Much like with positional parameters, each is evaluated in order that it is written in source, left to right, and each can make use of values assigned by earlier expressions. This means they are evaluated in the order that is written in the function's declaration, which may not match the order that values were passed from the caller in the arguments list.

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

One example comes from [`IPC::MicroSocket::Server`](https://metacpan.org/pod/IPC::MicroSocket::Server). This is currently using the `Sublike::Extended` module (see "Prototype Implementation" below) to provide the syntax. The actual module also uses `Object::Pad` to provide the `method` keyword; this example is paraphrased to avoid that. This module requires a `path` named argument, and optionally takes a `listen` argument, defaulting its value to 5 if the caller did not provide a defined value.

```perl
extended sub new_unix ( $class, :$path, :$listen //= 5 )
{
    ...
}
```

This replaces the previous version of the code which handled arguments by the more traditional approach of assigning into a hash:

```perl
sub new_unix ( $class, %args )
{
    my $path   = $args{path};
    my $listen = $args{listen} // 5;
    ...
}
```

Already the new code is shorter and more consise. Additionally it contains error checking that complains about missing mandatory keys, or unrecognised keys, which the previous version of the code did not include.

Another example, this time from [`Text::Treesitter::QueryCursor`](https://metacpan.org/pod/Text::Treesitter::QueryCursor). This method takes an optional argument named `multi`, tested for truth. If absent it should default to false.

```perl
sub next_match_captures ( $self, :$multi = 0 )
{
    ...
}
```

The previous version of this code did include complaints about unrecognised keys, and was rather longer because of it:

```perl
sub next_match_captures ( $self, %options )
{
    my $multi = delete $options{multi};
    keys %options and
        croak "Unrecognised options to ->next_captures: " . join( ", ", keys %options );

    ...
}
```

## Prototype Implementation

The [`XS::Parse::Sublike`](https://metacpan.org/pod/XS::Parse::Sublike) module contains parsing to allow third-party syntax modules to parse subroutine-like constructions, and includes a parser for named parameters already having this syntax. These are also made available to regular subroutines via the `extended` keyword provided by [`Sublike::Extended`](https://metacpan.org/pod/Sublike::Extended).

Additionally, other existing CPAN modules already parse syntax in this, or a very similar format:

* [`Function::Parameters`](https://metacpan.org/pod/Function::Parameters)

* [`Kavorka`](https://metacpan.org/dist/Kavorka/view/lib/Kavorka/Manual/Signatures.pod)

* [`Method::Signatures`](https://metacpan.org/pod/Method::Signatures)

All of these use the leading-colon syntax in a signature declaration to provide named parameters in the same style as this proposal. It would appear we are in good company here.

## Future Scope

* If the Metaprogramming API (PPC0022) gains introspection abilities to enquire about subroutine signature parameters, further consideration will need to be made in that API on how to represent the extra kinds of parameters added by this specification.

## Rejected Ideas

* This specification intentionally makes no changes to the call-site of any subroutines using named parameters.

## Open Issues

### Parameter Names

How to specify a named parameter whose name is anything other than the name of the lexical variable into which its value is assigned? This is more of note when considering that traditional argument handling techniques involving assignment into hashes can handle more styles of name that would be invalid for lexical variables - for example, names including hyphens.

Perhaps a solution would be to permit attributes on parameter variables, then define a `:name` attribute for this purpose.

```perl
sub display ( $message, :$no_colour :name(no-colour) ) {
    ...
}
```

Further thoughts in that direction suggested that this could also support multiple names with aliases:

```perl
sub display ( $message, :$no_colour :name(no-colour no-color) ) {
    ...
}
```

The attribute syntax starting with a leading colon does visiually look quite similar to the named parameter syntax which also uses. This is a little unfortunate, but perhaps an inevitable consequence of the limited set of characters available from ASCII. As attributes are already well-established as leading with a colon, the only other option would be to pick a different character for named attributes; but this would be contrary to the established convention of the existing modules listed above.

Perl does not currently support attributes being applied to parameters in signatures, named or otherwise, but this is the subject of a future PPC document I am currently drafting. I will add a reference here when it is published.

## Copyright

Copyright (C) 2024, Paul Evans.

This document and code and documentation within it may be used, redistributed and/or modified under the same terms as Perl itself.
