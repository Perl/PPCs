# Magic v2

## Preamble

    Author:  Paul Evans <PEVANS>
    Sponsor:
    ID:      0035
    Status:  Exploratory

## Abstract

A new internal mechanism that XS modules (and even parts of core perl itself) can use to provide extension behaviours on a variety of variable and other types. This is an evolution of the current `MAGIC` concept, extended to support more situations on more kinds of targets.

## Motivation

The existing `MAGIC` and `MGVTBL` structures are insufficiently extensible to new situations because they lack any embedded type information, or any version numbering by which new abilities can be added in a backwards-compatible way. They apply trigger functions to certain events in the lifetime of scalar variables, but the mechanism does not easily extend to other situations involving other kinds of entity that are not scalar variables.

Perl has many other kinds of entity that are not scalar variables. It would be useful to add magic with specific trigger functions onto array or hash variables, packages, and subroutines. It would also be useful to further extend the ability to attach magic onto things that aren't even (currently) expressed in terms of SVs themselves; such as the compile-time concept of lexical variables, subroutine parameters, fields of object classes, and similar.

## Specification

This specification proposes an entire tree of related structures that store definitions of particular kinds of magic (called the "magic functions" structures), an extension to the per-SV "MAGIC" structure by which these are attached to individual SVs, and a new bit-flag used to indicate the presence of this new system.

Perl5 commit `6d97c86` managed to free a single bit in the `mg_flags` field; it is this bit that we use to indicate that the MAGIC structure refers to version 2 of Magic. There will be a new bit flag for the `mg_flags` field, and a new test macro to check for it;

```c
#define MGf_MGv2   0x80
#define MgIsV2(mg) (mg->mg_flags & MGf_MGv2)
```

If this flag is not set on a `MAGIC`, then no further alterations are made; the structure is an existing (legacy) magic arrangement as defined by previous versions of perl.

If the flag is set, then it indicates that this `MAGIC` structure is in fact of the version 2 variety. This indicates, primarily, that the `mg_virtual` pointer refers to a new Magic v2 functions structure, rather than a legacy v1 version. It also may mean that the `MAGIC` structure is larger with more fields, depending on various other conditions. For backward-compatibility (see also below), it is important that a Magic v2 attachment structure still looks and behaves mostly like a (legacy) v1, because there may be code that still inspects it.

In order to provide function tables for a variety of different use-cases (being attached to different kinds of SVs), the new system will provide an entire tree of related data structures. Each structure would start with a common set of fields, before being extended in different ways to add pointers to trigger functions for various different cases. A field in the common structure prefix will indicate what exact structure type is being used, to ensure the implementation can check for invalid combinations of function structure and SV type. The common structure will also provide just the `free` and `clone` trigger functions, required for basic memory management of any of them:

```c
enum MagicShape {
    MGv2s_BASE,
    MGv2s_SCALARVAR,
    MGv2s_ARRAYVAR,
    MGv2s_HASHVAR,
    ...             /* other shapes can be added here as needed */
};

struct MagicFunctions {
    U32 ver;
    enum MagicShape shape;
    U32 flags;
    void (*free) (pTHX_ SV *sv, MAGIC *mg);
    void (*clone)(pTHX_ SV *osv, MAGIC *omg, SV *nsv, MAGIC *nmg,
        CLONE_PARAMS *param);
};
```

On top of this basic `struct MagicFunctions` we can then define other structures that can contain specific trigger functions for specific situations.

A "scalar variable" variant that provides basically the same functionality as existing v1 magic does now:

```c
struct ScalarVarMagicFunctions {
    ..., /* existing fields, where shape == MGv2s_SCALARVAR */
    void (*pre_get) (pTHX_ SV *sv, MAGIC *mg);
    void (*post_set)(pTHX_ SV *sv, MAGIC *mg);
};
```

Here I've decided to specifically name the triggers "pre_get" and "post_set" to emphasise that they are notifications around existing core perl behaviours, rather than being used to actually implement them. Additionally, the name "post_set" leaves open the possibility - in name at least - of adding a "pre_set" at some future time.

To cover the `clear` special-case that existing v1 magic has on array and hash variables, we can also define two more:

```c
struct ArrayVarMagicFunctions {
    ..., /* existing fields, where shape == MGv2s_ARRAYVAR */
    void (*clear)(pTHX_ SV *sv, MAGIC *mg);
};

struct HashVarMagicFunctions {
    ..., /* existing fields, where shape == MGv2s_HASHVAR */
    void (*clear)(pTHX_ SV *sv, MAGIC *mg);
};
```

As each of these structures is now a separate C-level type and can be distinguished by the value of the `shape` field, it easily allows for future additions to add more trigger functions for various other situations.

This particular document does not make further specification on the exact form of new trigger functions in these structures, nor suggesting any other trigger functions for other situations. Those will be the subjects of particular other documents that can build on top of this one. It may also be the case that these additional use-cases emerge gradually over time, by attempting to create specific modules; rather than being something that can be designed up-front on paper.

In contrast with (legacy) v1 Magic, for Magic v2 core perl will provide a number of accessor macros to help code that operates on these magic structures. Modules (and other core perl code) should use these macros when accessing these data structures.

```c
#define MgFLAGS(mg)  /* lvalue-capable accessor for the U8 flags */

#define MgFUNCS(mg)  /* read accessor for the functions table */

#define MgPRIV(mg)   /* lvalue-capable accessor for the U16 private flags */

#define MgAUXSV(mg)  /* read accessor for the stored aux SV */
```

A number of new API functions will also be added, to manage these new data structures.

```c
MAGIC *sv_magicv2_add(SV *sv, const struct MagicFunctions *funcs, U32 flags,
    SV *auxsv);

MAGIC *sv_magicv2_remove(SV *sv, MAGIC *mg);

MAGIC *sv_magicv2_remove_by_funcs(SV *sv, const struct MagicFunctions *funcs);
```

The first two should be familiar as updated versions of the existing `sv_magicext()` and `sv_unmagicext()`. The third here is a new behaviour, designed by observation that in a lot of cases when magic is being removed, we don't yet have a pointer to the specific magic structure, but we just know we want to remove all the magic with a given functions table. The new `sv_magicv2_remove_by_funcs()` provides that ability, and sets a common naming theme for a few more functions.

```c
MAGIC *sv_magicv2_find_by_funcs(SV *sv, const struct MagicFunctions *funcs);

MAGIC *sv_magicv2_findnext_by_funcs(SV *sv, const struct MagicFunctions *funcs, MAGIC *mg);

MAGIC *sv_magicv2_find_by_auxsv(SV *sv, const struct MagicFunctions *funcs);

MAGIC *sv_magicv2_findnext_by_auxsv(SV *sv, const struct MagicFunctions *funcs, MAGIC *mg);
```

These functions are provided to allow code to look through the set of magic annotations on an SV to find a specific one. These are provided as a nicer alternative to the (legacy) v1 technique of manually walking the `mg->mg_next` pointers; as new code should not be directly accessing the magic structures, outside of the defined macros.

## Backwards Compatibility

In terms of existing CPAN modules that already use (legacy) v1 magic: Nothing relevant here will be removed from perl core. Any `MAGIC` structure whose flags bits don't contain the `MGf_MGv2` bit will continue to have its existing meaning and work correctly. Any given perl program can freely use a mix of modules that use v1 and v2 magic annotations, and all will interact with the perl core correctly.

There is some small concern that modules that use these new structures might confuse any *other* modules loaded in the same process which try to introspect on all the available magic annotations on SVs. This would generally be a rare thing to happen; likely limited to just corner-case debug systems (for example, an existing version of `Devel::MAT` that had not been updated to be aware of the new mechanism). Such inspections are unlikely to affect the correct running of a program, and simply restrict the ability for those debug modules to operate correctly. There is still some useful merit in trying to design the new mechanisms to minimise any possible confusion here; though some amount of confusion may be inevitable.

## Security Implications

There are no directly-forseen implications of extending the internals to permit new modules to be written ways described here. However as with any new internal mechanism, these new abilities may themselves be used to create modules that would have some security impact. It would be a matter of considering the design of any actual module that builds on top of this, rather than the basic mechanism itself.

It may additionally be the case that the new mechanism created here would directly allow the creation of new security-focused modules that can enforce a more flexible and useful security model than is currently available in perl. Again; an issue that can be explored in more detail in other more specific documents.

## Examples

This proposal suggests adding a new C-level API, rather than anything that is directly visible to Perl authors. As such, any examples of it in use would need to consider what kinds of new modules it would allow to be created.

As this proposal (and the companion "attributes v2") are internal interpreter components that are intended for XS authors to use to provide end-user features, it would perhaps be more useful to consider examples of the kinds of modules that the combination of these two features would permit to be created.

For example, a subroutine attribute `:Void` could be created that forces the caller context of any `return` ops within the body of the subroutine, or its implicit end-of-scope expressions, to make them always run in void context.

```perl
use 5.xx;
use Subroutine::Attribute::Void;

sub debug($msg) :Void {
    print STDERR "DEBUG:> $msg\n";
}

print debug("start"), "middle", debug("end");
# Output printed to STDOUT is simply "middle", without the leading or trailing
# "1" return value from the print statement inside the function.
```

This kind of attribute would be easy to implement with some optree adjustment in the magic applied by the attribute, if the magic functions table applied to subroutines had sufficient trigger times within it to allow it to inspect and alter the optree of the subroutine being compiled.

Additional example ideas can be found in CPAN modules, which currently use internal mechanisms of other modules to provide functionality, that would be better provided by core perl itself.

[`Signature::Attribute::Alias`](https://metacpan.org/pod/Signature::Attribute::Alias) is an example of a module that adds an attribute to subroutine signature parameters. If this were built directly for core perl it would use the attributes-v2 mechanism alongside this magic v2 proposal:

```perl
use 5.xx;
use Signature::Attribute::Alias;

sub trim_spaces ($s :Alias) {
    $s =~ s/^\s+//;
    $s =~ s/\s+$//;
}
```

Likewise, [`Object::Pad::FieldAttr::Final`](https://metacpan.org/pod/Object::Pad::FieldAttr::Final) is an example of an attribute on a field of an object class that too would use the attributes v2 and magic v2 features in combination:

```perl
use Object::Pad::FieldAttr::Final;

class Rectangle {
    field $width  :param :reader :Final;
    field $height :param :reader :Final;
    ...
}
```

## Prototype Implementation

As this mechanism would need to be implemented by perl core itself, it is difficult to provide a decent prototype for it as an experimental basis.

However, in the specific case of extended magic trigger functions being available on PADNAME structures used in either subroutine signatures or object class fields, there are some CPAN modules that attempt to provide similar behaviours within their own ecosystems, for other modules to build on top of:

* [`XS::Parse::Sublike`](https://metacpan.org/pod/XS::Parse::Sublike) provides a mechanism for other modules to register attributes to attach to subroutine signature parameter variables. This mechanism combines the syntax-level form of the attribute, and the compile-time effects of operating on the optree of code surrounding the parameter. The latter parts of this may provide useful inspiration here.

* [`Object::Pad`](https://metacpan.org/pod/Object::Pad) provides a mechanism for other modules to register attributes to attach to object fields. Likewise, this combines both the parser syntax and the compile- and run-time effects in one place.

## Future Scope

The ideas outlined in this document are largely a reshaping of existing v1 magic in order to *permit* extension into new abilities and situations in future. Exactly what those new things are can be explored in other documents, on a per-situation basis.

## Rejected Ideas

## Open Issues

### Magic on non-SVs

As outlined above, a key new ability that would be very useful to provide is to attach magic annotations for trigger functions on lexical variables, including in places like subroutine signatures or fields of object classes. In both of those situations, there does not exist one single SV at the time that the attribute is parsed, that could have magic attached to it.

It remains therefore an open question on the best way to attach such a magic annotation onto the compile-time concept of a lexical variable.

## Copyright

Copyright (C) 2026, Paul Evans.

This document and code and documentation within it may be used, redistributed and/or modified under the same terms as Perl itself.
