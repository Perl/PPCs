# Magic v2

## Preamble

    Author:  Paul Evans <PEVANS>
    Sponsor:
    ID:      0036
    Status:  Exploratory

## Abstract

Extends the "Magic v2" mechanism with a specific Magic type for attaching annotations onto scalar values, in a way that propagates through assignments and calculations.

## Motivation

Perl currently has a scattering of abilities that can be used to annotate extra "side data" around values, some of which it uses for internally special-cased purposes.

* When running in taint-checking mode, values can be annotated as being "tainted", which causes changes of behaviour whenever those values are going to be emitted from the process boundary in places like file names or `exec()` arguments. 

* `version` strings, often called `v`-strings, contain within them a second string buffer, which remembers the original syntax form of the byte values that are stored in the (primary) string buffer. The main utility of this is to remember the presence of underscores in the numerical form of version numbers.

Besides these two special cases, core perl does not have any other facility to attach other data onto values undergoing computation; either for its own purposes or to offer to CPAN modules. In common with the theme for "Magic v2" being to remove these special cases and permit CPAN modules and other code to have the same abilities as are currently special-cased in core perl, this proposal aims to provide this as a generic ability available to any loaded module.

## Rationale

(explain why the (following) proposed solution will solve it)

## Specification

On top of the basic "Magic v2" specification, we add one new functions table shape:

```c
enum MagicShape {
    ...,
    MGv2s_SCALARVALUE,
};

struct ScalarValueMagicFunctions {
    ..., /* existing fields, where shape == MGv2s_SCALARVALUE */
    void (*infect)(pTHX_ SV *ssv, MAGIC *smg, SV *dsv, MAGIC *dmg);
};
```

The `infect` trigger function is used when copying the annotation from one SV into a different one. There are no other functions associated with other lifetime events on this magic shape. Annotations of this magic shape exist largely to store extra data; attached with a call to `sv_magicv2_add` and looked up again later using `sv_magicv2_find_by_funcs`.

As this magic shape is strongly associated with the value stored in the SV, rather than the SV itself, it has quite different rules for copying, clearing, and performing other calculations than other magic shapes have.

* When copying a value, this magic is propagated from the source to destination SV on calls such as `sv_setsv()` by invoking `infect`.

* When clearing a variable, this magic is removed from the variable SV on calls such as `sv_undef()`. A `free` trigger function on the base magic may be invoked if present.

* When calculating a new value as a result of one *or more* input values that have these annotations attached, the `infect` function on each annotation of each such value is invoked to copy it into the resulting destination. It may be that the result ends up with more than one annotation attached to it.

* When a container variable is `local`ised, magic annotations of this shape are not copied into the new temporary container.

These rules come about by considering the likely use-cases of this kind of magic shape; being used to implement security labelling or data-flow analysis. The intention here is that once an annotation is added to data, it is strongly associated with the idea of what that data represents, even as it moves through other variables and storage, is combined with other values, or split apart again. The "Examples" section below outlines a few possible ways this behaviour can be used to good effect.

## Backwards Compatibility

As this proposal adds a new ability to an already-new mechanism (magic v2), there should not exist any backwards-compatibility issues in simply making something new available.

There may however, be some issues that could arise if existing special-case logic were neatend up to use this new mechanism instead of the current special cases. As mentioned in the introduction, perl core currently has a set of special-case logic around v-strings to handle its specific unique requirements. At first glance it would seem like v-string magic could be converted to using Magic v2 in this manner. However, due to the current implementation, v-string annotations don't propagate as much through other calculations as the intended behaviour of viral-value magic would, so this code would have to take account of that. Additionally, there is known to be a lot of existing CPAN code (mostly in the form of hardcoded logic in `Module::Install`) which uses the `B` API to manually walk the magic chain on SVs, looking for magic whose type is `V` and directly obtain the v-string second string buffer from those. If existing core v-string logic were rewritten to use viral v2 magic, it should be careful to ensure that logic in `Module::Install` (and likely other places) keeps working.

## Security Implications

Normally, this section is used to explain around any potential security *problems* that might be introduced by the changes proposed by such a document. As is typical when considering adding new low-level abilities that modules can build on top of, the answer is usually "none foreseen, they would have to be considered by the modules being built".

But a much better answer here is to observe that this proposed mechanism adds a great ability to introduce new *solutions* to security problems. As perhaps was already hinted at in the introduction, the mechanism proposed here acts similar to but is more flexible and powerful than perl's existing "taint" behaviour, which was primarily intended for its security-related purposes. In practice, that mode is seldom used, because of its lack of flexibility. Taint mode just stores a single boolean annotation about any particular value ("is this considered tainted?"), which affects certain entirely-internal functions and operations. In contrast, this proposal would easily permit the creation of a more flexible set of policies; see the "Supertaint" example below.

## Examples

Both of the following examples demonstrate the kind of behaviour that could be provided by layering a module on top of this core-provided ability. In each case, the actual module code is elided, in favour of demonstrating various properties of what it can provide.

### "Supertaint" Security Labelling

Consider a module built on top of viral value magic, which associates a hash with any annotated value, into which can be stored key names to represent a set of labels. The module would provide functions to add or remove labels from perl values, as well as to query what labels are present. (The values are presumably not important here; values could just be boolean true or undef or similar). Such a module can easily be built on top of the `sv_magicv2_add`, `sv_magicv2_remove_by_funcs` and `sv_magicv2_find_by_funcs` API functions. The basic behaviour of `MGv2s_SCALARVALUE` in perl core guarantees that, once a label is added to a given value, any other values derived from it inherit the same label.

```perl
use Supertaint qw(
    supertaint_add_label
    supertaint_remove_label
    supertaint_has_label
);
```

Now consider a typical web application framework that being used to perhaps operate some simple CRUD application. It is an important security constraint to ensure that any data strings that users send to the application cannot be simply interpolated directly into any generated HTML output, or all sorts of security bugs can be created. There have been (and will likely continue to be) countless CVEs and other such issues covering these kinds of things.

In order to check that this security constraint is maintained, the web framework can use this `Supertaint` module to annotate incoming values from the web user, as soon as they are received, to note that they are not safe to be emitted directly in HTML code:

```perl
supertaint_add_label( html_unsafe => $value );
```

Once annotated, the perl interpreter ensures this annotation gets copied and maintained by any derived values, such as being concatenated into longer template strings. If at the time the web framework is about to emit some generated HTML back to the user it finds any of these annotations on the generated string, it should refuse to send that and instead give an error message that the application logic has failed somewhere.

```perl
supertaint_has_label( html_unsafe => $output ) and
    die "Refusing to send unsafe HTML output to the user";
```

This ensures that in this situation, no unsafe output could be generated.

Of course, on its own this behaviour isn't quite sufficient, because often users supply values that *do* need to be returned back in generated pages. For that situation web frameworks often provide some sort of "escaping" function. In this case, that function alone would be the only place where this supertaint label is removed:

```perl
sub html_escape ( $val ) {
   my $html = ...; # some code to HTML-escape the string in $val
   supertaint_remove_label( html_unsafe => $html );
   return $html;
}
```

By careful use of these three "supertaint" functions in the lowest level building blocks of the web framework, it can ensure a security model that it has controlled. It has applied something similar in spirit to perl's own "taint" markings, except that it has decided what the policy should be on when to add or remove the annotations, and what operations should be forbidden if they are found. Furthermore, because these annotations are just one label stored in a hash, a given application could maintain multiple concurrent policies - perhaps to track the safeness of using strings in SQL query strings, external API access, executing command strings, opening filehandles, and so on.

### Lineage Tracking

## Prototype Implementation

As this mechanism would need to be implemented by perl core itself, it is difficult to provide a decent prototype for it as an experimental basis.

## Future Scope

As this proposal creates a set of core perl mechanisms that CPAN modules can use to build on top of, the main future direction beyond here would be the creation of such modules. During that development work, it may turn out that the `struct ScalarValueMagicFunctions` table may need more trigger functions, or some other adjustments in its behaviour, to account for various requirements those modules need.

## Open Issues

## Copyright

Copyright (C) 2026, Paul Evans.

This document and code and documentation within it may be used, redistributed and/or modified under the same terms as Perl itself.
