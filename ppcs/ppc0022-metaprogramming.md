# Metaprogramming API

## Preamble

    Author:  Paul Evans <PEVANS>
    Sponsor:
    ID:      0022
    Status:  Exploratory

## Abstract

A coherent API for metaprogramming tasks; that is, code which is aware of its own code structure and can inspect or manipulate it.

## Motivation

As a dynamic language, programs written in Perl have the ability to be aware of the structure of their own code. Functions, packages, variables, and other constructions can be obtained or manipulated by name. While the techniques can certainly be over-used, it is a useful and necessary ability of what it means to be a "dynamic language", and where appropriate these kinds of techniques should be supported and encouraged.

Originally, Perl provided some abilities in the form of operations on glob references. Gradually over time, authors have required more abilities than this and increasingly have used a variety of CPAN modules to extend these abilities. As many of the operations provided ought to be considered fundamental language abilities for a dynamic language, it would be better if these were provided by core Perl itself.

This PPC aims to provide a set of API functions to cover many of these abilities. Similar to the case of PPC 0009 having added a namespace for general "builtin" functions, it is not intended that this PPC covers a total and complete list; but instead sets out an initial set of functions. It is expected that more would be added over time on an individual basis, as the need arises. In particular, this PPC does not address any additional API that may be needed by the new object system currently being developed under the `use feature 'class'` flag.

## Rationale

(explain why the (following) proposed solution will solve it)

## Specification

As this PPC adds a number of functions all on an overall-similar theme, they are all collected into a new toplevel namespace. For now this namespace is called `meta::`; though as per the rest of the details in this PPC that is currently very much up for debate.

Objects that are returned by these functions will be instances of some named package within the `meta::` namespace, though we do not yet specify exactly what they will be. Furthermore we make no statement about what actual container type of objects those actually are. Users should not attempt to interact with these objects except via the API specified here.

The get-like functions come in pairs, consisting of a function named `get_...` and one called `can_...`. If the target item exists, then in either case it is returned. The behaviour differs when the item does not exist. The `get` variant will throw an exception, whereas the `can` variant will return `undef`. This allows the caller to decide which behaviour is best for the use-case.

It is currently still an undecided question on what ought to be the correct behaviour for `add`-like functions when asked to create something that already exists, or the `remove`-like functions when asked to remove something that doesn't exist. It may be solved by something similar to above, where there are two differently-named functions that differ in that case. As yet there aren't any particularly compelling solutions.

### Functions to operate on Packages

These are functions to create, manipulate, and inspect packages themselves.

#### `get_package`

```perl
$metapackage = meta::get_package($name);
```

Returns a meta-package object instance to represent the named package, if such a package exists. If not an exception is thrown.

[[TODO: Decide whether "all packages exist", and thus this can never fail]]

#### `add_package`

```perl
$metapackage = meta::add_package($name);
```

Creates a new package of the given name, and returns a meta-package object instance to represent it.

#### `list_packages`

```perl
@metapackages = meta::list_packages($basename);
```

Returns a list of sub-packages within the given base package name, each as a meta-package object instance.

### Functions to operate on References

Some less common functions that may still be useful in some situations. These functions can be used to obtain meta-programming object instances in ways other than walking the symbol table for given names.

#### `for_reference`

```perl
$metavar = meta::for_reference($ref);
```

Given a reference to a package variable or subroutine, returns a meta-object instance to represent it. If `$varref` is not a SCALAR, ARRAY, HASH or CODE reference, an exception is thrown.

Anonymous subroutines should be supported; at least in order to obtain a meta-object to represent them so that methods like `set_subname` can be called on them.

*Note:* It is currently unspecified what happens if the reference refers to a lexical or anonymous variable, rather than a package one. This PPC does not specify any useful behaviour for such instances, but it's possible that future abilities may be added that become useful on lexical variables or anonymous containers.

### Methods on Meta-Package Objects

These methods all operate on a meta-package object obtained by one of the above functions.

#### `name`

```perl
$name = $metapackage->name;
```

Returns the fully-qualified name of the package as a plain string. This will match the string passed to `meta::get_package` used to obtain it, for example.

#### `can_symbol`, `get_symbol`

```perl
$metasymbol = $metapackage->can_symbol($name);

$metasymbol = $metapackage->get_symbol($name);
```

Returns a meta-symbol object instance to represent the symbol within the package, if such a symbol exists. If not either `undef` is returned or an exception is thrown.

The exact type of symbol that is returned depends on the first character of the `$name` parameter - i.e. the sigil. Sigils of `*`, `$`, `@`, `%` and `&` respectively will return a meta-symbol representing a glob itself, or the scalar, array, hash or code slots from it.

(Direct access to the scalar, array, hash or code slots of the glob is provided as a shortcut convenience for the common cases. Access to other more exotic slots of the glob such as filehandles or formats will have to be performed via the metaglob object itself).

#### `add_symbol`

```perl
$metasymbol = $metapackage->add_symbol($name, $value);
```

Creates a new named symbol. If a value is passed it must be a reference to an item of the compatible type. If no initialisation is provided for variables, an empty variable is created. If no initialisation is provided for a subroutine then a forward declaration is created which has no body yet.

#### `remove_symbol`

```perl
$metapackage->remove_symbol($name);
```

Removes a symbol. Nothing is returned.

#### `list_symbols`

```perl
@metasymbols = $metapackage->list_symbols($filter)
```

Returns a list of symbols in the given package, in no particular order. (And in particular, we make no guarantees about the order of the results as compared to the order of the source code which declared it).

TODO: Define what the filters look like. Need to be able to select scalar/array/hash/code as well as sub-stashes. Actually, do we need sub-stashes if we have `list_packages`?

### Methods on Meta-Symbol Objects

Several of the above methods return instances of a meta-symbol object. In practice, any meta-symbol object will represent either a glob (when using the "*" sigil), variable (when using the "$@%" sigils), or a subroutine (when using the "&"). Each of these cases is described further below. The following methods are available on all kinds.

#### `basename`

```perl
$name = $metasymbol->basename;
```

Returns the final part of the symbol's fully qualified name; the name within the package after the final `::` but **excluding** the leading sigil.

#### `sigil`

```perl
$sigil = $metasymbol->sigil;
```

Returns the single-character sigil that leads the symbol's name. This will be one of "*", "$", "@", "%" for variables or "&" for subroutines.

#### `package`

```perl
$metapackage = $metasymbol->package;
```

Returns the meta-package object instance representing the package to which this symbol belongs.

#### `name`

```perl
$name = $metasymbol->name;
```

Returns the fully-qualified name for this symbol, including its sigil and package name. This would be equivalent to

```perl
$name = $metasymbol->sigil . join "::", $metasymbol->package->name, $metasymbol->basename;
```

#### `is_`*\**

```perl
$bool = $metasymbol->is_glob;
$bool = $metasymbol->is_scalar;
$bool = $metasymbol->is_array;
$bool = $metasymbol->is_hash;
$bool = $metasymbol->is_subroutine;
```

Returns true in each of the four cases where the object represents the given type of symbol, or false in the other three cases.

These methods should not be considered an exhaustive selection. There may be object types to represent other glob slots such as filehandles, formats, and so on. As more types are defined, more methods would be added to distinguish them.

### Methods on Meta-Glob Objects

These methods are available on any meta-symbol object that represents a glob - i.e. one whose sigil is "*".

#### `can_scalar`, `get_scalar`

```perl
$metasymbol = $metaglob->can_scalar;
$metasymbol = $metaglob->get_scalar;
```

Returns the meta-symbol representing the scalar slot of the glob. If the slot has not been allocated, returns `undef` or throws an exception.

Similar methods would exist for the other slot types; `array`, `hash`, `code` at least.

The various methods on the package *could* be implemented indirectly via these objects. For example something such as the following:

```perl
method can_symbol($name)
{
    my ($sigil, $basename) = m/^([\*\$\@\%\&])(.*)$/;
    my $glob = $self->can_glob($basename) or return undef;
    $sigil eq "\*" and return $glob;
    $sigil eq "\$" and return $glob->can_scalar;
    $sigil eq "\@" and return $glob->can_array;
    ...
}
```

#### `add_scalar`

```perl
$metasymbol = $metaglob->add_scalar($ref);
```

Creates a scalar variable and adds it to the glob, returning a meta-symbol object to represent it. Optionally a reference to an existing item of the compatible type can be provided.

Likewise, similar would exist for the other slot types.

#### `remove_scalar`

Removes the scalar slot from the glob.

Likewise, similar would exist for the other slot types.

### Methods on Meta-Variable Objects

These methods are available on any meta-symbol object that represents a variable - i.e. one whose sigil is "$", "@" or "%".

#### `value`

```perl
$scalar = $metavar->value;
@array  = $metavar->value;
%hash   = $metavar->value;

$count  = scalar $metavar->value;
```

Returns the current value of the variable. Behaves as the variable itself would in either scalar or list context. Scalars variables yield their value in both scalar and list context. Arrays and hashes yield the count of items contained within when in scalar context, or a list of their (keys and) values in list context.

#### `reference`

```perl
$ref = $metavar->reference;
```

Returns a SCALAR, ARRAY or HASH reference to the variable itself. This allows the caller to potentially modify the variable, or perform operations on it other than simply listing all its values with `value`.

```perl
foreach my $k ( keys $metavar->reference->%* ) { ... }

push $metavar->reference->@*, @more_items;
```

### Methods on Meta-Subroutine Objects

These methods are available on any meta-symbol object that represents a subroutine - i.e. one whose sigil is "&".

#### `subname`

```perl
$name = $metasub->subname;
```

For regular subroutines obtained from the symbol table, returns the fully-qualified name of the function including its package name but *excluding* its leading "&" sigil.

For previously-anonymous subroutines that have since been given a name by `set_subname`, returns that name.

(Inspired by `Sub::Util::subname` and to some extent `Sub::Identity`)

#### `set_subname`

```perl
$metasub->set_subname($name);
  # returns $metasub
```

Sets the subname for a previously-anonymous subroutine, such as one generated by `sub { ... }` syntax.

TODO: Specify what happens if you try to do this on a regular symbol table subroutine. We probably shouldn't allow it.

(Inspired by `Sub::Util::set_subname`)

#### `prototype`

```perl
$prototype = $metasub->prototype;
```

Returns the function prototype string, or `undef` if there is none set.

#### `set_prototype`

```perl
$metasub->set_prototype($prototype)
  # returns $metasub
```

Sets a new value for the prototype string of the function, or removes it if the given value is `undef`.

Returns the meta-subroutine object instance itself so as to be useful for chaining.

(Inspired by `Sub::Util::set_prototype`)

## Backwards Compatibility

As this PPC only adds new functions in a new namespace there are not expected to be any large concerns about backwards compatibility. The individual abilities added are all things that perl already has the ability to do - either natively, or with the help of existing core or CPAN modules - so these functions are not able to put the interpreter into any new states that could not already be encountered.

Additionally, the entire API specified by this PPC should be possible to implement on top of existing perl core using only an additional XS module. As such it would be easily possible to provide it as a dual-life XS module on CPAN for the benefit of at least a few previous versions of perl.

## Security Implications

The point of all these functions is to provide an ability for a program to operate on its own internals. With that ability comes the inherent danger of operating on "the wrong bits"; parts that were not intended. This becomes especially important to consider when user-supplied values are being used as parts of the arguments to operate on. Care should be taken to ensure that arguments cannot be constructed so as to escape the intended areas that the program wishes to expose for manipulation. Some comparison with the dot-in-@INC issue may be relevant.

Some of these issues can be mitigated by careful use of the metapackage objects. For instance, a program wishing to expose symbols to user code only within a certain package can obtain the meta-package object for that package using a fixed static string, and then know that any symbol-access methods invoked on that object can only operate on symbols within that one package.

## Examples

(Should find some bigger use-cases and write them here.)

## Prototype Implementation

Several CPAN modules provide inspiration for abilities and function names:

* [`Package::Stash`](https://metacpan.org/pod/Package::Stash)

* [`Object::Pad::MOP::Class`](https://metacpan.org/pod/Object::Pad::MOP::Class) and related.

## Future Scope

* Currently this PPC makes no consideration for whatever extra abilities may be needed for `feature 'class'`. It is expected that additional abilities will be wanted that can operate on classes and members of them (fields, methods, etc...). An insipration can be taken from `Object::Pad::MOP::Class` and related. The extensible design of the meta-objects provided here should make those additions easy by simply adding more methods and classes to represent them.

* A tangentially-related topic is that of extended API/semantics for declarative attributes. The existing Perl attributes on variables and subroutines are nowhere near as powerful as the ones that will eventually be required by `feature 'class'` and are already implemented by `Object::Pad`. Whatever extensions are added to core perl to eventually support this will likely have meta-programming accessor methods associated with these object instances too.

* The set of functions provided here is fairly minimal, and no convenient shortcuts are currently considered. As use-cases arise, it may turn out that common patterns, such as reading the value of a given variable in a given package, turn out to be useful. Wrapper functions around these common behaviours could be added on a case-by-case basis.

## Rejected Ideas

* PadWalker's entire API of `peek_my`, `peek_our` and `peek_sub`, `closed_over`, `set_closed_over`.

These do not seem useful at the current time.

* B::Hooks::EndOfScope

* B::CompilerPhase::Hook

It may be useful to add support for these kinds of abilities somewhere in core perl, but this metaprogramming PPC does not seem to be the place for them.

## Open Issues

* Currently it is unspecified what the "add"-type functions will do if an existing item is already found under the proposed new name. Do they warn before replacing? Is it an error and the user must delete the old one first? Should we provide a "replace"-type function, for which it is an error for the original *not* to exist? Or maybe the semantic should be "add-or-replace" silently.

* Relatedly, it is unspecified what the "remove"-type functions will do if asked to remove an item that does not even exist. Should it fail, or should it just silently accept that it doesn't have to do anything?

## Copyright

Copyright (C) 2022-2023, Paul Evans.

This document and code and documentation within it may be used, redistributed and/or modified under the same terms as Perl itself.
