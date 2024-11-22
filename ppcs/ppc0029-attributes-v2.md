# Attributes v2

## Preamble

    Author:  Paul Evans <PEVANS>
    Sponsor:
    ID:      0029
    Status:  Exploratory

## Abstract

Define a new way that attribute definitions can be introduced such that the parser can invoke the third-party custom logic they provide; and additionally permit them to be attached to more types of target than Perl currently allows.

This document is largely concerned with C-level details of how the interpreter works, and the interface it exposes to authors of third-party XS modules. A few small extensions to existing syntax are made by extending existing concepts. Any larger ideas of extending the Perl-visible syntax of attributes are left for future discussion at a later time.

Throughout this document, the term "third-party" means any behaviour provided by additional modules loaded into the interpreter; whether these modules are shipped with the core perl distribution, on CPAN, or privately implemented by other means. This is distinct from true "builtin" behaviours, which are provided by the interpreter itself natively.

## Motivation

The Perl parser allows certain syntax elements, namely the declaration of variables and subroutines, to be annotated with additional information, called "attributes". Each attribute consists of a name and optionally a plain string argument, supplied after a leading colon after the declaration of the name of the element it is attached to.

There are a few of these attribute definitions built into core perl itself; for example:

```perl
my $counter :shared;

sub name :lvalue { ... }
```

The parser supports additional attributes that can be provided by modules, though the situation here has many shortcomings:

* Only lexical or package variables, and subroutines, support attribute syntax. It is not possible to declare attributes on other entities such as packages, or subroutine parameters.

* Attributes on anonymous subroutines are invoked only once at compiletime on the protosub, before any "clonecv" operation that would make a real closure. It cannot perform any behaviour after this time, and does not get to see the real closure CVs that Perl code gets references to.

* Attribute handling is looked up via the package hierarchy expressed in `@ISA`, rather than by lexical scope.

* Third-party attributes are implemented by providing a single `MODIFY_*_ATTRIBUTES` shouty-named method in a package, which is required to understand all the attributes at once. There is no standard mechanism to create individual attributes independently.

Many of these restrictions come from the way that third-party attributes are provided in the perl core, quite apart from its own built-in handling. Built-in attributes get to run their logic much earlier in the parser.

It is the aim of this specification to provide a better and more flexible way for third-party modules to declare and use attributes to allow authors to declare more interesting behaviours on elements of their code. These extensions come in the form of additional elements of the interpreter that are visible to XS modules.

## Rationale

(explain why the (following) proposed solution will solve it)

## Specification

Attributes defined by this specification will be lexical in scope, much like that of a `my` variable, `my sub` function, or any of the builtin function exports provided by the `builtin` module. This provides a clean separation of naming within the code.

An attribute is defined primarily by a C callback function, to be invoked by the parser _as soon as_ it has finished parsing the declaration syntax. This callback function will be passed the target (i.e. the item to which the attribute is being attached), and the optional contents of the parentheses used as an argument to the attribute. There is no interesting return value from this callback function.

An optional second callback function can be provided, for the purpose of parsing the incoming text from the source code into a value that the main apply function will use. Experience with the meta-programming layer in `Object::Pad` suggests this is useful, as often when meta-programming it is inconvenient to have to represent the parameters to an attribute as a flat text string.

Pointers to these two functions are found by a structure that is associated with the name of the new attribute, perhaps defined as the following (though exact type is still an open issue; see below):

```c
struct PerlAttributeDefinition
{
    U32 ver;
    U32 flags;
    SV * (*parse)(pTHX_ const SV *text, void *data);   /* optional, may be NULL */
    void (*apply)(pTHX_ SV *target, SV *attrvalue, void *data);
};
```

Additionally, the `ver` and `flags` fields are added for future flexibility. The `ver` field must be initialised by the code that defines the structure, to indicate what ABI version it is intended for. A suitable value is derived from the Perl version number; for example Perl version 5.41.3 would set the value `(5 << 16) | (41 << 8) | (3)`. In this way, a later version of the interpreter can operate correctly with code expecting earlier structure layouts. It is likely that a macro would be provided to compute the correct value at compile-time.

The `flags` field may contain either of two mutually-exclusive flags. `PERLATTR_NO_VALUE` indicates that the attribute definition does not expect to receive a value at all and asks that it be a parser error if the user supplied anything. Alternatively, `PERLATTR_MUST_VALUE` indicates that a value is required; it shall be a parser error for there not to be a value. If neither flag is supplied then any such value becomes optional - the `attrvalue` parameter may be given a valid SV, or may be `NULL`.

In order to create a new attribute, a third-party module author would create such a C structure containing pointers to the C functions to handle the attribute, and wrap it in some kind of SV - whose type is still yet to be determined (see "Open Issues" below). This would be done by a new API function:

```c
SV *newSVinternal_attribute_definition(const struct PerlAttributeDefinition *def, void *data);
```

(This is named "newSVinternal_..." to point out that it creates an SV intended for internal use by the interpreter, and not directly exposed to Perl code. This naming scheme should be followed by any other later additions of a similar theme).

The extra `data` parameter is not directly used by Perl itself, and is simply passed into any of the callback functions given in the structure itself, in case the module wishes to store extra data there.

This wrapping SV is then placed into the importing scope's lexical pad, using a `:` sigil and the name the attribute should use. It is important to stress that this SV represents the abstract concept of the attribute _in general_, rather than its application to any particular target. As the definition of an attribute itself is not modified or consumed by any particular application of it, a single SV to represent it can be shared and reused by any module that imports it.

When the parser is parsing perl code and finds an attribute declaration attached to some entity, it can immediately inspect the lexical pad (and recurse up to parent scopes if applicable) in an attempt to find one of these lexical definitions. The first one that is found is invoked immediately, before the parser moves on in the source code. If such an attempt does not find a suitable handler, the declaration can be stored using the existing mechanism for a later attempt via the previous implementation.

When attached to a package variable or package-named function, the target can be the entity itself (or maybe indirectly, its GV). When attached to a lexical, it is important to pass in the abstract concept of the target in general, rather than the current item in its scope pad. There is currently no suitable kind of SV to represent this - this remains another interesting open issue.

Additionally, new callsites can be added to the parser to invoke attribute callbacks in new situations that previously were not permitted. It would be a simple extension of the syntax of attributes on lexical variables to additionally have them apply to subroutine signature parameters.

```perl
sub func ( $x, $y :Attr, $z :Attr(Value) ) { ... }
```

Beyond this, this particular specification does not suggest any other places to accept attribute syntax, but this is noted in the "Future Scope" section below.

Note that this specification does not provide a mechanism by which attributes can declare what kinds of targets they are applicable to. Any particular named attribute will be attempted for _any_ kind of target that the parser finds. This is intentional, as it leads to a simpler model both for the interpreter and for human readers of the code, as it means any particular attribute name has at most one definition; its definition does not depend on the type of target to which it is applied. It is the job of the attribute callback itself to enquire what kind of target it has been invoked on, and reject it if necessary - likely with a `croak()` call of some appropriate message.

```c
void apply_attribute_CallMeOnce(pTHX_ SV *target, SV *attrvalue)
{
  if(SvTYPE(target) != SVt_PVCV)
    croak("Can only apply :CallMeOnce to a subroutine");

  ...
}
```

As there is no interesting result returned from the attribute callback function, it must perform whatever work it needs to implement the requested behaviour purely as a side-effect of running it. While a few built-in attributes can be implemented perhaps by adjusting SV flags (such as `:lvalue` simply calling `CvLVALUE_on(cv)`), the majority of interesting use-cases would need to apply some form of extension to the target entity, such as Magic or the newly-proposed "Hooks" mechanism. 

### An Aside On Hooks

While hooks will be described in a separate PPC document, it may be useful to provide a brief outline here in order to better evaluate how this proposal would interact with them. This section is specifically not considered part of this specification, but simply gives a high-level overview of the approximate shape these hooks will likely take.

Hooks are an evolution of (or perhaps a replacement of) the existing mechanism that Perl core currently calls "magic". Magic is defined by a set of trigger functions that can be attached to an SV, that provide additional behaviour and associate value storage to certain activities that happen to the SV it is attached to. Hooks will do the same thing.

The current "magic" mechanism in Perl defines a few trigger functions related to book-keeping the magic mechanism itself (the `clear` and `free` actions), as well as actions related to scalar variables (the `set` and `get` actions, as well as the rarely-used `len`). Magic does not define any actions related to other kinds of variables (arrays, hashes, globs), nor does it provide actions relating to values (such as when they are copied), or to various life-cycle management of subroutines (stored in `SVt_PVCVs`). These are the limitations that "hooks" attempts to overcome.

Hooks will be defined by a similar structure to magic, though having more fields relating to other lifecycle activities. Likely how this will work is that all hooks would start with a standard structure that has a field identifying what kind of hook this particular structure is, as well as the fields common to all such hooks:

```c
struct HookFunctionsCommon {
    U32 ver;     /* identifies an API version for flexibility */
    U32 flags;   /* contains some flags indicating what specific shape of hook this is */
    void (*free)(pTHX_ SV *sv, Hook *hk);
    void (*clear)(pTHX_ SV *sv, Hook *hk);
};
```

Thereafter, specific sets of hook function structures can be defined for various situations, such as viral value-attached hooks, or hooks on different kinds of variables. Two such examples:

```c
struct HookFunctionsValue {
    HOOK_COMMON_FIELDS;
    /* for copying this scalar value-shaped hook from one SV to the next */
    void (*copy)(pTHX_ SV *osv, SV *nsv, Hook *hk);
};

struct HookFunctionsSubroutine {
    HOOK_COMMON_FIELDS;
    /* to handle various stages of parsing the subroutine */
    void (*pre_blockstart)(pTHX_ SV *sv, Hook *hk);
    void (*post_blockend)(pTHX_ SV *sv, Hook *hk);
    ...
    /* to handle calls to this subroutine */
    OP *(*callchecker)(pTHX_ SV *sv, OP *o, Hook *hk);
    ...
};
```

This all said, it is still entirely undecided whether this new mechanism will be named "hooks" and replace magic, or simply be a next iteration of the existing "magic" concept and extend from it. Those are discussions for a different PPC document.

## Backwards Compatibility

The new mechanism proposed here is entirely lexically scoped. Any attributes introduced into a scope will not be visible from outside. As such, it is an entirely opt-in effect that would not cause issues for existing code that is not expecting it.

The only Perl-level change of syntax suggested here is to permit applying attributes to subroutine parameters. The suggested syntax for this is currently a compile-time syntax error in Perl, so this should cause no problem.

```
$ perl -E 'use feature "signatures"; sub f ( $x :Attr ) { }'
Illegal operator following parameter in a subroutine signature at -e line 1, near "( $x :Attr "
syntax error at -e line 1, near "( $x :Attr "
Execution of -e aborted due to compilation errors.
```

## Security Implications

## Examples

As both this proposal and the companion "Hooks" are internal interpreter components that are intended for XS authors to use to provide end-user features, it would perhaps be more useful to consider examples of the kinds of modules that the combination of these two features would permit to be created.

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

```perl
method __add__ :overload(+) ( $other, $swap = false ) { ... }
```

needs to check signature.

## Prototype Implementation

As both these mechanisms would need to be implemented by perl core itself, it is difficult to provide a decent prototype for as an experimental basis.

However, both parts of the mechanism are similar to existing technology currently used in [`Object::Pad`](https://metacpan.org/pod/Object::Pad) for providing third-party extension attributes. In `Object::Pad` the two mechanisms are conflated together - extension hooks can be provided, but must be registered with an attribute name. Source code that requests that attribute then gets that set of hooks attached.

The mechanism proposed here makes the following improvements over the existing approach in `Object::Pad`:

* The concept of named attributes and extension hooks are separated. While each is intended to be used largely by the other, they are independent in case situations require the use of each separately.

* By storing the attribute definitions in the pad, each local scope can give the attribute its own name, allowing renaming in scopes if that would avoid name collisions. In comparison, the ones in `Object::Pad` have a single fixed name, whose visiblity is simply enabled by a lexically-scoped key in the hint hash.

## Future Scope

## Rejected Ideas

* Filtering or flags on attribute definitions to say what kind of target they apply to. By omitting this, a simpler model is provided. It avoids complex questions on how to handle hierarchial classifications, such as that classes are packages, or subroutine parameters are lexical variables. By ensuring that any particular attribute name has at most one definition in any scope, end-users can more easily understand the model provided.

## Open Issues

### SV Type to Represent an Attribute Definition

Attribute definitions need to be SVs in order to live in the lexical pad. There is currently no suitable SV type for this, so one will have to be created. Will it be specific to attributes, or would one type of SV suffice to be shared by various other possible future use-cases of C-level entities visible in the lexical pad, while not intended to be visible to actual perl code as first-class values directly?

### Passing Lexical Target Information

It would first appear that lexical variables and subroutine parameters can be represented by their PADNAME structure, but notably the padname itself does not actually store the pad offset of the named entity. Perhaps the target argument for these should just be the pad offset of the target entity, leaving the invoked callback to find the offset in the compliing pad itself?

This suggests that the actual values passed to specify the target will depend on what kind of target it is. Package-named targets can be passed the target SV itself and its naming GV, whereas lexical targets need to be specified as its pad offset within the currently-compiling pad.

It may make sense to have two different callbacks, one for GV-named targets and one for lexicals

```c
struct PerlAttributeDefinition
{
    ...
    void (*apply_pkg)(pTHX_ SV *target, GV *namegv,      SV *attrvalue, void *data);
    void (*apply_lex)(pTHX_ SV *target, PADOFFSET padix, SV *attrvalue, void *data);
};
```

Or perhaps one function that takes a distinguishing enum and union type to convey the information:

```c
enum PerlAttributeTargetKind {
    ATTRTARGET_PKGSCOPED,
    ATTRTARGET_LEXICAL,
};

union PerlAttributeTarget {
    struct { SV *sv; GV *namegv;      } pkgscoped;
    struct { SV *sv; PADOFFSET padix; } lexical;
};

struct PerlAttributeDefinition
{
    ...
    void (*apply)(pTHX_ 
        enum PerlAttributeTargetKind kind, union PerlAttributeTarget target,
        SV *attrvalue, void *data);
};
```

At present there does not appear to be a distinguishing reason to prefer one style over the other. This remains a question for experimentation with the implementation.

In either case, if the application function wishes to create a new target entity to replace the one it was given, it can do this. In the case of package-named targets, it can replace the appropriate slot in the naming GV. In the case of lexically named targets, it can replace the item in the pad.

Either of these arrangements would make it easy to add other kinds of targets at a later date - for example, perhaps something that operates on optree fragments directly so it can be applied to code fragments like operators or function calls. Whatever later additions are made, it is important to keep in mind that in general an attribute application function may wish to provide a new value for target, so its value must be passed by some kind of mutable storage.

### Split the Pad into Scope + Scratchpad

While not unique to this proposal, the ongoing increase in use of lexical imports in various parts of perl means that pads in typical programs - both at the file and subroutine level - are getting wider, with more named items in there. Because currently the pads are shared with true per-call lexicals and temporaries, this means that any recursive functions consume more space in unnecessary elements. Every recursive depth of function call requires the entire width of the pad to be cloned for each level. The more "static" elements in the pad, the more wasted space because those elements are not going to vary with depth.

It would be a nontrivial undertaking, but it may become useful in terms of memory (and CPU cycles) savings to consider splitting the pad into two separate entities. A "scope" pad would exist just once per file or subroutine, and contain lexically-imported named elements such as imported functions or attribute definitions. Separately a "scratchpad" pad would then only contain the lexical variables and temporary values that are needed once for every depth of recursion of the function. By splitting the pad in such a way, it reduces the amount of work that `pp_entersub` has to perform when recursing into functions that have wide pads because now only the true variables-per-call need to be cloned; the static scope would exist just once for all depths.

### Perl-Level API

The mechanisms described in this document exist entirely at the C level, intended for XS modules to make use of. It should be possible to provide wrappings of most of the relevant parts, at least for simple cases, for use directly by Perl authors without writing C code.

Existing experience with [`XS::Parse::Keyword::FromPerl`](https://metacpan.org/pod/XS::Parse::Keyword::FromPerl) shows that while such an interface could easily be provided, in practice it is unlikely anyone would make use of it as it requires understanding details of its operation in sufficient depth that any author would be just as well served by writing the XS code directly. Therefore no such wrapping is yet proposed by this document but could be added later on, either by a core-provided or CPAN module.

## Copyright

Copyright (C) 2024, Paul Evans.

This document and code and documentation within it may be used, redistributed and/or modified under the same terms as Perl itself.
