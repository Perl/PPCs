# Optional Chaining

## Preamble

    Author:  Breno G. de Oliveira <garu@cpan.org>
    Sponsor:
    ID:      0021
    Status:  Draft

## Abstract

This PPC proposes a new operator, `?->`, to indicate optional dereference
chains that short-circuit to an empty list when the left side is undefined.

## Motivation

Chained dereferencing of nested data structures and objects is quite common
in Perl programs, and developers often find themselves needing to check
whether the data is there in the first place before using the arrow notation,
otherwise the call may trigger a runtime error or modify the original data
structure due to unwanted autovivification.

The current syntax for these verifications can be quite long, hard to write
and read, and prone to human error - specially in chained methods, as you
need to be careful not to call the same method twice.

So the idea is to be able to replace this:

```perl
    my $val;
    if (   defined $data
        && defined $data->{deeply}
        && defined $data->{deeply}{nested}
        && defined $data->{deeply}{nested}[0]
        && defined $data->{deeply}{nested}[0]{data}
    ) {
        $val = $data->{deeply}{nested}[0]{data}{value}
    }
```

With this:

```perl
    my $val = $data?->{deeply}?->{nested}?->[0]?->{data}?->{value};
```

And be able to replace this:

```perl
    my $val;
    if (defined $obj) {
        my $tmp1 = $obj->this;
        if (defined $tmp1) {
            my $tmp2 = $tmp1->then;
            if (defined $tmp2) {
                $val = $tmp2->that;
            }
        }
    }
```

With this:

```perl
    my $val = $obj?->this?->then?->that;
```

## Rationale

An "optional chaining" (sometimes referred to as "null safe", "safe call",
"safe navigation" or "optional path") operator would let developers access
values located deep within a chain of connected references and fail
gracefully without having to check that each of them is defined.

This should result in shorter, simpler and more correct expressions whenever
the path being accessed may be missing, such as when exploring objects and
data structures without complete guarantees of which branches/methods are
provided.

Similar solutions have been thoroughly validated by many other popular
programming languages like JavaScript, Kotlin, C#, Swift, TypeScript, Groovy,
PHP, Raku, Ruby and Rust, in some cases for over 15 years now [1].

## Specification

The `?->` operator would behave exactly like the current dereference arrow
`->`, interacting with the exact same things and with the same precedence,
being completely interchangeable with it. The only difference would be that,
whenever the lefthand side of the operator is undefined, it short-circuits
the whole expression to an empty list `()` (which becomes `undef` in scalar
context).

One could say that:

    EXPR1 ?-> EXPR2

is equivalent to:

    defined EXPR1 ? EXPR1->EXPR2 : ()

with the important caveat that EXPR1 is only evaluated once.

## Backwards Compatibility

All code with `?->` currently yields a compile time syntax error, so there
are no expected conflicts with any other syntax in Perl 5.

Notable exceptions are string interpolation (where `"$foo->bar"` already
ignores the arrow and `"$foo?->{bar}"` resolves to showing the reference
address followed by a literal `?->{bar}`) and regular expressions (where
`?` acts as a special character).  Optional chains should be disallowed in
those scenarios (but see 'Open Issues' below).

Static tooling may confuse the new operator with a syntax error until
updated.

Because it is an infix operator with special/reserved characters and short-
circuiting, this feature cannot be easily emulated as-is with a CPAN module.
XS::Parse::Infix could help, but even that only works on a patched version of
perl.

## Security Implications

None foreseen.

## Examples

Below are a few use case examples and explored edge-cases, with their current
Perl 5 equivalent in the comments.

Expected common uses:

```perl
    # $val = defined $foo && defined $foo->{bar} ? $foo->{bar}[3] : ();
    $val = $foo?->{bar}?->[3];

    # $tmp = defined $obj ? $obj->m1 : (); $val = defined $tmp ? $tmp->m2 : ();
    $val = $obj?->m1?->m2;

    # $val = defined $obj ? $obj->$method : ();
    $val = $obj?->$method;

    # $ret = defined $coderef ? $coderef->(@args) : ();
    $ret = $coderef?->(@args);

    # $foo->{bar}{baz} = 42 if defined $foo && defined $foo->{bar};
    $foo?->{bar}?->{baz} = 42;

    # %ret = defined $href ? $href->%* : ();
    %ret = $href?->%*;

    # $n = defined $aref ? $aref->$#* : ();
    $n = $aref?->$#*;

    # foreach my $val (defined $aref ? $aref->@* : ()) { ... }
    foreach my $val ($aref?->@*) { ... }

    # @vals = defined $aref ? $aref->@* : ();
    @vals = $aref?->@*;  # note that @vals is (), not (undef).

    # @vals = defined $aref ? $aref->@[ 3...10 ] : ();
    @vals = $aref?->@[ 3..10 ]

    # @vals = map $_?->{foo}, grep defined $_, @aoh;
    @vals = map $_?->{foo}, @aoh;

    # \$foo->{bar} if defined $foo;
    \$foo?->{bar};  # as with regular arrow, becomes \($foo?->{bar})

    # my $class = 'SomeClass'; $class->new if defined $class;
    my $class = 'SomeClass'; $class?->new;

    # my $obj = %SomeClass:: ? SomeClass->new : ();
    my $obj = SomeClass?->new;  # TBD: see 'Future Scope' below.

    # my @objs = (%NotValid:: ? NotValid->new : (), %Valid:: ? Valid->new : ());
    my @objs = ( NotValid?->new, Valid?->new ); # @objs == ( ValidObject )
```

Unusual and edge cases, for comprehension:

```perl
    # $y = ();
    # if (defined $x) {
    #   my $tmp = $i++;
    #   if (defined $x->{$tmp}) {
    #     $y = $x->{$tmp}->[++$i]
    #   }
    # }
    $y = $x?->{$i++}?->[++$i];

    # $tmp = ++$foo; $val = defined $tmp ? $tmp->{bar} : ();
    $val = ++$foo?->{bar};  # note that this statement makes no sense.

    # my $val = $scalar_ref->$* if defined $scalar_ref;
    my $val = $scalar_ref?->$*;

    # $ret = defined $coderef ? $coderef->&* : ();
    $ret = $coderef?->&*;

    # $glob = $globref->** if defined $globref;
    $glob = $globref?->**;
```

## Prototype Implementation

None.

## Future Scope

Because the idea is to be completely interchangeable with the arrow notation,
it would be important to cover class methods, where an arrow is used but
the check cannot be 'defined' because there is no concept of definedness
on barewords.

```perl
    my $obj = SomeModule?->new;
```

The equivalence, in this case, would be:

```perl
    my $obj = %SomeModule:: ? SomeModule->new : ();
```

While this is the actual goal, a first version of the operator could ignore
this and stick with named variables.

Also, optional chains under string interpolation and regular expressions
could be enabled in the future, hidden behind a feature flag to prevent
backwards compatibility issues, much like what was done with `postderef_qq`.
However, due to the nature of the operator, the need for its usage on those
contexts should not be too big - and even then it can be mitigated by efforts
such as Template Literals (PPC0019).

## Rejected Ideas

The idea of a similar operator has been going on and off the p5p list since
at least 2010 [2] in various shapes and forms. While generally very well
received, discussion quickly ended in either feature creep or bikeshedding
over which symbols to use. Below, I will try to address each question that
arose in the past, and the design decisions that led to this particular PPC.

* Why not just wrap everything in an eval or try block?

Besides being shorter, the optional chain will NOT silence actual errors
coming from a method, or from a defined value that is not a ref.

* Why not just a lexical pragma?

Providing the definedness check on arrow operators via a pragma is
unfortunately not good enough as it (a) doesn't give you enough granularity
and control of your chain in, say, `$o?->m1->m2?->m3;`; (b) promotes
[action at a distance](https://en.wikipedia.org/wiki/Action_at_a_distance_(computer_programming));
and (c) even if only enabled lexically, provides little to no gain in terms
of reading/writing statements as `{no X; $o->m}` vs `$o->m if defined $o`.

* Why is the token `?->` and not "X"

There have been a lot of different proposals for this operator over the
years, and even a community poll [3] to decide which one to use. While `~>`
won, it can be hard to distingish from `->`. Also, `~` in perl is already
associated with negation, regexes, and the infamous smartmatch operator.
Likewise, the `&` character is used in many different contexts (bitwise,
logic, prototype bypassing, subroutines) and adding another one seemed
unnecessary.

`?->` was the runner up in the poll, and its popularity is justified: it
alludes to a ternary check and, even in regexes, to only proceeding if
whatever came before is there. It also leaves the arrow intact, indicating
it is checking something that comes _before_ the dereferencing takes place
(unlike, for example, `->?` or `->>`). It is also the chosen notation for
most languages that implement this feature, so why surprise developers with
another way to express what they are already familiar with?

Finally, `//->` was considered but `//` is defined-or, not defined-and as
this new operator, so it could be even more confusing to developers.

* Why add this just to arrow dereferencing and not to "X"

Because the optional chain is trying to solve the specific (and real-world)
issue of having to add a lot of repetitive and error-prone boilerplate tests
on an entire chain individually, and nothing else.

While we could (hypothetically) try to expand this notion to other operators
(e.g. `?=`, `?=~`, etc) or even explore related ideas (such as a non-null
assertion operator like `!->`), the benefit of doing so is unclear at this
point, and would require a lot of effort picking appart which operators
should and shouldn't include the "optional" variation [4].

* Why definedness and not truthfulness? Or 'ref'? Or 'isa'?

Semantically, `undef` means there is nothing there. We still want the code
to fail loudly if we are dereferencing anything defined, as it would indicate
something wrong with the code, the underlying logic, or both. It is also
important to note that Perl allows you to call methods on strings (then
treated as package/class names), so we cannot reliably test for 'ref'
without making things really convoluted with assumptions, rules and
exceptions for each type of data.

* Can we have implicit `?->` after its first use? Or have it flip/flop
against `->` on a chain?

The proposed optional chain short-circuit is not a bugfix of the arrow
dereference's behaviour, it's a shortcut to a common (and error prone)
construct that happens when the developer wants to achieve just that. Short-
circuiting instead of runtime exceptions could hide bugs, so we require
developers to make a conscious and explicit choice every time they want this
feature in any part of the chain. It may be a little bit tedious to
write longer optional chains, but it will surely be easier to read,
understand and spot logic errors if the operator is explicit.

This decision is endorsed by the fact that all implementations of the
null-safe / optional chain operator in other languages also require it to be
explicit. Besides, evidence suggests less than 20% of the usage will chain
more than two calls[5].

A possible alternative to make the optional chain "greedy" could be the
addition of yet another operator (like `??->`), but they are beyond the scope
of this PPC and should be explored in another draft, if the need arises.

* Why not other identity values according to context? Like '' when
concatenating strings, 0 on addition/subtraction and 1 on
multiplication/division?

While tempting, it would not only produce values that could be confused with
an actual successful call, it would also mean we'd have to check all types
of data and agree on their identity value (what is the identity of a sub?
Or a glob?).

Instead, one could just use the already provided `//` operator to achieve
the same results, e.g.: `"value is: " . ($ref?->{val} // '')`.

## Open Issues

None.

## References:

1. https://github.com/apache/groovy/commit/f223c9b3322fef890c6db261720f703394c7cf27
2. https://www.nntp.perl.org/group/perl.perl5.porters/2010/11/msg165931.html
3. https://www.perlmonks.org/?node_id=973015
4. Python's proposal, and the ongoing discussion around it, are a noteworthy
   external example of the consequences of trying to add that logic to many
   different operators. https://peps.python.org/pep-0505
5. https://github.com/alangpierce/coffeescript-soak-stats

## Copyright

Copyright (C) 2022, Breno G. de Oliveira.

This document and code and documentation within it may be used, redistributed
and/or modified under the same terms as Perl itself.
