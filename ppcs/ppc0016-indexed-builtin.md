# A built-in for getting index-and-value pairs from a list

## Preamble

    Author:  Ricardo Signes <rjbs@semiotic.systems>
    Sponsor:
    ID:      0016
    Status:  Implemented

## Abstract

This PPC proposes `indexed`, a new builtin for interleaving a list of values
with their index in that list.  Among other things, this makes key/value
iteration on arrays easy.

## Motivation

With v5.36.0 poised to add n-at-a-time foreach, easily getting a list of
index/value pairs from an array makes iteration over the pairs also becomes
easy.

## Rationale

If we start with the specific case of iterating over the indexes and values of
an array using two-target foreach, we might write this:

```perl
for my ($i, $value) (%array[ keys @array ]) {
  say "$i == $value";
}
```

This is tolerable, but a bit verbose.  If we bury our target array deep in a
structure, we get this:

```perl
for my ($i, $value) ($alpha->{beta}->[0]->%[ keys $alpha->{beta}->[0]->@* ]) {
  say "$i == $value";
}
```

This is pretty bad.

With `indexed`, we write this:

```perl
for my ($i, $value) (indexed $alpha->{beta}->[0]->@*) {
  say "$i == $value";
}
```

This is probably about as simple as this can get without some significant new
addition to the language.

## Specification

    indexed LIST

`indexed` takes a list of arguments.

In scalar context, `indexed` evalutes to the number of entries in its argument,
just like `keys` or `values`.  This is useless, and issues a warning in the new
"scalar" category:

    Useless use of indexed in scalar context

In void context, the `Useless use of %s in void context` warning is issued.

In list context, `indexed LIST` evalutes to a list twice the size of the list,
meshing the values with a list of integers starting from zero.  All values are
copies, unlike `values ARRAY`.  (If your LIST was actually an array, you can
use the index to modify the array that way!)

## Backwards Compatibility

There should be no significant backwards compatibility concerns.  `indexed`
will be imported only when requested.  Static analysis tools may need to be
updated.

A polyfill for indexed can be provided for older perls, but may not be as
optimizable.

## Security Implications

Nothing specific predicted.

## Examples

(See the examples under **Rationale**.)

I expect that docs for `keys` and `values` will be updated to reference
`indexed` as well, and we'll add a note about it to the documentation on `for`
and possibly pair slices.

When n-at-a-time foreach is no longer experimental, we should refer to the
combination of `for my (...) (...)` with `indexed` as forming an alternative to
`each` in the documentation for `each`.

## Prototype Implementation

None.

## Future Scope

I believe this will be complete as is.

## Rejected Ideas

This proposal replaces one for `kv` which could be called on hash or array
literals to act like a combination of `keys` and `values`.

That proposal replaced one for a slice syntax that evaluated to a slice that
omitted nothing.

## Open Issues

None?

## Copyright

Copyright (C) 2021, Ricardo Signes.

This document and code and documentation within it may be used, redistributed
and/or modified under the same terms as Perl itself.
