# String Literals with Expression Interpolation

## Preamble

    Author:  Ricardo Signes <rjbs@semiotic.systems>
    Sponsor: RJBS
    ID:      RFC 0019
    Status:  Draft

## Abstract

This document proposes a new quote-like operator for string literals.  This
operator, `qt`, is meant to serve as an alternative to `qq`, with simpler rules
for understanding what is interpolated and how.  It takes its design from
JavaScript's template literals and Ruby's string interpolation.

Instead of allowing a subset of variable expressions directly within a string,
qt allows *any* expression to be interpolated when delimited within the string
literal.

## Motivation

Existing interpolating strings are extremely convenient, but have shortcomings
built into their design.  Examples:

```perl
my $what   = "balloons";
my $colors = [ qw( red white blue ) ];

my $object   = Party->new({ type => "birthday", bring => $what });

$s = "$problems";     # "" . $problems
$s = "$colors->[0]";  # "" . $colors->[0]

$s = "@$colors";      # join($", @$colors)
$s = "$colors->@*";   # If postderef_qq on:  join($", $colors->@*)
                      # Otherwise         :  "" . $colors . "->@*"

$s = "Bring $object->{bring}";  # "Bring " . $object->{bring}
$s = "Bring $object->bring";    # "Bring " . $object . "->bring"
```

As we look into adding new forms of dereference (for example the proposed `?->`
operator), or at figuring out how to make `postderef_qq` a default behavior, we
are likely to keep hitting confusing problems with the behavior of `qq`.  We
should add something that designs away all these problems now and for the
future.

## Rationale

The proposed `qt` operator only looks for one special token in a string
literal: `{`.  Source content until the matching `}` is treated as a scalar
expression to be interpolated into the string in place of the `{...}`
construct.  This is arbitrarily extensible for other existing expressions and
for new ones that may be added.

To include a verbatim `{` in the string value, escape it:  `\{`

## Specification

This:

```
qt{A { TEXT } B};
```

will be equivalent to

```
qq{A ${ \scalar(TEXT) } B};
```

To provide a heredoc version of qt, this:

```
<<qt{END}
TEXT
END
```

will apply qt-like interpolation to TEXT.  This (very rarely-seen) peril will
exist in qt heredocs as it does in qq heredocs:

```
$str = <<"END";
A ${
END
}
END
```

(The peril here is that the first `END` terminates the string, meaning that the
`${` is never closed.)

## Backwards Compatibility

Existing static analysis should be able to understand `qt` as a new quote-like
operator fairly easily.  If the analyzer parses `${...}` within a `qq` string,
parsing the `{...}` forms in a `qt` string should be possible.

`qt` will need to be made available by a feature guard.  Use of the qt operator
in source without the feature enabled would often parse as a subroutine call.

There is prior art on the CPAN, most specifically in
[Quote::Code](https://metacpan.org/pod/Quote::Code), which provides a `qc`.
The syntax provided by Quote::Code is similar to, but not identical with, that
proposed here.

## Security Implications

None identified at time of writing.

## Examples

The **Motivation** section, above, provides examples of what *doesn't* work
well with existing interpolation.  Documentation can start with simple
interpolation in both, then show how to use qt strings to allow more forms of
interpolation.

```
# No interpolation
qt{Greetings};

# Simple scalar interpolation
qt<Greetings, {$title} {$name}>;

# Interpolation of method calls
qt"Greetings, {$user->title} {$user->name}";

# Interpolation of various expressions
qt{It has been {$since{n}} {$since{units}} since your last login};

qt{...a game of {join q{$"}, $favorites->{game}->name_words->@*}};
```

## Prototype Implementation

Quote::Code, mentioned above, provides something very similar.

## Future Scope

`qt` is "qq but different".  If successful, it might suggest the usefulness of
qt-like forms of qr, m, s, and other string-interpolating contexts.  This also
suggests that another possible design for qt would be a pragma to change the
behavior of qq *and other interpolating quote-like operators*.  This is more
complex for the reader (because any given piece of code must be considered in
terms of the enabled features), but eliminates the proliferation of QLOPs.

## Rejected Ideas

This proposal uses `{...}` instead of `${...}` because `${...}` expects a
*reference* inside in all other contexts, which is not the case here.

This proposal uses `{...}` (like JavaScript) instead of `#{...}` (like Ruby)
because the author didn't think the extra character was likely valuable enough
for disambiguation between literal `{` and start of interpolated expression.

This proposal does not offer a means to interpolate a list without (say)
`join`.  It would require more syntax and leave the user falling back on
remembering that multiple arguments to `print` are joined with one thing (`$,`,
usually an empty string) but multiple list elements interpolated into a string
are joined with another (`$"`, usually a space).

Quote::Code has `qc_to` for heredocs instead of a qc-quoted heredoc terminator.
Matching "terminator quoting determines heredoc interpolation" seemed more
"keep similar things similar".

## Open Issues

What, if anything, do we do now about interpolating into regex?

## Copyright

Copyright (C) 2022, Ricardo Signes.

This document and code and documentation within it may be used, redistributed and/or modified under the same terms as Perl itself.
