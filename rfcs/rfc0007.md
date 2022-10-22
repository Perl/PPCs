# source encoding pragma

## Preamble

    Author:  Ricardo Signes <rjbs@semiotic.systems>
    Sponsor: Ricardo Signes <rjbs@semiotic.systems>
    ID:      0007
    Status:  Draft

## Abstract

This RFC proposes a new pragma, `source::encoding`, to indicate the encoding of
the source document.

## Motivation

At present, unless in a scope in which `use utf8` has been enabled, bytes read
from source correspond directly to the codepoints as which they are
interpreted.  This leads to some surprising behaviors.  A Latin-1 encoded
source file will have its literal strings match Unicode semantics when matching
regular expressions.  Meanwhile, a UTF-8 encoded source file's strings may not
appear to do so, but will behave correctly when printed to a UTF-8 terminal.

All these behaviors can be explained, but can still surprise both beginner and
expert.  To eliminate surprise at runtime, this proposal intends to give the
programmer a means to declare that non-ASCII bytes are a compile-time error.
It also proposes to make that declaration implicit in "use v5.38" and later
version declarations.

## Rationale

The biggest goal here is to make "use v5.38" sufficient to avoid runtime
confusion falling out of non-ASCII source.  Given the complexity of "just make
it all Unicode and UTF-8", the goal is to alert the programmer that they've
used non-ASCII in their source without declaring that they've thought about it.

The behaviors implied by "use VERSION" are generally made individually
controllable, so a separate control must be provided.  Rather than provide a
"use ascii" that parallels "use utf8", a single "use source::encoding" is
provided so that a common name can be used for both declaring ASCII-only and
UTF-8 encoding.

## Specification

A new pragma will be created, source::encoding, which can be given one of two
arguments:  `utf8` or `ascii`.

`use source::encoding "utf8"` will have the same effect as `use utf8`.

`use source::encoding "ascii"` will indicate that a compile-time error should
be raised when reading a non-ASCII byte in the source.

`no source::encoding` will return to the default behavior of reading bytes into
codepoints.

`use v5.38` (and later) will implicitly set the source encoding to ASCII.
Using the feature bundle will have no effect on source encoding.

## Backwards Compatibility

Static analysis that currently attempts to detect `use utf8` will need to
be updated to also detect `use source::encoding ARG`.  This creates a
significant complication, because ARG can be a variable.  On the other hand,
the problem is already quite difficult, because any library loaded at compile
time could affect the encoding of the scope currently being compiled, by dint
of how `$^H` works.

The source::encoding library can be backported to earlier perls, but only for
utf8, not ascii, unless a source filter is used -- which may actually be a
reasonable use case for source filtering.

## Security Implications

None foreseen.

## Examples

Producing examples where non-ASCII source leads to confusion is like shooting
fish in a barrel.

    use strict;
    use feature 'say';

    my $string = "Queensrÿche";

    say length $string;
    say "contains non-words" if $string =~ /\W/;
    say $string;

Many different problems may arise if the source is encoded as Latin-1 versus
UTF-8, whether `use utf8` is inserted, whether `use feature "unicode_strings"`
is enabled, and so on.

By adding `use source::encoding "ascii"`, all (or nearly all) of those problems
are replaced by the simple question of "How shall we represent the 8th position
of that string in the source code?"

## Prototype Implementation

It would be possible to implement this with a source filter, but the author has
not attempted to do so.

## Future Scope

In the future, if the semantics of strings and filehandles are expanded to
better cover encoding issues, it may become practical to change the `use vX`
behavior to prefer UTF-8 to ASCII.

## Rejected Ideas

The first proposal to eliminate non-ASCII source footguns was to have use v5.38
enable the utf8 pragma.  The objection raised was that this would lead to new
kinds of confusion related to decoded (text) strings being printed to
filehandles with no encoding layer.  Although this is explainable behavior, the
current behavior may be less confusing in its practical effect, in some
circumstances.  Instead, "demand ASCII" has been proposed because it keeps
semantics within the common space of ASCII, Latin-1, and UTF-8.

The `encoding` pragma once performed a very similar task to the new proposed
pragma, but for arbitrary encodings.  It has been unsupported for several
years.  Reclaiming this name for this function is tempting, but seems likely to
cause confusion.

## Open Issues

None?

## Copyright

Copyright (C) 2021, Ricardo Signes

This document and code and documentation within it may be used, redistributed
and/or modified under the same terms as Perl itself.
