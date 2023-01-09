# Built-in lexical exporting

## Preamble

    Author:  Curtis Poe <curtis.poe@gmail.com>
    Sponsor:
    ID:      OVID-0019
    Status:  Draft

## Abstract

This RFC proposes adding built-in lexical exporting via an `:export`
attribute.

## Motivation

To eliminate the need for the myriad different CPAN modules for exporting
functions. This RFC is only for functions, not variables.

## Rationale

There are many exporter modules available. The core `Exporter` is the best
known, and has the curious feature of allowing procedural modules to use
inheritance to implement exporting via package variables. It has a clumsy
interface and many alternatives arose:

* [Sub::Exporter](https://metacpan.org/pod/Sub::Exporter)
* [Exporter::Lite](https://metacpan.org/pod/Exporter::Lite)
* [Exporter::Easy](https://metacpan.org/pod/Exporter::Easy)
* [Exporter::Tiny](https://metacpan.org/pod/Exporter::Tiny)

And more, including many modules which manually implement exporting directly
in the `import` method.

There are also many attribute-based exporters:

* [Exporter::Simple](https://metacpan.org/pod/Exporter::Simple)
* [Perl6::Export](https://metacpan.org/pod/Perl6::Export) (not really attributes)
* [Perl6::Export::Attrs](https://metacpan.org/pod/Perl6::Export::Attrs)
* [Exporter::Attributes](https://metacpan.org/pod/Exporter::Attributes)

And so on.

## Specification

### Usage

Because lower-case attribute names are reserved for Perl's future expansion,
we do not need a feature guard. Instead, we can simply `use v5.XX` to pull in
the new feature. However, even that might not be needed if they're using a
future Perl since it doesn't conflict with anything default.

```perl
package My::Module {
    use v5.38;

    sub bar  :export           {...} # @EXPORT_OK
    sub baz  :export(strings)  {...} # %EXPORT_TAGS
    sub bay  :export(strings)  {...} # %EXPORT_TAGS
    sub quux :export(DEFAULT)  {...} # @EXPORT
    sub whee                   {...} # not exported
}
```

The above behaves similarly to:

```perl
package My::Module {
    use base 'Exporter';

    our @EXPORT    = qw(quux);
    our @EXPORT_OK = qw(
        bar
        baz
        bay
    )
    our %EXPORT_TAGS = qw(
        'all'     => \@EXPORT_OK,
        'strings' => qw(bay baz),
    );

    sub bar  {...}
    sub baz  {...}
    sub bay  {...}
    sub quux {...}
    sub whee {...}
}
```

The requested functions are imported lexically via the
`builtin::export_lexically` function, rather than appearing in the package
symbol table. This means that code within that lexical scope can see and call
them as normal, but they are not visible from outside the scope to which they
are exported, such as from other callers or as object or class methods. This
removes the need for `namespace::clean` and related modules.

Upper-case `:export` attribute arguments are reserved for Perl to avoid
clashing with user-defined tags. This is because in this author's experience,
export tags are _usually_ lower-case.

### Interface

The various incantations of `:export` should populate the respective
`@EXPORT`, `@EXPORT_OK`, and `%EXPORT_TAGS` package variables to maintain a
consistent API with the `Exporter` module.

The grammar for the tag is:

    <export>  ::= ':export' [ <options> ]
    <options> ::=  '(' ( 'DEFAULT' | <tags> ) ')'
    <tags>    ::=  <tag> {',' <tag> }
    <tag>     ::=  \w+


### Exporting

* `sub foo :export          {}` Export only on demand
* `sub foo :export(DEFAULT) {}` Always export
* `sub foo :export(strings) {}` Export with `:strings` tag or directly
* `sub foo :export(as,df)   {}` Export with either the `:as` or `:df` tags, or directly

Any function that has a tag or tags (such as `strings` in the above example),
can also be imported directly by name.

### Importing

* `use Mod;` Import only functions declared with `:export(DEFAULT)`
* `use Mod qw(foo bar baz)` Import functions iff declared with `:export`
* `use Mod ':strings';` Import only functions tagged with `strings`
* `use Mod ':all';` All function with any form of `:export` tag are imported

### Implementation

The `builtin::export_lexically` function will be used to provide the actual
functionality. `:export` tags, if used, their options, will be used to decide
which functions used to decide which functions are lexically exported.

This module will not allow unexporting lexical functions.

Only coderefs are supported (variables can be wrapped in coderefs). We may
revisit this decision in the future.

## Backwards Compatibility

For basic usage without an `import` method, there are no
backwards-compatibility issues. There are, however a few surprises.

* Debugger

When using the debugger, it's common to test the return value of a subroutine:

```
auto(-1)  DB<2> v
30
31:    if (@errors) {
32:        warn join "\n" => @errors;
33==>    say status();
34     }
35
36:    unless ( $opt_for{description} ) {
37:        say "Please enter a description for this $opt_for{type} entry:";
38:        chomp( my $description = <STDIN> );
39:        $opt_for{description} = $description;
DB<2> x status()
Undefined subroutine &main::status called at ...
```

The subroutine is bound a compile-time and removed from the namespace. You can
step into it, but you can't call it directly from the debugger. This should be
considered.

* Exporting Spec

We often have the following near the top of our code:

```perl
our @EXPORT    = qw(quux);
our @EXPORT_OK = qw(
    bar
    baz
    bay
)
our %EXPORT_TAGS = qw(
    'all'     => \@EXPORT_OK,
    'strings' => qw(bay baz),
    'other'   => qw(bay)
);
```

That would be rewritten as this:

```perl
sub quux :export(DEFAULT)       {...}
sub bar  :export                {...}
sub baz  :export(strings)       {...}
sub bay  :export(strings,other) {...}
```

Some developers like a single exporting spec at the top of a module. Others
like viewing a function and seeing directly if it can be exported:

```perl
sub foo :export () {
    ...
}
```

* The `import()` method

This one is tricky because we have to maintain backwards-compatibility.

If there is no `import()` method defined in the class  then any attempt to
import a function not designated for export should be a fatal error:

```
$ perl -MMy::Module=whee -E 1
"whee" is not exported by the My::Module module
Can't continue after import errors at -e line 0.
BEGIN failed--compilation aborted.
```

Any attempt to import a non-existent export (e.g., `use Foo ':bar'`) should
also be fatal.

For the following, we'll use the following stub example of an `import()`
method:

```perl
package My::Module {
    use v5.38;

    sub import ($class, @args) {
        ...
    }
    sub bar  :export           {...} # @EXPORT_OK
    sub baz  :export(strings)  {...} # %EXPORT_TAGS
    sub bay  :export(strings)  {...} # %EXPORT_TAGS
    sub quux :export(DEFAULT)  {...} # @EXPORT
    sub whee                   {...} # not exported
}
```

If an `import()` method is defined in the package, it will need to call
`UNIVERSAL::import()` to have the lexical importing handled correctly.

```perl
sub import ($class, @args) {
    my ( $import, $mine ) = separate_args(@args);
    # do stuff with $mine
    UNIVERSAL::import($import->@*);
}
```

For the above, how do we tell `UNIVERSAL::import` which class its importing
into? I suppose we could do `local @_ = $import->@*; goto &UNIVERSAL::import`,
but that's finicky, it's going to break, and we're trying to discourage use of
`@_` in signatured subs. I expect there's some new work done in Perl which I'm
missing here.

## Security Implications

None anticipated, but by having the imports being lexical, it means outside
code might have a harder time replacing these functions. Thus, it might be
more secure.

## Examples

```perl
package My::Module {
    use v5.38;

    sub bar  :export           {...} # @EXPORT_OK
    sub baz  :export(strings)  {...} # %EXPORT_TAGS
    sub bay  :export(strings)  {...} # %EXPORT_TAGS
    sub quux :export(DEFAULT)  {...} # @EXPORT
    sub whee                   {...} # not exported
}

use My::Package 'bar';     # imports bar() and quux

use My::Package ':strings' # imports baz, bay, and quux

use My::Package;           # imports quux

use My::Package ();        # imports nothing

use My::Package qw/bar :strings -quux/; # don't import quux

use My::Module 'whee'; # fatal error
```

## Prototype Implementation

None that I'm aware of, though
[Perl6::Export::Attrs](https://metacpan.org/pod/Perl6::Export::Attrs) seems
the closest.

## Future Scope

We might want to allow variables and constants to be exported. We might always
want export tags to contain other export tags to allow "grouping" of groups.

The 'is not exported' error message could list what is actually exported by
the module.

An introspection API for procedural code (similar to a MOP), could be
provided. As part of this, it could provide an export spec. This could allow
Perl to start providing "smart" documentation that reads the code in addition
to POD.

We could also determine a syntax as importing as a different name. This should
be safe because they're lexical imports.

## Rejected Ideas

This was part of larger "AMORES" pre-RFC to improve the overall state of
procedural code. That pre-RFC was rejected, but I was asked to put this into
its own RFC.

## Open Issues

None?

## Copyright

Copyright (C) 20222-23, Curtis "Ovid" Poe

This document and code and documentation within it may be used, redistributed
and/or modified under the same terms as Perl itself.
