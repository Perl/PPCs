# New API for conversion from UTF-8 to UV

## Preamble

    Author:  K. H. Williamson <khw@cpan.org>
    ID:      RFC-0022
    Status:  Draft


## Abstract

Introduce an API more convenient to use safely

## Motivation

The existing API requires disambiguation between a NUL character and malformed
UTF-8, which callers tend to not do, and the caller besides has to take
extra steps to correctly implement current best practices for dealing with
malformed UTF-8.

## Rationale

This API has no ambiguity between success and failure, and always returns 
based on best practices. Other proposals were discarded in the pre-RFC process:
[Pre-RFC: New C API for converting from UTF-8 to code point](http://nntp.perl.org/group/perl.perl5.porters/264207)

## Specification

I think it best to start with just the functions that will be applicable in
almost all situations.  The pod for these is

### Common use functions

=for apidoc      |bool|utf8_to_cp       |const char * s|const char * e|UV *cp|Size_t * len
=for apidoc_item |bool|utf8_to_cp_flags |const char * s|const char * e|UV *cp|Size_t * len|U32 flags
=for apidoc_item |bool|utf8_to_cp_nowarn|const char * s|const char * e|UV *cp|Size_t * len

These each translate UTF-8 into UTF-32 (or UTF-64 on platforms where a UV is 64
bits long), returning <true> if the operation succeeded without problems; and
false otherwise.  (On EBCIDIC platforms, the input is considered to be
UTF-EBCDIC rather than UTF-8.)

They differ in how they handle problematic input.

More precisely, they each calculate the first code point represented by the
sequence of bytes bounded by <*s> .. <(*e) - 1>, interpreted as UTF-8.
<e> must be strictly greather than <s>.  The functions croak otherwise.

Since UTF-8 is a variable length encoding, the number of bytes examined also
varies.  The algorithm is to first look at <*s>.  If that represents a full
UTF-8 character, no other byte is examined, and the code point is calculated
from just it.  If more bytes are required to represent a complete UTF-8
character, <(*s) + 1> is examined as well.  If that isn't enough, <(*s) + 2> is
examined next, and so forth, up through <(*e) - 1>, quitting as soon as a
complete character is found.

If the input is valid, <true> is returned; the calculated code point is
stored in <*cp>; and the number of UTF-8 bytes it consumes from <s> is
stored in <*len>.

If the input is in some way problematic, <false> is returned; the Unicode
REPLACEMENT CHARACTER is stored in <*cp>; and the number of UTF-8 bytes
consumed from <s> is stored in <*len>.  This number will always be > 0, and
is the correct number to add to <s> to continue examining the input for
subsequent characters.  This behavior follows current best practices for
handling problematic UTF-8, which have evolved based on experiences with
security attacks using malformations.

UTF-8 syntax allows for the expression of 31 bit (30 in UTF-EBCDIC) code points.
But Unicode has deemed all those above U+10FFFF to be illegal, and reserves
certain others for internal use.  Perl predates Unicode, and by default
considers the code points above Unicode to be valid, as well as the reserved
ones.  Furthermore, Perl has created an extended UTF-8 (and UTF-EBCDIC) that
allows for the expression of code points up to 64 bits wide.

<utf8_to_cp> and <utf8_to_cp_nowarn> presume all the Perl extensions and
reserved code points are valid.  They are suitable for use when there is no
need to worry about those being an issue.  The other functions allow the caller
to control more precisely what inputs are considered valid.

Most callers of these functions will want to either croak on malformed input or
forge ahead (using the returned REPLACEMENT CHARACTER), depending on the
circumstances of the call.  In the latter case, the results won't be "correct",
but will be as good as possible, and would be apparent to anyone examining the
outputs, as the REPLACEMENT CHARACTER has no use in Unicode other than to
signify such an error.

A typical use case for forging ahead no matter what, would be:

 while (s < e) {
     UV cp;
     Size_t len;

     (void) utf8_to_cp(s, e, &cp, &len);
     // handle the code point

     s += len;
 }

And if the caller wants to do something different when the input isn't valid:

 while (s < e) {
     UV cp;
     Size_t len;

     if (utf8_to_cp(s, e, &cp, &len) {
        // handle the code point
     }
     else {
        // croak or recover from the error
     }

     s += len;
 }

C<utf8_to_cp> will raise warnings for malformations if UTF8 warnings are
enabled;  C<utf8_to_cp_nowarn> will never raise a warning.

Neither C<utf8_to_cp> nor C<utf8_to_cp_nowarn> will raise warnings for
the extended set of code points accepted by Perl.

If (unlikely) you need the Unicode versus native code point on an EBCDIC
machine, modify the success case in the above example to: 

     if (utf8_to_cp(s, e, &cp, &len) {
        cp = NATIVE_TO_UNICODE(cp);
     }

(<REPLACEMENT CHARACTER> is the same in both character sets, so the failure
case doesn't need to be modified.)

<utf8_to_cp_flags> can be used to more finely control what classes of UTF-8
return <true> versus <false>, and what classes raise warnings when encountered.

First, to turn off warnings for any of the problems that <utf8_to_cp> would
warn on, include the flag <UTF8_NO_WARN> in the <flags> parameter.

Next, there are three classes of code points that are unquestionably accepted
by <utf8_to_cp> and <utf8_to_cp_nowarn> that with this function can
independently raise a warning when encountered, and/or be disallowed, returning
<false> with the code point set to REPLACEMENT CHARACTER.

One class is the surrogate characters, withdrawn from general use by Unicode
(and now reserved by it for aiding in specifying a different encoding, UTF-16).
Including the flags UTF8_DISALLOW_SURROGATE and/or UTF8_WARN_SURROGATE in the
<flags> parameter will respectively cause the function to return <false> when
one is encountered and/or to raise a warning, if either <utf8> or <surrogate>
warnings are enabled.

The seecond class, is comprised of the non-character code points.  These are
reserved by Unicode mostly for use as sentinels.  UTF8_DISALLOW_NONCHAR and
UT8_WARN_NONCHAR control the behavior when these are encountered.  Either the
<utf8> or <nonchar> warning must be enabled for warnings to actually be raised.

Third, are the code points above the Unicode-allowed maximum of U+10FFFF.
These are called "supers" in Perl terminology.  UTF8_DISALLOW_SUPER and
UTF8_WARN_SUPER control the behavior for these, with the warnings categories
<utf8> or <non_unicode>.  Since it is a Perl-designed extension to express code
points using more than 31 bits, it is much less likely that a program written
in another language would understand these than the smaller ones, which were
acceptable until withdrawn from use by Unicode.  Therefore, you can allow/not
warn on the smaller ones, while disallowing and/or warning on the Perl-extended
ones.  Use UTF8_DISALLOW_PERL_EXTENDED and UTF8_WARN_PERL_EXTENDED.  The
warnings categories are the same for all supers: <utf8> and <non_unicode>.

To disallow and/or warn on all three categories at once, the shortcut flags
UTF8_DISALLOW_ILLEGAL_INTERCHANGE and/or UTF8_WARN_ILLEGAL_INTERCHANGE can be
used.  Because Unicode changed its guidance on non-character code points in its
Corregindum 9, there are UTF8_DISALLOW_ILLEGAL_C9_INTERCHANGE and
UTF8_WARN_ILLEGAL_C9_INTERCHANGE, which make illegal just the surrogates and
above Unicode code points.

When a code point is disallowed, <false> is returned and <*cp> is set to
REPLACEMENT CHARACTER.

The same basic logic is used for this function.  For example,

 while (s < e) {
     UV cp;
     Size_t len;

     if (utf8_to_cp_flags(s, e, &cp, &len, (UTF8_DISALLOW_ILLEGAL_INTERCHANGE
                                          | UTF8_WARN_ILLEGAL_INTERCHANGE))
     {
        // handle the code point
     }
     else {
        // croak or recover from the error
     }

     s += len;
 }

### utf8_to_cp_errors

=for apidoc|bool|utf8_to_cp_errors|const char * s|const char * e|UV *cp|Size_t * len|U32 flags|U32 * errors

This function is like <utf8_to_cp_flags> but is for code that needs to know
what the precise malformation(s) are when an error is found.  If you also need
to know the generated warning messages, use <utf8_to_cp_msgs> instead.

It is like <utf8_to_cp_flags> but it takes an extra parameter placed after
all the others, <errors>.  If this parameter is 0, this function behaves
identically to <utf8_to_cp_flags>.  Otherwise, <errors> should be a pointer
to a <U32> variable, which this function sets to indicate any errors found.
Upon return, if <*errors> is 0, there were no errors found.  Otherwise,
<*errors> is the bit-wise <OR> of the bits described in the list below.  Some
of these bits will be set if a malformation is found, even if the input
<flags> parameter indicates that the given malformation is allowed; those
exceptions are noted:

#### <UTF8_GOT_PERL_EXTENDED>

The input sequence is not standard UTF-8, but a Perl extension.  This bit is
set only if the input <flags> parameter contains either the
<UTF8_DISALLOW_PERL_EXTENDED> or the <UTF8_WARN_PERL_EXTENDED> flags.

#### <UTF8_GOT_CONTINUATION>

The input sequence was malformed in that the first byte was a UTF-8
continuation byte.

#### <UTF8_GOT_LONG>

The input sequence was malformed in that there is some other sequence that
evaluates to the same code point, but that sequence is shorter than this one.

Until Unicode 3.1, it was legal for programs to accept this malformation, but
it was discovered that this created security issues.

#### <UTF8_GOT_NONCHAR>

The code point represented by the input UTF-8 sequence is for a Unicode
non-character code point.
This bit is set only if the input <flags> parameter contains either the
<UTF8_DISALLOW_NONCHAR> or the <UTF8_WARN_NONCHAR> flags.

#### <UTF8_GOT_NON_CONTINUATION>

The input sequence was malformed in that a non-continuation type byte was found
in a position where only a continuation type one should be.  See also
<UTF8_GOT_SHORT>.

#### <UTF8_GOT_OVERFLOW>

The input sequence was malformed in that it is for a code point that is not
representable in the number of bits available in an IV on the current platform.

#### <UTF8_GOT_SUPER>

The input sequence was malformed in that it is for a non-Unicode code point;
that is, one above the legal Unicode maximum.
This bit is set only if the input <flags> parameter contains either the
<UTF8_DISALLOW_SUPER> or the <UTF8_WARN_SUPER> flags.

#### <UTF8_GOT_SURROGATE>

The input sequence was malformed in that it is for a Unicode UTF-16 surrogate
code point.  This bit is set only if the input <flags> parameter contains
either the <UTF8_DISALLOW_SURROGATE> or the <UTF8_WARN_SURROGATE> flags.

#### <UTF8_GOT_SHORT>

The input sequence was malformed in that the bytes from <s> to <e-1> did not
form a complete code point.  In other words, the input is for a partial
character sequence.

<UTF8_GOT_SHORT> and <UTF8_GOT_NON_CONTINUATION> both indicate a too short
sequence.  The difference is that <UTF8_GOT_NON_CONTINUATION> indicates always
that there is an error, while <UTF8_GOT_SHORT> means that an incomplete
sequence was looked at.   If no other flags are present, it means that the
sequence was valid as far as it went.  Depending on the application, this could
mean one of three things:

1) The <e> parameter passed in was too small, and the function was prevented from
examining all the necessary bytes.

2) The buffer being looked at is based on reading data, and the data received so
far stopped in the middle of a character, so that the next read will
read the remainder of this character.  (It is up to the caller to deal with the
split bytes somehow.)

3) This is a real error, and the partial sequence is all we're going to get.

The main use for this function would be for code that is draining a buffer
being filled by some other code.  <UTF8_GOT_SHORT> would then be a temporary
situation, and not a real error.

### utf8_to_cp_msgs

=for apidoc|bool|utf8_to_cp_msgs  |const char * s|const char * e|UV *cp|Size_t * len|U32 flags|U32 *errors|HV *msgs

This function is for use by code where the standard behavior for problematic
code points just doesn't work.  Encode is the only module currently known to use
this.

This function is like <utf8_to_cp_errors>, except there is an extra final
parameter, <msgs>, that is an HV*.  If this parameter is 0, the function
behaves identically to <utf8_to_cp_errors>, storing a bit for each
malformation found in any non-NULL <*errors>.

Otherwise, the function doesn't warn on any malformation.  Instead, it creates
an AV to store information about each found malformation, setting <*msgs> to
it.  The elements of the array are ordered so that the first message that would
have been displayed is in the 0th element, and so on.  Each element is a hash
with three key-value pairs, as follows:

<text>

The text of the message that would have been output, as a <SVpv>.

<warn_categories>

The warning category (or categories) packed into a SVuv>.

<flag>

A single flag bit associated with this message, in a <SVuv>.
The bit corresponds to some bit in the <*errors> return value,
such as <UTF8_GOT_LONG>.

The caller, of course, is responsible for freeing any returned AV.

## Backwards Compatibility

This is a new interface which I will add support to in Devel::PPPort.  After
that is done, I will issue pull requests to the relatively few places in CPAN
that use the current API.

## Security Implications

This aims to remove any existing security flaws, and to make it easy to fix any
new ones that may come along, without any XS changes.

## Examples

See the Specification

## Prototype Implementation

None; this is just an alternative API to the existing implementation

## Future Scope

None

## Rejected Ideas

See [Pre-RFC: New C API for converting from UTF-8 to code point](http://nntp.perl.org/group/perl.perl5.porters/264207)

## Open Issues

Are these the right names for the functions?  The most obvious name,
<utf8_to_uv> was used in earlier perls.  Other names for the functions that
replaced it got more convoluted.  I'd like to get back to a simple name, but I
believe I can't reuse any that have ever been in use.

## Copyright

Copyright (C) 2022, K. H. Williamson

This document and code and documentation within it may be used, redistributed and/or modified under the same terms as Perl itself.

