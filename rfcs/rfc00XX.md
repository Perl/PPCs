# Built-in lexical exporting

## Preamble

    Author:  Curtis Poe <curtis.poe@gmail.com>
    Sponsor:
    ID:      XX
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

And so on.

## Specification

Because lower-case attribute names are reserved for Perl's future expansion,
we do not need a feature guard. Instead, we can simply `use v5.XX` to pull in
the new feature.

    package My::Module {
        use v5.38;

        sub bar  :export            {...} # @EXPORT_OK
        sub baz  :export(:strings)  {...} # %EXPORT_TAGS
        sub bay  :export(:strings)  {...} # %EXPORT_TAGS
        sub quux :export(:DEFAULT)  {...} # @EXPORT
        sub whee                    {...} # not exported
    }

The above can be written as this:

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

Imported functions are lexical and do not exist in the importing namespace,
removing the need for namespace::clean and related modules.

Upper-case `:export` attribute arguments are reserved for Perl to avoid
clashing with user-defined tags. This is because in this author's experience,
export tags are _usually_ lower-case.

## Backwards Compatibility

For basic usage without an `import` method, there are no
backwards-compatibility issues. There are, however a few surprises.

* Debugger

When using the debugger, it's common to test the return value of a subroutine:

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

The subroutine is bound a compile-time and removed from the namespace. You can
step into it, but you can't call it directly from the debugger. This should be
considered.

* Exporting Spec

We often have the following near the top of our code:

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

Note that there is a need to duplicate some function names.

Some developers like a single exporting spec at the top of a module. Others
like viewing a function and seeing directly if it can be exported:

    sub foo :export () {
        ...
    }

* The `import()` method

This one is tricky because we have to maintain backwards-compatibility.

If there is no `import()` method defined in the class (inheriting does not
count?), then any attempt to import a function not designated for export should
be a fatal error:

    $ perl -MMy::Module=whee -E 1
    "whee" is not exported by the My::Module module
    Can't continue after import errors at -e line 0.
    BEGIN failed--compilation aborted.

For the following, we'll use the following stub example of an `import()`
method:

    package My::Module {
        use v5.38;

        sub import ($class, @args) {
            ...
        }
        sub bar  :export            {...} # @EXPORT_OK
        sub baz  :export(:strings)  {...} # %EXPORT_TAGS
        sub bay  :export(:strings)  {...} # %EXPORT_TAGS
        sub quux :export(:DEFAULT)  {...} # @EXPORT
        sub whee                    {...} # not exported
    }

If an `import()` method is defined, it's possible that it might have special
importing functionality provided, but we will _still_ allowing importing of
functions with `:export` tags. We pass the import arguments unchanged to the
`import()` method (in `@args`, above) and the module author will be
responsible for filtering those manually. This seems unfortunate, but if
someone uses `use v5XX`, their import mechanism shouldn't suddenly break.

As a counter-argument, it's unlikely someone will be using `:export`
attributes, so we could simply filter out all subnames so they don't populate
`@args`, or filter out all subnames tagged with `:export`.

For the above, that means that these are problematic:

    use My::Module ':srtings';
    use My::Module 'whee';

For `:srtings`, that's almost certainly misspelled. For `whee`, that's not
supposed to be exported, but perhaps the module author wants to allow that in
the argument list.

To side-step this, we could create an alternate to the `import()` method,
perhaps as something analogous to a phaser, but taking arguments, or a new
`IMPORT` method. Bike-shedding may now commence!

## Security Implications

None anticipated, but by having the imports being lexical, it means outside
code might have a harder time replacing these functions.

## Examples

    package My::Module {
        use v5.38;

        sub bar  :export            {...} # @EXPORT_OK
        sub baz  :export(:strings)  {...} # %EXPORT_TAGS
        sub bay  :export(:strings)  {...} # %EXPORT_TAGS
        sub quux :export(:DEFAULT)  {...} # @EXPORT
        sub whee                    {...} # not exported
    }

    use My::Package 'bar';     # imports bar() and quux

    use My::Package ':strings' # imports baz, bay, and quux

    use My::Package;           # imports quux

    use My::Package ();        # imports nothing

    use My::Package qw/bar :strings -quux/; # don't import quux

    use My::Module 'whee'; # fatal error

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

Copyright (C) 2022, Curtis "Ovid" Poe

This document and code and documentation within it may be used, redistributed
and/or modified under the same terms as Perl itself.
