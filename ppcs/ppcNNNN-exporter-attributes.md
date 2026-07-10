# Exporter Attributes

## Preamble

    Author:  Paul Evans <PEVANS>
    Sponsor:
    ID:      TODO
    Status:  Exploratory

## Abstract

Adds a core-builtin mechanism for exporting named symbols from one module to another, providing abilities similar to the familiar `Exporter.pm` and others, in a core-supported way. Individual items to be exporter are annoted locally with an attribute, rather than needing to be named in one list.

## Motivation

The concept of exporting symbols from one module to another has been around for a long time in Perl. The `Exporter.pm` module is as old as Perl 5 itself, and is the standard solution used for this task. However, there are also many CPAN-based alternatives, which offer more or different abilities on this idea.

Perl version 5.18 added the idea of lexical subroutines, whose names are looked up in the current lexical scope, rather than being visible to an entire package. This has useful properties for keeping that name local to a particular scope, and is especially useful when the importing scope is part of an object class. Lexical subroutine names are not candidates for named method resolution on instances of that class, in the way that symbolic subroutines are.

Perl version 5.36 added the `builtin` namespace within the interpreter, which adds a number of useful utility functions. The `builtin` namespace also acts as an exporter, whose import effects are applied lexically rather than symbolically. This means that code which uses functions from the `builtin` module manage to keep those names separate from anything externally visible. The `builtin` module also provides a function for modules to implement this lexical import ability for themselves. As yet, no core-supplied exporter solution exists in an easy-to-use form for other modules to use.

Several CPAN-based alternatives exist:

 * [`Exporter::Attribute`](https://metacpan.org/pod/Exporter::Attribute)

 * [`Exporter::Simple`](https://metacpan.org/pod/Exporter::Simple)

 * [`Export::Attrs`](https://metacpan.org/pod/Export::Attrs)

 * [`Export::Lexical`](https://metacpan.org/pod/Export::Lexical)

This specification borrows inspiration from these, along with overall the discussion held on [`PPCs#80`](https://github.com/Perl/PPCs/discussions/80)

## Specification

### In the exporting module

The new mechanism is activated on package by using the new `:exporter` attribute on the package declaration itself. If there are multiple `package` statements, any of them are sufficient to activate the mechanism; it does not need to be repeated on each. This attribute can be considered the new equivalent of a `use Exporter` declaration.

```perl
package MyProject::Utilities :exporter;

...
```

Once activated, individual elements defined within the package can be made available as *candidates for* import by another module by applying the `:export` attribute to them. While this will primarily be found applied to subroutines, this proposal also allowed it on variables.

```perl

sub a_utility_function :export ( $arg0, @args )
{
   ...
}
```

The presence of this attribute only marks that the module allows this item to be imported by a calling scope, not that it is forced upon the caller. This attribute can be considered equivalent to naming the exported item in the `@EXPORT_OK` list for `Exporter`, for example.

Additional arguments can be passed to the `:export` attribute, separated by spaces.

* `:export(name)` provides a different name that this item will appear to be visible as to importers. By default each exported item will appear to be visible to callers under the same name as its definition within its own module.

* `:export(:tagname)` adds this item to the export group named by the given tag.

* `:export(:DEFAULT)` adds this item to the "default" export group, which is imported into callers by default. This can be considered equivalent to `Exporter`'s `@EXPORT` list, and ought to be rarely used.

While the above examples all use symbolic subroutines, it is also permitted to attach the `:export` attribute to lexicals when defined within the main scope of the module's file. Whether an exported item was defined symbolically or lexically, it appears identically to importers of the module. This should be considered an internal detail specific to the current implementation of the exporting module, and has no relevant meaning on how the importer can use the item.

```perl
package MyProject::Utilities :exporter;

my sub another_utility_function :export
{
   ...
}
```

Attempts to attach the `:export` attribute to lexical items defined inside subroutines or looping constructs should be rejected at compile-time:


```perl
sub a_function {
   # This should raise a compiletime error
   my sub this_is_not_allowed :export { ... }
}

foreach ( @ARGV ) {
   # As should this 
   my sub this_is_not_allowed :export { ... }
}
```

Since variables are imported by a name that includes the leading sigil, the names of subroutines vs variables are syntactically distinct, and thus it is permitted to export a subroutine and a variable (or even multiple variables of different sigils) all having the same basename. However, attempts to export two or more items that would have matching names should at least provoke a warning, if not an error. This specification does not currently recommend one over the other.

It is unlikely to be useful to attach the `:export` attribute to a `method` subroutine within a class, though perl will still understand and be able to implement what was requested. This situation should provoke a warning in the exporting module.

Since the exporting ability revolves around adding an `import` method to the exporting package, it should be considered at least a warning (if not an error) for the package to otherwise attempt to declare or provide its own `import` method, as this will directly conflict with the one implicitly provided by use of the `:exporter` attribute. However, there may still exist cases where the exporting module wishes to augment the behaviour of this `import` method - this is explored further in the "Open Issues" section below.

As this behaviour is specified as being a lexical import, it would not be considered a good idea to update existing modules that currently use `Exporter.pm` into using this instead, as that will surprise existing users of the module with a change in import semantics. This mechanism is primarily intended for newly-written code, or when such an update can be effectively handled without surprise.

### In the importing site

Other code can then import this symbol into a lexical scope by using a familiar-looking `use` declaration. Note that by default this import will happen lexically, the same behaviour as the `builtin` module, various core-provided pragma modules, and as implemented by some CPAN exporters such as [`Exporter::Lexical`](https://metacpan.ord/pod/Exporter::Lexical).

```perl
{
    use MyProject::Utilities qw( a_utility_function );
    a_utility_function( ... );
}
```

This should have the same semantics as the familiar `Exporter.pm`, with regard to a default import list, import groups, and so on. Namely, that lacking an import list is equivalent to requesting the `:DEFAULT` group, and any group names can be specified along with indidividual item names to request importing all of the items within those groups.

Experience with `Exporter.pm` and other replacements of it suggests that it would also be useful to provide the ability to rename imports (that is, give the imported item a different name in the importing scope from the name it had in the exporter), and to omit specifically-named items from import groups or the default group. As yet no specific syntax suggestion is made here; that remains an item in the "Open Issues" list.

## Backwards Compatibility

This entire specification is implemented in two new named attributes, which would have been compile-time errors on older Perls that do not recognise them. As such there are no compatibility issues foreseen by this proposal.

## Security Implications

This proposal doesn't introduce any new issues that weren't already present in existing mechanisms like `Exporter.pm`. As long as the exporting module does not make use of the `:DEFAULT` import group, then importing code only receives new names that were specifically requested.

While the `:DEFAULT` group does provide the ability for exporter modules to inject function names into unwitting code that imports it, such ability has always been present since the earliest versions of `Exporter.pm` itself, shipped with the very first version of Perl 5.

Exporting modules should be careful when applying the `:DEFAULT` import group. When explaining this new proposal in core perl documentation, the dangers of this particular part should be explained in detail, to minimise the risk of users inadvertantly causing problems by using it.

## Examples

## Prototype Implementation

Many of the CPAN modules named in the Motivation section can be considered as prior art on the syntax and semantics, though none of them are directly usable as a true core-provided implementation of the specification.

## Future Scope

## Rejected Ideas

## Open Issues

* Clarify what "valid but dubious" situations should provoke warnings, or errors. Should multiple symbols of the same name be an actual error? Or just a warning with last-match-wins sematnics?

* How to specify export of anything not created by a `sub`, `my sub`, `our` or `my` declaration in the module's source code (XSUBs, `use constant`s, reëxport of imported items, etc)

* Whether to permit customisation of the `import` method in some way in the exporting module. Perhaps consider an `IMPORT` phaser block or similar ideas that have better specified ways to coëxist alongside the implicit import behaviour

* How to request renaming and omitting of items from groups, at import-time

* Whether to permit, and if so how, the importer to request that items be imported symbolically rather than lexically

* Whether to permit, and if so how, the exporter to change its own default behaviour to being symbolic. If we permit this, then modules that currently use `Exporter.pm` or other modules could switch to using this core-provided mechanism without surprising any existing users of those modules

## Copyright

Copyright (C) 2026, Paul Evans.

This document and code and documentation within it may be used, redistributed and/or modified under the same terms as Perl itself.
