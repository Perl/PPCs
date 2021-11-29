# Support overloaded objects in join(), substr() builtins

## Preamble

    Authors: Eric Herman <eric@freesa.org>, Philippe Bruhat <book@cpan.org>
    Sponsor: Paul Evans <leonerd@leonerd.org.uk>
    ID:
    Status:  Draft

## Abstract

As of Perl version 5.34, `overload` is incomplete and surprising for
string operations:

* the `join` builtin should be using the `concat` (`.`) overload
  when operating on overloaded objects (including the separator),
* `overload` should support the `substr` operation.

## Motivation

The perl builtin `join`, when operating on overloaded objects will
stringify them and the result will no longer be an object of the class
with overloads, even if the `concat` (`.`) function is overloaded.

Additionally, the `overload` module does not currently support overloading
the `substr` operation.

The current behaviour is inconsistent and confusing.

There is at least one CPAN module which works around the `join`
deficiency, (the [`join` function of
`String::Tagged`](https://metacpan.org/pod/String::Tagged#join), but not
universally.

The CPAN module
[`overload::substr`](https://metacpan.org/pod/overload::substr)) is a
complete solution, but we think that deferring to CPAN when Perl should
do the right thing is not a satisfying answer.

## Rationale

Extending `overload` to support `substr`, and `join` to use `concat`
would be less surprising and more consistent with the "Do What I Mean"
spirit of Perl.

Adding this to Perl would make `overload::substr` obsolete and simplify
the implementation of `String::Tagged` and other similar modules which
have written their own `join`.

## Specification

The documentation for `join` would be amended with the following:

> If any of the arguments (`EXPR` or `LIST` contents) are objects which
> overload the concat (`.`) operation, then that overloaded operation
> will be invoked.

The documentation for `overload` would be amended to extend the complete
list of keys that can be specified in the `use overload` statement to
include `substr`. The module `overload::substr` code and documentation
should be used as a reference and guide for implementation in core. Note
that the current documentation for this module highlights a need for
additional tests, which must be a part of a core implementation.

An open question is whether `split` requires an overload target,
and should a fallback implementation be provided using the
overloaded `substr`?

## Backwards Compatibility

### `substr`

There is no need autogenerate an implementation for `substr` if it is
missing, as the stringification will happen as it does now.

`overload::substr` would become an optional dependency for modules
that needs to support older Perls.

### `join`

If code relies upon the `join` function to convert objects to plain
strings, this would break that code, thus a feature flag seems
necessary.

Modules with overloading, which had to implement their own `join` (to
get a different behaviour than the stringification of the builtin
`join`) could add version-conditional logic to work with different
versions of Perl and delegate to the builtin `join` if the feature is
available. New modules may choose not to implement their own `join`,
thus dropping support for older versions of perl.

## Security Implications

We do not yet foresee security issues. Guidance is welcome.

## Examples

For `join`:

```perl
# when @list contains elements with concat overloading,
# we expect this code:
my $ret = join $sep, @list;

# to behave like this code:
my $ret = reduce { ( $a . $sep ) . $b } @list;
```

For `substr`, the `overload::substr` module will be the guide for an
initial implementation. Its documentation includes the following example:

```perl
package My::Stringlike::Object;

use overload::substr;

sub _substr
{
   my $self = shift;
   if( @_ > 2 ) {
      $self->replace_substr( @_ );
   }
   else {
      return $self->get_substr( @_ );
   }
}
```

## Prototype Implementation

For `substr`: <https://metacpan.org/pod/overload::substr>.

## Future Scope

Looking at `perlfunc`, we find numerous builtins which, when applied
on a string, will return a modified version of the string. Each of these
is worthy of discussion:

* `chomp`
* `chop`
* `fc`
* `lc`
* `lcfirst`
* `quotemeta`
* `reverse`
* `split`
* `tr///` and `y///`
* `uc`
* `ucfirst`

## Rejected Ideas

Not applicable.

## Open Issues

None yet.

## Copyright

Copyright (C) 2021, Philippe Bruhat and Eric Herman.

This document and code and documentation within it may be used,
redistributed and/or modified under the same terms as Perl itself.
