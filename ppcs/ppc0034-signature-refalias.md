# Ref-aliased parameters in subroutine signatures

## Preamble

    Author:  Gianni Ceccarelli <dakkar@thenautilus.net>
    Sponsor:
    ID:      0034
    Status:  Exploratory

## Abstract

Allow declaring ref-aliased parameters in subroutine signatures.

## Motivation

As of Perl version 5.42, we can do:

    use v5.42;
    use experimental 'declared_refs';

    sub foo {
        my (\%a) = @_;
        ...
    }

but we cannot do:

    sub foo(\%a) {
        ...
    }

## Rationale

Subroutine signatures work very similarly to the old-style `my … =
@_;` line, including the slurpy parameter at the end. But, as noted
above, they don't allow declaring a ref-aliased parameter. This feels
inconsistent.

## Specification

In addition to the form `$things`, a scalar parameter may also be
declared as ref-alias (requires the `refaliasing` feature and the
`declared_refs` feature, see "Assigning to References" and "Declaring
a Reference to a Variable" in perlref for more details):

    sub go_over(\@things) { say "Oooh, $_!" for @things }

This subroutine must still be called with a scalar value, but the
value must now be a reference to an array. Equivalently:

    sub look_at(\%these) {
        say "$_ means $these{$_}" for sort keys %these;
    }

must be called with a reference to a hash, and:

    sub normalise(\$string) {
        $string = lc($string);
    }

must be called with a reference to a scalar. As with normal scalar
parameters, ref-aliased parameters can be ignored:

    sub ignore(\@) { ... }

and have default values:

    sub walk($cb, \@nodes, \%seen ||= {}) {
        for my $node (@nodes) {
            next if $seen{$node->id}++;
            $cb->($node);
            walk($cb, $node->children, \%seen);
        }
    }

The last example is equivalent to:

    sub walk($cb, $nodes, $seen ||= {}) {
        for my $node ($nodes->@*) {
            next if $seen->{$node->id}++;
            $cb->($node);
            walk($cb, $node->children, $seen);
        }
    }

with different legibility trade-offs.

Notice that arguments are still passed by reference, so any
modification to their contents will be seen by the caller. In other
words:

    sub my_push(\@items, $new_one) {
        push @items, $new_one;
    }

works exactly the same way as:

    sub my_push($items, $new_one) {
        push $items->@*, $new_one;
    }

or

    sub my_push {
        my (\@items, $new_one) = @_;
        push @items, $new_one;
    }

## Backwards Compatibility

The proposed syntax is currently not valid, so it can not generate
confusion for existing code.

`B::Deparse` will need to be made aware of it.

`PPI` can parse it just fine:

    sub foo(\@a=[])

produces:

    PPI::Statement::Sub
      PPI::Token::Word      'sub'
      PPI::Token::Whitespace      ' '
      PPI::Token::Word      'foo'
      PPI::Structure::Signature      ( ... )
        PPI::Statement::Expression
          PPI::Token::Cast      '\'
          PPI::Token::Symbol      '@a'
          PPI::Token::Operator      '='
          PPI::Structure::Constructor      [ ... ]

`Perl::Critic` seems to have no problems with it: given the above
declaration, it complains that «Magic variable "@a" should be assigned
as "local"»; it does the same if I write `sub foo($a=[])`.

## Security Implications

None foreseen.

## Examples

`LWP::Protocol::http::hlist_remove` looks like this:

    sub hlist_remove {
        my($hlist, $k) = @_;
        $k = lc $k;
        for (my $i = @$hlist - 2; $i >= 0; $i -= 2) {
	        next unless lc($hlist->[$i]) eq $k;
	        splice(@$hlist, $i, 2);
        }
    }

It could become:

    sub hlist_remove(\@hlist, $k) {
        $k = lc $k;
        for (my $i = @hlist - 2; $i >= 0; $i -= 2) {
	        next unless lc($hlist[$i]) eq $k;
	        splice(@hlist, $i, 2);
        }
    }

## Prototype Implementation

Paul Evans implemented most of this already:
https://metacpan.org/pod/Sublike::Extended#Refalias-Parameters
only missing the default value assignments.

## Future Scope

Perl 5.44 will support named parameters in signatures. It would make
sense to also allow ref-aliased named parameters; the obvious syntax
would be `:\%foo`, which may look too much "punctuation soup" (thanks
Paul for that expression).

## Rejected Ideas

Paul Evans suggested using an attribute instead:

    sub foo (%a :refalias)

which would extend to named parameters:

    sub foo (:%a :refalias)

and be similar to a proposed "aliased scalars" feature:

    sub foo ($a :alias) { $a=1 }
    # equivalent to:
    sub foo { $_[0]=1 }

One of the problems with the attribute-based approach is that `sub
foo(%a)` and `sub foo(%a :refalias)` have very different signatures
(the first one takes an even-sized list, the second one takes a single
scalar), but they look very similar.

Everywhere else in Perl, `%a` and `@a` imply multiple values / list
context, but `\%a` and `\@a` are scalars / induce scalar context.  I
believe we should not break this pattern.

## Open Issues

How do we handle ref-aliased named parameters?

## Copyright

Copyright (C) 2025, Gianni Ceccarelli.

This document and code and documentation within it may be used,
redistributed and/or modified under the same terms as Perl itself.
