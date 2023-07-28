# A builtin for Lexical Export

## Preamble

    Author:  Paul Evans <PEVANS>
    Sponsor:
    ID:      0020
    Status:  Implemented

## Abstract

Add a new function to the `builtin` package to perform lexical export into the scope currently being compiled.

## Motivation

"Traditional" Perl modules of the era around Perl 5.8 have always used symbolic aliasing into the caller's package to implement their export mechanism. This was the only mechanism available to them at the time, and served well in the era of procedural modules with functions in them. As more code became object-oriented, using packages to implement classes, the problems of making these exports visible as named symbols in the class's namespace became apparent, because each imported function would be visible as a named method on the class, whether it wanted that or not.

A new feature of the Perl 5.18 release was the ability to create lexically-named subroutines, whose names are accessible within some lexical scope but not visible in the package namespace, and thus not visible to callers from outside the code, even via method reflection. Since then, the ability has not been further expanded on by Perl core. While a few CPAN modules have tried to implement the ability, the idea does not seem to have caught on in the majority of cases.

The `builtin` module added in Perl 5.36 was the first core module to export its named functions lexically into the calling scope, rather than symbolically into the caller's package. It aims to set a new standard for better-behaved module exporting, but is currently handled by its own internal implementation which is not yet exposed in a way that other modules can make use of.

## Rationale

By providing a mechanism by which other modules written in pure perl code can take advantage of lexical export themselves, we hope that more will begin to take advantage of this ability. This will lead to modules that behave more politely in the face of the growing collection of modules written in object-oriented style. In addition, as addressed by the "Future Scope" section, we hope this leads on to the creation of better Exporter-like modules or other core perl abilities to take advantage of lexical export.

## Specification

A new function in the `builtin` package, called `export_lexically`. This function takes an even-sized name/value list of pairs. It does not return anything.

```perl
builtin::export_lexically $name1, $value1, $name2, $value2, ...;
```

Each name/value pair acts independently. The name gives the new name to be created in the scope being compiled, and the value is a reference to the item the name should have.

Four kinds of items are supported:

+ If the value is a `CODE` reference, a new named subroutine will be created in the scope. The corresponding name argument may optionally begin with a `&` sigil.

+ If the value is a `SCALAR` reference, a new scalar variable will be created in the scope. The corresponding name argument must begin with a `$` sigil.

+ If the value is an `ARRAY` reference, a new array variable will be created in the scope. The corresponding name argument must begin with a `@` sigil.

+ If the value is a `HASH` reference, a new hash variable will be created in the scope. The corresponding name argument must begin with a `%` sigil.

If no sigil is present on the name argument, it is presumed to be of the first kind, and the value must be a `CODE` reference.

As the only effect this function has is to add new things to the compiletime lexical scope, it shall be an error to call this function during regular runtime, when no code is currently being compiled. It shall also be an error to pass non-reference values, references to unsupported items (e.g. globs), or references whose type does not match the sigil of its name.

## Backwards Compatibility

As this specification adds one new optionally-requested builtin function name there are not expected to be any backward compatiblity concerns. Additionally, the concept of lexical exports and lexically-named subroutines has existed in Perl since version 5.18, so these are not new concepts.

## Security Implications

None are expected.

## Examples

### A simple direct application:

```perl
{
  BEGIN {
    builtin::export_lexically ten => sub { 10 };
  }

  say "Ten plus ten is ", ten() + ten();
}

# ten() is no longer visible here
```

### A tiny module exporting lexically

```perl
package Example::Module;

sub import
{
  shift;

  my %syms;
  $syms{$_} = __PACKAGE__->can( $_ ) // die "No such function '$_'"
    for @_;

  builtin::export_lexically %syms;
}

sub foo { ... }
sub bar { ... }

1;
```

## Prototype Implementation

An initial implementation of this specification has been written at https://github.com/leonerd/perl5/tree/lexically-export, and raised as a Pull Request against core perl5 at https://github.com/Perl/perl5/pull/19895.

## Future Scope

Once a basic mechanism for lexical export exists in the language, it becomes tempting to look at the various Exporter-like modules, either bundled with perl (e.g. [`Exporter`](https://metacpan.org/pod/Exporter) itself) or on CPAN. Various discussions in the past have raised the idea of having those modules perform lexical export of the requested names.

Another possible idea becomes the thought of adding an `:export` attribute for functions and variables, bundling most of the export ability directly into the core language and avoiding most of the need to invoke an external module to do it.

Both of these ideas are out of scope for this PPC, but would be made significantly easier by its completion.

## Rejected Ideas

## Open Issues

## Copyright

Copyright (C) 2022, Paul Evans.

This document and code and documentation within it may be used, redistributed and/or modified under the same terms as Perl itself.
