# Attributes v2 and Hooks

## Preamble

    Author:  Paul Evans <PEVANS>
    Sponsor:
    ID:      TODO
    Status:  Exploratory

## Abstract

Jointly:

* define a new way that attribute definitions can be introduced such that the parser can invoke the third-party custom logic they provide; and additionally

* define more extensive kinds of magic-like structure for attaching custom behaviour onto existing Perl data structures, such as variables, subroutines, and parts thereof.

These two ideas are presented together in one document because, while there could be some valid use cases of each on its own, it is the combination of the two that provides most of the power to create extension modules that can extend the language in new ways.

Throughout this document, the term "third-party" means any behaviour provided by additional modules loaded into the interpreter; whether these modules are shipped with the core perl distribution, on CPAN, or privately implemented by other means. This is distinct from true "builtin" behaviours, which are provided by the interpreter itself natively.

## Motivation

### Attributes

The Perl parser allows certain syntax elements, namely the declaration of variables and subroutines, to be annotated with additional information, called "attributes". Each attribute consists of a name and optionally a plain string argument, supplied after a leading colon after the declaration of the name of the element it is attached to.

There are a few of these attribute definitions built into core perl itself; for example:

```perl
my $counter :shared;

sub name :lvalue { ... }
```

The parser supports additional attributes that can be provided by modules, though the situation here has many shortcomings:

* Only lexical or package variables, and subroutines, support attribute syntax. It is not possible to declare attributes on other entities such as packages, or subroutine parameters.

* Attributes on anonymous subroutines are invoked only at "clonecv"-time; the time when an anonymous function gets turned into a closure with variable captures. It cannot perform any behaviour before this time.

* Attribute handling is looked up via the package heirarchy expressed in `@ISA`, rather than by lexical scope.

* Third-party attributes are implemented by providing a single `MODIFY_*_ATTRIBUTES` shouty-named method in a package, which is required to understand all the attributes at once. There is no standard mechanism to create individual attributes independently.

Many of these restrictions come from the way that third-party attributes are provided in the perl core, quite apart from its own built-in handling. Built-in attributes get to run their logic much earlier in the parser.

It is the aim of this specification to provide a better and more flexible way for third-party modules to declare and use attributes to allow authors to declare more interesting behaviours on elements of their code.

### Hooks

In addition to the limitations of attribute syntax described above, there are very limited options available to the would-be implementors of attributes, as ways to provide the behaviour their attribute would have.

Scalar variables in Perl support a concept called "magic", by which custom behaviour can be attached onto a variable, to be invoked whenever the variable is read from, written to, or goes out of scope. While in theory other kinds of variables (arrays and hashes) do support magic, in practice the magic is not truely aware of the container-like nature of the variable to which they are attached, and cannot provide customisation around things like element iteration or access, without a lot of weird tricks. Magic on any other kind of SV (such as subroutines or stashes, the data stores used to implement packages and classes) is virtually non-existent, limited only to passive storage of additional notation data, and notification of the attached entity's destruction. Other concepts that are not even backed by true SVs, such as the abstract notion of a subroutine parameter or an object field, do not support magic at all.

Many limitations of the core magic system can be seen in core perl itself. While magic is used to implement a lot of the interesting scalar variables (such as `$$` for fetching the current process's PID, or `$&`, the result of the most recent regexp match), it is not used directly for creating things like array or hash variables with custom behaviour, or providing tie on these things. While there _is_ magic involved in these concepts, that magic is often just used as a marker to store extra information to allow special-purpose code in the interpreter to implement those other behaviours. Yet other kinds of magic are used for more tagging of additional data that don't provide additional behaviour on their own.

It would seem that the existing concept of "magic" in the Perl core is simultanously too limited and inflexible to provide most interesting custom behaviours, and at the same time overly elaborate for simply attaching additional data or destruction notification onto non-scalar variables.

It is the aim of this specification to provide a new generation of magic-like behaviour, both for core perl to use for its own purposes in a far more uniform way, and to allow third-party module authors to attach more interesting behaviours when requested; perhaps by using an attribute.

## Rationale

(explain why the (following) proposed solution will solve it)

## Specification

### Attributes

Attributes defined by this specification will be lexical in scope, much like that of a `my` variable, `my sub` function, or any of the builtin function exports provided by the `builtin` module. This provides a clean separation of naming within the code.

An attribute is defined in its base layer, by a single C callback function, to be invoked by the parser _as soon as_ it has finished parsing the declaration syntax. This callback function will be passed the target (i.e. the item to which the attribute is being attached), and the optional contents of the parentheses used as an argument to the attribute. There is no interesting return value from this callback function.

```c
typedef void PerlAttributeCallback(pTHX_ SV *target, SV *attrvalue);
```

In order to create a new attribute, a third-party module author would create such a C function containing whatever behaviour is required, and wrap it in some kind of SV - whose type is still yet to be determined (see "Open Issues" below). This wrapping SV is then placed into the importing scope's lexical pad, using a `:` sigil and the name the attribute should use. It is important to stress that this SV represents the abstract concept of the attribute _in general_, rather than its application to any particular target. As the definition of an attribute itself is not modified or consumed by any particular application of it, a single SV to represent it can be shared and reused by any module that imports it.

When the parser is parsing perl code and finds an attribute declaration attached to some entity, it can immediately inspect the lexical pad (and recurse up to parent scopes if applicable) in an attempt to find one of these lexical definitions. The first one that is found is invoked immediately, before the parser moves on in the source code. If such an attempt does not find a suitable handler, the declaration can be stored using the existing mechanism for a later attempt via the previous implementation.

When attached to a package variable or package-named function, the target can be the entity itself (or maybe indirectly, its GV). When attached to a lexical, it is important to pass in the abstract concept of the target in general, rather than the current item in its scope pad. There is currently no suitable kind of SV to represent this - this remains another interesting open issue.

Additionally, new callsites can be added to the parser to invoke attribute callbacks in new situations that previously were not permitted; such as package declarations or subroutine parameters.

Note that this specification does not provide a mechanism by which attributes can declare what kinds of targets they are applicable to. Any particular named attribute will be attempted for _any_ kind of target that the parser finds. It is the job of the attribute callback itself to enquire what kind of target it has been invoked on, and reject it if necessary - likely with a `croak()` call of some appropriate message.

```c
void attribute_callback_CallMeOnce(pTHX_ SV *target, SV *attrvalue)
{
  if(SvTYPE(target) != SVt_PVCV)
    croak("Can only apply :CallMeOnce to a subroutine");

  ...
}
```

As there is no interesting result returned from the attribute callback function, it must perform whatever work it needs to implement the requested behaviour purely as a side-effect of running it. While a few built-in attributes can be implemented perhaps by adjusting SV flags (such as `:lvalue` simply calling `CvLVALUE_on(cv)`), the majority of interesting use-cases would need to apply some form of extension hook to the target entity. These hooks are described in the other half of this proposal.

### Hooks

(details on how the thing is intended to work)

## Backwards Compatibility

### Attributes

The new mechanism proposed here is entirely lexically scoped. Any attributes introduced into a scope will not be visible from outside. As such, it is an entirely opt-in effect that would not cause issues for existing code that is not expecting it.

### Hooks

## Security Implications

## Examples

As both parts of this proposal are interpreter internal components that are intended for XS authors to use to provide end-user features, it would perhaps be more useful to consider examples of the kinds of modules that the combination of these two features would permit to be created.

For example, a subroutine attribute `:void` could be created that forces the caller context of any `return` ops within the body of the subroutine, or its implicit end-of-scope expressions, to make them always run in void context.

```perl
use Subroutine::Attribute::Void;

sub debug($msg) :void {
    print STDERR "DEBUG:> $msg\n";
}

print debug("start"), "middle", debug("end");
# Output printed to STDOUT is simply "middle", without the leading or trailing
# "1" return value from the print statement inside the function.
```

This kind of attribute would be easy to implement with some optree adjustment in the hook applied by the attribute, but would not be possible to create by the existing mechanisms because they cannot run at the right times.

## Prototype Implementation

As both mechanisms proposed by this document would need to be implemented by perl core itself, it is difficult to provide a decent prototype for as an experimental basis.

However, both parts of the mechanism are similar to existing technology currently used in [`Object::Pad`](https://metacpan.org/pod/Object::Pad) for providing third-party extension attributes. In `Object::Pad` the two mechanisms are conflated together - extension hooks can be provided, but must be registered with an attribute name. Source code that requests that attribute then gets that set of hooks attached.

The mechanism proposed here makes the following improvements over the existing approach in `Object::Pad`:

* The concept of named attributes and extension hooks are separated. While each is intended to be used largely by the other, they are independent in case situations require the use of each separately.

* By storing the attribute definitions in the pad, each local scope can give the attribute its own name, allowing renaming in scopes if that would avoid name collisions. In comparison, the ones in `Object::Pad` have a single fixed name, whose visiblity is simply enabled by a lexically-scoped key in the hint hash.

## Future Scope

## Rejected Ideas

* Filtering or flags on attribute definitions to say what kind of target they apply to. By omitting this, a simpler model is provided. It avoids complex questions on how to handle heirarchial classifications, such as that classes are packages, or subroutine parameters are lexical variables. By ensuring that any particular attribute name has at most one definition in any scope, end-users can more easily understand the model provided.

## Open Issues

### SV Type to Represent an Attribute Definition

Attribute definitions need to be SVs in order to live in the lexical pad. There is currently no suitable SV type for this, so one will have to be created. Will it be specific to attributes, or would one type of SV suffice to be shared by various other possible future use-cases of C-level entities visible in the lexical pad, while not intended to be visible to actual perl code as first-class values directly?

### Passing Attribute Definition SV into the Callback

It may be the case that at import time, extra information is attached to the (unique) SV that gets imported into the caller's scope, which is intended for inspection by the attribute callback. Perhaps it makes sense to pass in the definition's SV as another argument to the callback function.

### Passing Lexical Target Information

It would first appear that lexical variables and subroutine parameters can be represented by their PADNAME structure, but notably the padname itself does not actually store the pad offset of the named entity. Perhaps the target argument for these should just be the pad offset of the target entity, leaving the invoked callback to find the offset in the compliing pad itself?

This might suggest that actually the target should be specified as two - or maybe even three - parameters, some of which would be zero / NULL depending on circumstance.

```c
typedef void PerlAttributeCallback(pTHX_
    SV *target, GV *targetname, /* for package targets, or NULL/NULL */
    PADOFFSET targetix,         /* for lexical targets, or zero */
    SV *attrvalue);
```

But perhaps at that point, it makes more sense to have two different callbacks, one for GV-named targets and one for lexicals?

```c
struct PerlAttributeCallbacks {
  void (*apply_pkg)(pTHX_ SV *target, SV *targetname, SV *attrvalue);
  void (*apply_lex)(pTHX_ PADOFFSET targetix,         SV *attrvalue);
};
```

Or perhaps one function that takes a distinguishing enum and union type to convey the information:

```c
enum PerlAttributeTargetKind {
  ATTRTARGET_PKGSCOPED,
  ATTRTARGET_LEXICAL,
};

union PerlAttributeTarget {
  struct { SV *sv, GV *namegv; } pkgscoped;
  struct { PADOFFSET padix;    } lexical;
}

typedef void PerlAttributeCallback(pTHX_ 
    enum PerlAttributeTargetKind kind, union PerlAttributeTarget target,
    SV *attrvalue);
```

### Split the Pad into Scope + Scratchpad

While not unique to this proposal, the ongoing increase in use of lexical imports in various parts of perl means that pads in typical programs - both at the file and subroutine level - are getting wider, with more named items in there. Because currently the pads are shared with true per-call lexicals and temporaries, this means that any recursive functions consume more space in unnecessary elements. Every recursive depth of function call requires the entire width of the pad to be cloned for each level. The more "static" elements in the pad, the more wasted space because those elements are not going to vary with depth.

It would be a nontrivial undertaking, but it may become useful in terms of memory (and CPU cycles) savings to consider splitting the pad into two separate entities. A "scope" pad would exist just once per file or subroutine, and contain lexically-imported named elements such as imported functions or attribute definitions. Separately a "scratchpad" pad would then only contain the lexical variables and temporary values that are needed once for every depth of recursion of the function. By splitting the pad in such a way, it reduces the amount of work that `pp_entersub` has to perform when recursing into functions that have wide pads because now only the true variables-per-call need to be cloned; the static scope would exist just once for all depths.

### Perl-Level API

The mechanisms described in this document exist entirely at the C level, intended for XS modules to make use of. It should be possible to provide wrappings of most of the relevant parts, at least for simple cases, for use directly by Perl authors without writing C code.

Existing experience with [`XS::Parse::Keyword::FromPerl`](https://metacpan.org/pod/XS::Parse::Keyword::FromPerl) shows that while such an interface could easily be provided, in practice it is unlikely anyone would make use of it as it requires understanding details of its operation in sufficient depth that any author would be just as well served by writing the XS code directly. Therefore no such wrapping is yet proposed by this document but could be added later on, either by a core-provided or CPAN module.

## Copyright

Copyright (C) 2024, Paul Evans.

This document and code and documentation within it may be used, redistributed and/or modified under the same terms as Perl itself.
