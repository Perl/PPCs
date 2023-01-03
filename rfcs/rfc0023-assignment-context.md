# Assignment context attributes

## Preamble

    Author:  Curtis "Ovid" Poe <curtis.poe@gmail.com>
    Sponsor:
    ID:      0023
    Status:  Draft

## Abstract

Add `:scalar`, `:list`, and `:void` attributes for subroutines/methods
to ensure their results are handled correctly rather than relying on the
`wantarray` heuristic.

## Motivation

* Create attributes for subroutines which require their return values to be
  immediately assigned or ignored.
* Not handling the context correctly will be a fatal error
* Reduce dependency on fragile `wantarray` heuristics.
* Reduce boilerplate code being hand-written over and over. 

## Rationale

It's very easy to write `wantarray` code that doesn't really do what you
want. For example, I might have code which returns data which must be handled,
so I use `wantarray` to guarantee that it's not called in void context:

    sub munginator {
        if ( !defined wantarray ) {
            die "Do not call me in void context";
        }
        ...
        return $some_scalar;
    }

The intent is that this should `die`:

    my $value = get_value();
    munginator($value); # fatal error!

And this should work:

    my $value = get_value();
    $value = munginator($value);

But here we have an edge case:

    if ( munginator($value) ) {
        ...
    }

That's evaluated in scalar context, so our `wantarray` heuristic has failed.
We would like to not only stop reliance on a dodgy heuristic, but to not have
to keep writing extra code to handle this case.

Instead, it would be nice to write the following:

    sub munginator :scalar {
        ...
        return $some_scalar;
    }

Any code calling `munginator` must do so by assigning the return value to a
scalar. Otherwise the `if ( munginator($value) ) {...}` code fails as
intended.

## Specification

Three new attributes are added, enabled via a feature named
`assignment_context` (a better name would be good).

1. `:scalar` - Return value must be assigned to a scalar, in scalar context
2. `:list` - Return value must be assigned to a list
3. `:void` - Return value must not be assigned to anything

These attributes can be used on named subroutines/methods:

    sub foo :list {...}

Or anonymous subroutines:

    my $foo = sub :void {...};

Or a block `eval` (this is for completeness and may be be needed):

    my $result = eval :scalar {...};

The assignment must be in the statement calling the subroutine and cannot
propagate. Thus, this will not work:

    my $result = foo();
    sub munginator :scalar {...}
    sub foo {
        return munginator(42);
    }

The above would need to be written similarly to this:

    my $result = foo();
    sub munginator :scalar {...}
    sub foo {
        return my $result = munginator(42);
    }

## Backwards Compatibility

I am not aware of any core code which relies on these attributes. It's
entirely possible (even likely) that there are CPAN or DarkPAN modules which
do.

## Security Implications

None that I'm aware of.

## Prototype Implementation

None.

## Open Questions

What should this do?

    my ($value) = some_sub();

If `some_sub()` is marked as a `:list`, the above is correct, but if
`some_sub()` returns more than one element, the behavior may not match
developer expectations.

If `some_sub()` is  marked as `:scalar`, the above should still
work, but the assignment is in list context, not scalar context.

# Exaamples

    # munges arguments directly and has no guaranteed return value,
    # so calling this is void context is required
    sub lc_values :void ($hashref) {
        $_ = lc $_ for values %hashref->%*;
    }

    # Returns copy of the hash
    # so calling this is list context is required
    sub lc_values :list (%hash) {
        my %new_hash = map { $_ => lc $hash{$_} } keys %hash;
        return %new_hash;
    }

    # Returns copy of the hashref
    # so calling this is scalar context is required
    sub lc_values :list ($hashref) {
        return { map { $_ => lc $hashref->{$_} } keys $hashref->%* }
    }

## Prototype Implementation

At the present time, creating a prototype implementation would likely require
XS code and heavy knowledge of the Perl internals. I cannot do this.

## Future Scope

It is possible that we might allow propagation of assignment context if the
calling subroutine has an `identical` assignment context attribute.

    my $result = foo();
    sub munginator :scalar {...}
    sub foo :scalar {
        return munginator(42);
    }

But for now, I wanted to keep this simple.

## Copyright

Copyright (C) 2023, Curtis "Ovid" Poe

This document and code and documentation within it may be used,
redistributed and/or modified under the same terms as Perl itself.

