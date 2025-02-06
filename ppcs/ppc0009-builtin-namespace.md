# Namespace for Builtin Functions

## Preamble

    Author:  Paul Evans <PEVANS>
    Sponsor:
    ID:      0009
    Status:  Incorporated

## Abstract

Define a mechanism by which functions can be requested, for functions that are provided directly by the core interpreter.

## Motivation

The current way used to request utility functions in the language is the `use` statement, to either request some symbols be imported from a utility module (some of which are shipped with perl core; many more available on CPAN), or to enable some as named features with the `use feature` pragma module.

```perl
use List::Util qw( first any sum );
use Scalar::Util qw( refaddr reftype );
use feature qw( say fc );
...
```

In this case, `say` and `fc` are provided by the core interpreter but everything else comes from other files loaded from disk. The `Scalar::Util` and `List::Util` modules happen to be dual-life shipped with core perl, but many others exist on CPAN.

This document proposes a new mechanism for providing commonly-used functions that should be considered part of the language:

```perl
use builtin qw( reftype );
say "The reference type of ref is ", reftype($ref);
```

or

```perl
# on a suitably-recent perl version
say "The reference type of ref is ", builtin::reftype($ref);
```

**Note:** This proposal largely concerns itself with the overal *mechanism* used to provide these functions, and expressly does not go into a full detailed list of individual proposed functions that ought to be provided. A short list is given containing a few likely candidates to start with, in order to experiment with the overall idea. Once found stable, it is expected that more functions can be added on a case-by-case basis; perhaps by using the PPC process or not, as individual cases require. In any case, it is anticipated that this list would be maintained on an ongoing basis as the language continues to evolve, with more functions being added at every release.

## Rationale

While true syntax additions such as infix operators or control-flow keywords have often been added to perl over the years, there has been little advancement in regular functions - or at least, operators that appear to work as regular functions. Where they have been added, the `use feature` mechanism has been used to enable them. This has two notable downsides:

* It confuses users, by conflating the control-flow or true-syntax named features (such as `try` or `postderef`), with ones that simply add keywords that look and feel like regular functions (such as `fc`).

* Because `feature` is a core-shipped module that is part of the interpreter it cannot be dual-life shipped to CPAN, so newly-added functions cannot easily be provided to older perls.

As there have not been many new regular functions added to the core language itself, the preferred mechanism thus far has been to add functions to core-shipped dual-life modules such as [Scalar::Util](https://metacpan.org/pod/Scalar::Util). This itself is not without its downsides. Although it is possible to import functions lexically, almost no modules do this; instead opting on a package-level import into the caller. This has the effect of making every imported utility function visible from the caller's namespace - which can be especially problematic if the caller is attempting to provide a class with methods:

```perl
package A::Class {
  use List::Util 'sum';
  ...
}

say A::Class->new->sum(1, 2, 3);  # inadvertantly visible method

# This will result in some large number being printed, because the
# List::Util::sum() function will be invoked with four arguments - the
# numerical value of the new instance reference, in addition to the three
# small integers given.

```

A related issue here is that the process of adding new named operators to the perl language is very involved and requires a lot of steps - many updates to the lexer, parser, core list of opcodes, and so on. This creates a high barrier-to-entry for any would-be implementors who wish to provide a new regular function.

## Specification

This document proposes an implementation formed from two main components as part of core perl:

1. A new package namespace, `builtin::`, which is always available in the `perl` interpreter (and thus acts in a similar fashion to the existing `CORE::` and `Internals::` namespaces).

2. A new pragma module, `use builtin`, which lexically imports functions from the builtin namespace into its calling scope.

As named pragma modules are currently implemented by the same file import mechanism as regular modules, this necessitates the creation of a `builtin.pm` file to contain at least part of the implementation - perhaps the `&builtin::import` function itself. This being the case, a third component can be created:

3. A new module `builtin.pm` which can be dual-life shipped as a CPAN distribution called `builtin`, to additionally contain an XS implementation of the provided functions.

This combination of components has the following properties:

* By use of the new package namespace, code written for a sufficiently-new version of perl can already make use of the provided functions by their fully-qualified name:

```perl
say "The reference type of anonymous arrays is ", builtin::reftype([]);
```

* Code can be written which imports these functions to act as regular named functions, in a similar way familiar from utility modules like `Scalar::Util`:

```perl
use builtin 'reftype';
say "The reference type of anonymous arrays is ", reftype([]);
```

* Named functions are imported lexically into the calling block, not symbolically into the calling package. This does differ from the behaviour of traditional utility function modules, but more closely matches the expectations from other pragma modules such as `feature`, `strict` and `warnings`. Overall it is felt this is justified by its lowercase name suggesting its status as a special pragma module.

* Object classes do not inadvertantly expose them all as named methods:

```perl
package A::Class {
  use builtin 'sum';
  ...
}

say A::Class->new->sum(1, 2, 3);  # results in the usual "method not found" exception behaviour
```

Although this document does not wish to fully debate the set of functions actually provided, some initial set is required in order to bootstrap the process and experiment with the mechanism. Rather than proposing any new functions with unclear design, I would recommend sticking simply to copying existing widely-used functions that already ship with core perl in utility modules:

* From `Scalar::Util`, copy `blessed`, `refaddr`, `reftype`, `weaken`, `isweak`.

* From `Internals`, copy `getcwd` (because it is used by some core unit tests, and it would be nice to remove it from the `Internals` namespace where it ought never have been in the first place).

## Backwards Compatibility

This proposal does not introduce any new syntax or behavioural change, aside from a new namespace for functions and a new pragma module. As previous perl versions do not have a `builtin::` namespace nor a `use builtin` pragma module, no existing code will be written expecting to make use of them. Thus there is not expected to be any compability concerns.

As a related note, by creating a dual-life distribution containing the `builtin.pm` pragma module along with a polyfill implementation of any functions it ought to contain, this can be shipped to CPAN in order to allow code written using this new mechanism to be at least partly supported by older perl versions. Because the pragma still works as a regular module, code written using the `use builtin ...` syntax would work as intended on older versions of perl if the dual-life `builtin` distribution is installed.

## Security Implications

There are none anticipated security implications of the builtin function mechanism itself. However, individual functions that are added will have to be considered individually.

## Examples

## Prototype Implementation

None yet.

## Future Scope

As this proposal does not go into a full list of what specific functions might be provided by the mechanism, this is the main area to address in future. As a suggestion, I would make the following comments:

* Most of the `Scalar::Util` functions should be candidates

* Most of the `List::Util` functions that do not act as higher-order functionals can probably be included. This would be functions like `sum`, `max`, `pairs`, `uniq`, `head`, etc. The higher-order functionals such as `reduce` or its specialisations like `first` and `any` would not be candidates, because of their "block-like function as first argument" parsing behaviour at compiletime.

* Some of the `POSIX` functions that act abstractly as in-memory data utilities, such as `ceil` and `floor`. I would not recommend adding the bulk of the operating system interaction functions from POSIX.

* There are other PPCs or Pre-PPC discussions that suggest adding new named functions that would be good candidates for this module. They can be considered on their own merit, by reference to this PPC. At time of writing this may include new functions to handle core-supported boolean types (PPC 0008) or the new module-loading function (PPC 0006).

* Once a stable set of functions is defined, consider creating version-numbered bundles in a similar theme to those provided by `feature.pm`:

```perl
use builtin ':5.40';  # imports all those functions defined by perl v5.40
```

* Once version-numbered bundles exist, consider whether the main `use VERSION` syntax should also enable them; i.e.

```perl
use v5.40;  # Does this imply  use builtin ':5.40'; ?
```

## Rejected Ideas

### Multiple Namespaces

An initial discussion had been to consider giving multiple namespaces for these functions to live in, such as `string::` or `ref::`. That was eventually rejected as being overly complex, and inviting a huge number of new functions. By sticking to a single namespace for all regular functions, we apply a certain amount of constraining pressure to limit the number of such functions that are provided.

## Open Issues

### Package Name

What is the package name these functions are provided in?

The discussion above used `builtin::`. Other proposed suggestions include `function::` or `std::`.

### Pragma Module Name

What is the module name for the lexical-import pragma?

The discussion above used `use builtin`, to match the package name. Technically it does not have to match the package name. In particular, if during implementation or initial use it is found to be problematic that the name does match, the import module could use a plural form of the same word; as

```perl
use builtins qw( function names here );
```

### Version Numbering

How to version number the `builtin` pragma module?

This becomes an especially interesting when considering the dual-life module distribution provided as a polyfill for older perls.

A case can be made that its version number should match the version of perl itself for which it provides polyfill functions. Thus, code could write:

```perl
use builtin v5.40;
# and now all the builtin:: functions from perl v5.40 are available
```

This does initially seem attractive, until one considers the possibility that a dual-life implementation of these polyfills might contain bugs that require later revisions to fix. How would the version numbering of the dual-life distribution reflect the fact that the implementation contains a bugfix on top of these?

### Polyfill for Unavailable Semantics

While not directly related to the question of how to provide builtin functions to new perls, by offering to provide a dual-life module on CPAN as a polyfill for older perl releases, the question arises on what to do if older perls cannot support the semantics of a provided function. The current suggestion of copying existing functions out of places like `Scalar::Util` does not cause this problem, but when we consider some of the additional PPCs we run into some more complex edge-cases.

For example, PPC 0008 proposes adding new functions `true` and `false` to provide real language-level boolean values, and an `isbool` predicate function to enquire whether a given value has boolean intention. The first two can be easily provided on older perls, but polyfilling this latter function is not possible, because the question of "does this value have boolean intention?" is not a meaningful question to ask on such perls. There are a number of possible ways to handle this situation:

1. Refuse to import the symbol - `use builtin 'isbool'` would fail at compiletime

2. Import, but refuse to invoke the function - `if( isbool $x ) { ... }` would throw an exception

3. Give a meaningful but inaccurate answer - `isbool $x` would always return false, as the concept of "boolean intention" does not exist here

Each of these could be argued as the correct behaviour. While it is not directly a question this PPC needs to answer, it is at least acknowledged that some added polyfill functions would have this question, and it would be encouraged that all polyfilled functions should attempt to act as consistently as reasonably possible in this regard.

## Copyright

Copyright (C) 2021, Paul Evans.

This document and code and documentation within it may be used, redistributed and/or modified under the same terms as Perl itself.
