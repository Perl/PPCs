# Map with different topic variable

## Preamble

    Author:     Graham Knop <haarg@haarg.org>
    ID:         0033
    Status:     Draft

## Abstract

Allow `map` and `grep` to be given a different variable to be used as the topic
variable, rather than using `$_`. Also allow multiple variables to be given to
do n-at-a-time iteration.

## Motivation

Traditionally, `map` and `grep` loop over a list, aliasing `$_` to each list
entry. While this can be convenient in many cases, it gets awkward when nested
maps are needed. You need to manually save the `$_` value in another variable.

```perl
my %hash = (
  foo => [ 1, 2, 3, 4 ],
  bar => [ 5, 6, 7, 8 ],
);

my @out = map {
  my $key = $_;
  map { "$key: $_" } $hash{$key}->@*;
} keys %hash;
```

Using `$_` can also be dangerous if you need to call code not under your
direct control, due to it being the implicit target of operations like
`readline`.

It would be more convenient if you could use a different variable for the
topic, similar to what `for` allows.

```perl
my @out = map my $key {
  map my $i { "$key: $i" } $hash{$key}->@*;
} keys %hash;
```

A natural extension of this syntax would be to allow n-at-a-time iteration, as
`for` can do on perl 5.36+.

```perl
my @out = map my ($key, $val) {
  map my $i { "$key: $i" } $val->@*;
} %hash;
```

All of this would apply to `grep` as well.

As the syntax proposed is currently invalid, a feature should not be needed to
enable it.

## Rationale

The syntax chosen is meant to follow from the syntax of `for`, treating `map`
as "`for` as an expression". The chosen syntax does not create any
ambiguities, and naturally extends to follow the syntax used by `for` for
n-at-a-time iteration.

## Specification

### `map my VAR BLOCK LIST`

This will evaluate `BLOCK` for each element of `LIST`, aliasing `VAR` to each
element. Its behavior will otherwise match `map BLOCK LIST`. It is only
possible to use `my` variables for this.

### `map my (VAR, VAR) BLOCK LIST`

This will evaluate `BLOCK` for each set of two elements in `LIST`, aliasing
the `VAR` to the first of each set, and the second `VAR` to the second. More
than two variables can be used to iterate over sets of three or more items.

On the last iteration, there may not be enough elements remaining to fill
every `VAR` slot. In this case the extra `VAR` slots will be filled with
`undef`.

## Backwards Compatibility

As the syntax chosen is currently invalid, it should not present any backwards
compatibility concerns.

While the parsing of `map` and `grep` is complex and has ambiguities, the
additions proposed are bounded and simple to parse, so they do not introduce
any new ambiguities.

## Security Implications

None foreseen.

## Examples

```perl
my @my_array = (11 .. 14);

my @trad_map = map { $_ + 1 } @my_array;
# ( 12, 13, 14, 15 )

my @new_map = map my $f { $f + 1 } @my_array;
# ( 12, 13, 14, 15 )

my @pair_map = map my ($f, $g) { $f + $g } @my_array;
# ( 23, 27 )

my @new_grep = grep my $f { $f > 12 } @my_array;
# ( 13, 14 )

my %my_hash = (
   first_key => 11,
   second_key => 14,
);

my %pair_grep = grep my ($f, $g) { $f =~ /sec/ } %my_hash;
# ( second_key => 14 )
```

## Prototype Implementation

  - List::Util includes the functions `pairmap` and `pairgrep` which can do
    2-at-a-time iteration. These use the "magic" `$a` and `$b` variables.

## Future Scope

  - Doing n-at-a-time iteration is useful, but authors may not have a use for
    all of the variables, especially with `grep`. Could the extra variables be
    eliminated? Maybe `grep my ($key, undef) { ... } ...` could be supported?

  - `map`, `grep`, and even `for` could possibly be extended to allow
    refaliasing syntax for their variables.

    ```perl
    my %hash = (
      key1 => [ 1, 2, 3 ],
      key2 => [ 4, 5, 6 ],
    );
    my %out = map my ($key, \@values) { ... } %hash;
    ```

  - Subs with a `(&@)` prototype are, in part, meant to allow mimicking the
    syntax of `map` and `grep`. An example of this being `List::Util::any`:

    ```perl
    use List::Util qw(any);

    my $found = any { /ab/ } qw( 1234 abcd );
    ```

    It would be desirable to be able to to write a function that accepted the
    same syntax this this PPC proposes.

## Rejected Ideas

  - `my` is required because the behavior of `for` when used with non-`my` or
    predeclared variables is confusing and hard to explain. The newer syntax
    `for my ($var1, $var2) { ... }` also requires `my`, both for that reason
    as well as implementation details. Making the `my` implicit also
    introduces a difference from `for` making it harder to document and
    explain.

## Open Issues

Nothing at this time.

## References

  - Pre-PPC discussion: https://www.nntp.perl.org/group/perl.perl5.porters/2022/11/msg265104.html

## Copyright

Copyright (C) 2022, Graham Knop

This document and code and documentation within it may be used, redistributed
and/or modified under the same terms as Perl itself.
