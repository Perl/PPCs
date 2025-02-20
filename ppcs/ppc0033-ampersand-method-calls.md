# Allow calling subs using method-like syntax

## Preamble

    Author:  Graham Knop <haarg@haarg.org>
    ID:      0033
    Status:  Implemented

## Abstract

Add a syntax for calling lexical methods that is similar to the existing
method syntax, rather than needing to call them like subs. The new syntax
`$object->&method(@args)` would be equivalent to sub call to `method`, but
using a syntax closer to method calls.

## Motivation

When using a lexical sub inside an object class, the subs must be called as
`method($object, @args)`. This works, but for many people, this feels wrong
as a way to call something thought of as a private method. This has resulted
in some people using syntax like `$object->${\&method}(@args)`, or continuing
to use code refs stored in a lexical variable (`$object->$method(@args)`).

## Rationale

`$object->method(@args)` always does a method lookup, ignoring the local
context. Changing this would result in a lot of broken code. Changing this
only for new code (via feature or other lexical effect) would introduce
confusion and would also require some new (or pessimised) syntax to allow
the previous behavior.

`$object->&method(@args)` would be an unambiguous way to call the `method` in
the current scope, whether a lexical or package sub.

A truly private method does not need to do any lookup via `@ISA`, so being
equivalent to a sub call makes sense.

This also would pair well with
[PPC0021](ppc0021-optional-chaining-operator.md) to allow optional sub calls
as part of a chain.

## Specification

`$object->&method(@args)` would behave exactly the same as `&method($object,
@args)`. `method` would be searched for in the current lexical scope. If not
found, it would be looked up in the current package. If still not found, the
current package's `AUTOLOAD` would be called. If `AUTOLOAD` does not exist, an
error would be issued. The call would ignore prototypes, just as traditional
method calls and `&subroutine()` calls do.

Methods defined using the `class` syntax would also be callable using this
syntax.

## Backwards Compatibility

The syntax `->&word` is currently a syntax error, so no backwards
compatibility issues are forseen.

## Security Implications

As this is purely a syntax feature that is currently a syntax error, no
security implications are forseen.

## Examples

```perl
use v5.36;

package Foo {
    sub new ($class) {
        return bless {
            field => 5,
        }, $class;
    }

    my sub private ($self) {
        return $self->{field};
    }

    sub _old_school_private ($self) {
        return $self->{field};
    }

    sub public ($self) {

        # call via lexical method is allowed
        say "my field: " . $self->&private;
        # exactly equivalent to:
        say "my field: " . private($self);


        # can also be used to call package methods. But won't look up via @ISA
        say "my field: " . $self->&_old_school_private;
        # exactly equivalent to:
        say "my field: " . _old_school_private($self);
    }
}

my $foo = Foo->new;
$foo->public;

# this is an error, as there is no sub named "private" in scope
$foo->&private;

# this is also an error, as "public" does not exist in the current scope
$foo->&public;
```

## Prototype Implementation

* [Object::Pad::LexicalMethods](https://metacpan.org/pod/Object::Pad::LexicalMethods)
  implements lexical methods and `->&` method calls for
  [Object::Pad](https://metacpan.org/pod/Object::Pad).

## Future Scope

Lexical methods (`my method foo`) are not currently supported under the
`class` syntax. If they were added, `->&method` would be the preferred way to
call them.

## Rejected Ideas

Allowing `$object->method` to call lexical subs would break existing code and
would interfere with the ability to call the actual method. Perl doesn't know
what type `$object` is, so it can't know when it is appropriate to do method
lookup and when to call a local sub. Using a different syntax at the call site
avoids these issues.

## Open Issues

None currently.

## Copyright

Copyright (C) 2025, Graham Knop

This document and code and documentation within it may be used, redistributed
and/or modified under the same terms as Perl itself.
