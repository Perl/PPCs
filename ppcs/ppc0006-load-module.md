# Module Loading with "load\_module"

## Preamble

    Author:  Ricardo Signes <rjbs@semiotic.systems>
    Sponsor: Ricardo Signes <rjbs@semiotic.systems>
    ID:      0006
    Status:  Incorporated

## Abstract

This PPC proposes a new built-in, "load\_module", to load modules by module
name (rather than filename) at runtime.

## Motivation

Perl's `require` built-in will interpret a string argument as a filename and
attempt to load this filename in @INC.  Often, a Perl programmer would like to
load a module by module name at runtime.  That is, they want to pass `require`
a variable containing `Foo::Bar` rather than `Foo/Bar.pm`.  To solve this, a
number of libraries have been published to the CPAN with various quirks, extra
behaviors, and bugs.  This has led to more than one run of "everyone should
switch from using loader A to loader B."

## Rationale

This PPC proposes exactly one behavior:  load a named module by module name or
die trying.  Putting this in the core should allow using exactly the same
loading logic as the core uses.  Avoiding any further behaviors or add-ons
eliminates argument about what or how.

## Specification

```pod
=item load_module EXPR

This loads a named module from the inclusion paths (C<@INC>).  EXPR must be
a string that provides a module name.  It cannot be omitted, and providing
an invalid module name will result in an exception.

The effect of C<load_module>-ing a module is the same as C<require>-ing, down
to the same error conditions when the module does not exist, does not compile,
or does not evalute to a true value.

C<load_module> can't be used to require a particular version of Perl, nor can
it be given a bareword module name as an argument.

C<load_module> is only available when requested.
```

Note:  The "how" of "when requested" is to be determined.  If implemented
today, it would be provided behind "use feature".  [Current p5p
discussion](https://github.com/Perl/PPCs/blob/master/ppcs/ppc0009.md) suggests
that in the future it would be imported with std.pm or builtin.pm.

## Backwards Compatibility

No backward compatibility concerns are identified here.  The deparser and other
tools will need updating.  It should be fairly possible to provide a polyfill
for this behavior, with some minor rough edges, by falling back to pure Perl
module name validation and eval or filename-based require.

## Security Implications

The major security problem to be considered is arbitrary code loading.  If the
module name validation is robust, then we should see the same guarantees as
with `require`.

## Examples

For examples, consider Module::Load, Class::Load, Module::Runtime, and others.
A simple use case might be:

```perl
my $config = read_config;
for my $module (keys $config->{prereq}->%*) {
  my ($version) = $config->{prereq}{$module};
  load_module $module;
  $module->VERSION($version) if defined $version;
}
```

## Prototype Implementation

Numerous prototype implementations exist, in effect.  This PPC is the request
to build the real deal.

## Future Scope

One feature seen in several module loaders is "try to load, returning false if
it fails" or variations on that theme.  These, among other things, allow
differentiation between failure to find the file, failure to compile, and other
errors.

These problems already exist with `use` and `require`.  Addressing them through
more clearly-defined exceptions has been proposed in the past, and this PPC has
not attempted to fix them just for `load_module`.  Instead, a future fix should
address all at once.

## Rejected Ideas

A second argument to load, to specify a minimum version, has not been included.
This is already covered by calling `$module->VERSION($version)` and would be a
further distinction from `require`.

## Open Issues

* how is the feature made available?

## Copyright

Copyright (C) 2021, Ricardo Signes

This document and code and documentation within it may be used, redistributed
and/or modified under the same terms as Perl itself.
