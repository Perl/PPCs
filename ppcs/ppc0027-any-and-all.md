# More list processing operators inspired by List::Util

## Preamble

    Author:  Paul Evans <PEVANS>
    Sponsor: 
    ID:      0027
    Status:  Exploratory

## Abstract

Creates several new list-processing operators, similar to the existing `grep`, inspired by the same-named functions in modules like `List::Util` or `List::Keywords`.

## Motivation

Most code of any appreciable size tends to make use of at least the `any` or `all` functions from `List::Util`. Due to limits of their implementation they are not as efficient to call as core's `grep` operator. The implementations provided by `List::Keywords` are more efficient, on the same level as core's `grep`, though being a non-core module it does not appear to be used anywhere near as much in practice.

## Rationale

## Specification

New named features that, when enabled, activate syntax analogous to the existing `grep` operator, named `any` and `all`:

```
any { BLOCK } LIST

all { BLOCK } LIST
```

These operators are similar to `grep` in scalar context, though yield a simple boolean truth value relating to how many input values made the filter block yield true. `any` yields true when its block yields true for at least one of its input values, or false if they all yield false. `all` yields true only if every input value makes the block yield true, or false if at least one yields false.

A consequence of these rules is what happens when given an empty list. A call to `any` with an empty list does not have any input values which made the block return true, so its result is false. Conversely, a call to `all` with an empty list does not have any input values which made the block return false, so its result is true.

The key difference between these operators and `grep` is that these will short-circuit and stop evaluating the block once its result is determined. In particular, the first time `any` sees a true result, it knows its result so it can stop; only testing more input values while each yields false. Conversely, `all` will stop as soon as it sees a false result, knowing that to be its answer; it only continues while each value yields true from the block. This short-circuiting is a key reason to choose these over the `grep` operator.

Additionally, because each operator returns a fixed boolean truth value, the caller does not have to take special precautions against a value that would appear false, which satisfies the filter code block. Such a value would cause the operator to return true, even if the matching value itself appears false.

Like `grep`, each is a true operator, evaluating its block expression without an interposed function call frame. Thus any `caller` or `return` expression or similar within the block will directly affect the function containing the `any` or `all` expression itself.

These operators only yield a single scalar; in list context therefore they will just provide a single-element list containing that boolean scalar. This is so that there are no "surprises" if the operator is used in a list context, such as when building a key/value pair list for the constructor of an object. By returning a single false value even as a list, rather than an empty list, such constructions do not cause issues.

For example:

```
Some::Class->new(
    option => (any { TEST } list, of, things),
    other  => $parameter,
);
```

## Backwards Compatibility

As these new operators are guarded by named features, there are no immediate concerns with backward compatiblity in the short-term.

In the longer term, if these named features become part of a versioned feature bundle that is enabled by a corresponding `use VERSION` declaration there may be concerns that the names collide with functions provided by `List::Util` or similar modules. As the intention of these operators is to provide the same behaviour, this is not considered a major problem. Differences due to caller scope as outlined above may be surprising to a small number of users.

## Security Implications

## Examples

```
use v5.40;
use feature 'any';

if( any { $_ > 10 } 5, 10, 15, 20 ) { say "A number above 10" }
```

## Prototype Implementation

The overall behaviour of these operators is primarily demonstrated by functions from core's existing [`List::Util`](https://metacpan.org/pod/List::Util) module. Additional examples of a more efficient keyword-and-operator implementation can be found in [`List::Keywords`](https://metacpan.org/pod/List::Keywords).

## Future Scope

### Named Lexicals

The `List::Keywords` module also provides an interesting "named lexical" syntax to its operators, allowing the user to specify a lexical variable, rather than the global `$_`, to store each item for iteration:

```
use List::Keywords qw( any );

if( any my $item { we_want($item) } @items ) {
    say "There's an item we want here";
}
```

These lexicals are useful when nesting multiple calls to list-processing operators, to avoid collisions in the use of the `$_` global, and lead to cleaner code. They are also useful for suggesting how to support n-at-a-time behaviour of `grep` and `map`-like functions.

If this feature is to be considered, it will require careful thought on how it might interact with the so-far-unspecified idea of accepting `any EXPR, LIST` as `grep` currently does. I would recommend not allowing that variant, to allow for easier implementation of these named lexicals in future as they provide advantages that outweigh the minor inconvenience of having to wrap the expression in brace characters.

### Other Operators

* The other two variant behaviours of `none` and `notall`. These simply invert the sense of the filter block.

* Another variation on the theme, `first`. This returns the value from the list itself, that first caused the filter block to be true.

## Rejected Ideas

### Block-less syntax

Supporting syntax analogous to the "deferred-expression" form of `grep EXPR, LIST`.

### Keywords as Junctions

Using the `any` and `all` keywords to make junction-like behaviour. Such is already provided by other modules, for example [`Data::Checks`](https://metacpan.org/pod/Data::Checks) in a title-case form and thus would not collide with the all-lowercase keywords provided here. This is already possible:

```
use Data::Checks qw( Any ... );
use Syntax::Operator::Is;

if( $x is Any( things... ) ) { ... }
```

In any case, as junctions behave like values, they do not require special syntax like the block-invoking keywords proposed here, so they can be provided by regular function-call syntax from regular modules.

## Open Issues

* There could be anything up to five new operators added by this idea. Do they all get their own named feature flags? Do they all live under one flag?

* Should the flag be called `any`? That might be confusing as compared to the `:any` import tag which would request all features.

## Copyright

Copyright (C) 2024, Paul Evans.

This document and code and documentation within it may be used, redistributed and/or modified under the same terms as Perl itself.
