# Lexically require modules

## Preamble

    Author:  Ovid
    Sponsor:
    ID:      23 
    Status:  Draft

## Abstract

When writing a module, the `use` and `require` statements make the modules
available globally. This leads to strange bugs where you can write
`my $object = Some::Class->new` and have it work, even if you didn't
require that module. This transitive dependency is fragile and will break
if the code requiring `Some::Class` decides to no longer require it.

This PPC proposes:

```perl
package Foo;
use feature 'lexical_require`;
use Some::Module;
```

Within a given lexical scope, **ß**, if the 'lexical_require' feature is used,
code outside of scope **ß** cannot call methods against class names that have
not been explicitly required within the current package, and doing so would
throw an exception. Methods would be allowed against any object (blessed
reference), but not against a class.

Note: that doesn't mean the transitive dependencies aren't available. If
scope **ß** uses `lexical_require` and `Hash::Ordered`, but scope **∂** uses
`Hash::Ordered` but _doesn't_ use `lexical_require`, then `Hash::Ordered` is
still available to everyone as a transitive dependency. However, individual module
authors will still have greater safety in knowing that people are not directly
relying on their internals.

## Motivation

* Accidentally relying on transitive dependencies is fragile because, unless
  documented, transitive dependencies are not guaranteed to exist.
* Currently, loading a module injects it into a global namespace, so it's not
  easy to prevent this problem.
* Transitive dependencies are even more fragile is the code is conditionally
  required:

```perl
if ($true) {
    require Some::Module;
}
```

In the above, the transitive dependency can fail if `$true` is never true.

The initial discussion is on [the P5P mailing
list](https://www.nntp.perl.org/group/perl.perl5.porters/2023/07/msg266678.html).

## Rationale

* The new syntax ensures that the module author can `require` or `use` a module and not
worry that other code will accidentally be dependent on internal implementation details.

## Specification

For the given lexical scope—block or file—`use feature 'lexical_require'` will
allow code to use the required module. Code _outside_ of that scope cannot use
the required module unless it explicitly uses it, or there's another
transitive dependency injecting that module into the global namespace.

```perl
package Foo {
    use feature 'lexical_require';
    use Hash::Ordered;
    no feature 'lexical_require';
    use Some::Class;
    ...
}
my $object = Some::Class->new; # succeeds if `Some::Class` has a `new` method
my $cache  = Hash::Ordered->new; # fails
```

Note that, if possible, this should also apply to package variables. In the
above, `$Hash::Ordered::VERSION` should fail. This is again, to prevent
accidentally relying on code that might not be there. If `Foo` switches from
`Hash::Ordered` to a similar module, all code relying on `Hash::Ordered` as a
transitive dependency would break.  As a module author, I don't want to break
someone else's code just because I changed internal details that they should
not know about.

## Backwards Compatibility

This feature should be 100% backwards compatible for new code. If retrofitted
into existing code, any code relying on a transitive dependency might break
and need to explicitly declare that dependency.

These are probably going to be runtime errors, not compile-time.

Other than the above caveats, I am not aware of any tooling which will be
negatively impacted by this. However, I don't know the
[`Devel::Cover`](https://metacpan.org/pod/Devel::Cover) internals and I
suspect there might be an issue there.

I suspect (hope), that the following will not be impacted:

* [`B::Deparse`](https://metacpan.org/pod/B::Deparse)
* [`Devel::NYTProf`](https://metacpan.org/pod/Devel::NYTProf)
* [`PPI`](https://metacpan.org/pod/PPI) (hence [`Perl::Critic`](https://metacpan.org/pod/Perl::Critic) etc)

## Security Implications

If anything, this might improve security by not allowing code to have an
accidental dependency on code it doesn't explicitly use.

## Examples

From the above:

```perl
package Foo {
    use feature 'lexical_require';
    use Hash::Ordered;
    no feature 'lexical_require';
    use Some::Class;
    ...
}
my $object = Some::Class->new; # succeeds if `Some::Class` has a `new` method
my $cache  = Hash::Ordered->new; # fails
```

## Prototype Implementation

None.

## Future Scope

In the future, it might be nice to have `namespace` declarations.

```perl
namespace Las::Vegas;
package ::Casino; # package Las::Vegas::Casino

```

For the above, what happens in `Las::Vegas` stays in `Las::Vegas`.

The above allows you to declare a new namespace and everything within that
namespace is private to it. Only code that is officially exposed can be used.
Companies can have teams using separate namespaces and only official APIs can
be accessed. You can't "reach in" and override subroutines. This would require
developers to plan their designs more carefully, including a better
understanding of dependency injection and building flexible interfaces.

## Rejected Ideas

There really hasn't been any previous solutions on the CPAN that I've seen.
I've seen closure-based solutions using lexical subs, but they're harder to
write.

## Open Issues

We may have issues with test suites. They often take advantage of locally
overriding/replacing a subroutine and if that's declared in a transitive
dependency, it might fail.

## Copyright

Copyright (C) 2023, Curtis "Ovid" Poe

This document and code and documentation within it may be used, redistributed and/or modified under the same terms as Perl itself.
