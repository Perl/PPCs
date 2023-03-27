# Stable SV Boolean Type

## Preamble

    Author:  Paul Evans <PEVANS>
    Sponsor:
    ID:      0008
    Status:  Implemented

## Abstract

Add to regular (defined and non-referential) scalars the ability to track whether they represent a boolean value. Values created from boolean expressions (such as `$x == $y`) will reliably remember this fact, in a way that can be introspected even if the value is stored into a variable and retrieved later.

## Motivation

Language interoperability concerns sometimes lead to situations where it is necessary to represent typed data - such as strings or numbers - in a way that can be reliably distinguished. Elsewhere in Perl there are attempts to add this distinction. This PPC attempts to address how to handle values of a boolean type; values that represent a simple true-or-false nature.

For example, JSON and MsgPack are commonly-encountered serialisation formats that Perl can generate and parse, though in both cases while the formats themselves can represent boolean truth values distinctly from numbers or strings, perl cannot preserve that distinction. Compare this to some other languages popular at the moment:

```
$ python3 -c 'import json; print(json.dumps([1, "1", 1 == 1]))'
[1, "1", true]

$ nodejs -e 'console.log(JSON.stringify([1, "1", 1 == 1]))'
[1,"1",true]

$ perl -MJSON::MaybeUTF8=encode_json_utf8 -E 'say encode_json_utf8([1, "1", 1 == 1])'
[1,"1",1]
```

There is a tendancy for modules to make up this shortfall with workarounds like the `JSON::PP::Boolean` type:

```
$ perl -MJSON::MaybeUTF8=decode_json_utf8 -MData::Dump=pp 
   -E 'say pp(decode_json_utf8($ARGV[0]))' '[1,true]'

[1, bless(do{\(my $o = 1)}, "JSON::PP::Boolean")]
```

Obviously such a solution is specific to JSON encoding and does not apply to, for example, message gateway between JSON and MsgPack, which would require some translation inbetween. A true in-core solution to this problem would have many benefits to interoperability of data handling between these various modules.

((TODO: Add some comments about purely in-core handling as well that don't rely on serialisation))

## Rationale

## Specification

There are two parts to this specification; the lower-level in-core details of the C code implementation that are visible to XS code; and the higher-level Perl-visible details that are made visible to Perl programs.

### C-level Internals

At the C level, SVs will require a macro to test whether they contain a boolean. 

```
SvIsBOOL(sv)
```

The core immortals `PL_sv_yes` and `PL_sv_no` will always respond true to `SvIsBOOL()`, as will any SV initialised by copying from them by using `sv_setsv()` and friends. The upshot here is that the result of boolean-returning ops will be `SvIsBOOL()` and this flag will remain with any copies of that value that get made - either over the arguments stack or stored in lexicals or elements of aggregate structures.

These `SvIsBOOL()` values will still be subject to the usual semantics regarding macros like `SvPV` or `SvIV` - so numerically they will still be 1 or 0, and stringily they will still be "1" and "" (though see the Future Scope section below).

It may be possible to find a spare `SvFLAGS` field for this purpose, at which point there would be macros on a theme similar to the `SvPOK`/`SvIOK`/etc.. set. I would suggest avoiding the single-letter abbreviations of the older style of code - letters aren't so expensive these days that we can't at least spell out the word "BOOL" in full.

Flag bits being rare and expensive, it may turn out that finding a spare flag bit is simply impossible (or at least, undesired). This could be implemented instead by using special pointer values in the `SvPVX()` slots of such booleans. A small change to `sv_setsv()` and friends would be possible to cause it to copy the actual pointer value itself, such that any "boolean" SV can be distinguished by pointer equality of this field, to that of the original "yes" and "no" immortals.

### Perl-level Interface

Once the interpreter can reliably store the concept of "is a boolean" and expose this at least to XS modules, the question remains on how pureperl code can make use of it. How can boolean values be created, and how can we test if a given value is a boolean?

For creating such values, in many cases it is sufficient to simply use any of the existing boolean predicate operators, for example the comparison operators:

```perl
my $sv = 4 > 5;   # The $sv now has SvIsBOOL
```

It may be considered useful to provide two new zero-arity functions, `true` and `false`, to explicitly create these values (which are at least a little more obvious in intent than the equivalent `!!1` and `!!0`). These can be requested from the `builtin` module (PPC 0009):

```perl
use builtin 'true', 'false';

func(1, "1", true);       # Three distinct values
func(0, "0", "", false);  # Four distinct values
```

The question remains on how pureperl code can inspect the "type" of a value to inspect if it is one of these special boolean values. While perl core doesn't have any testing keywords for this, the nearest match to currently available features may be to add a new builtin function:

```perl
use builtin 'isbool';

sub distinguish($x) {
  say !isbool $x ? "not a boolean" :
              $x ? "true"          :
                   "false";
}
```

Whatever solution is considered here should be designed to interact well with any possible larger considerations from the fallout of stronger string-vs-number distinctions, and other ideas currently floating around.

I accept that this is the weakest part of this suggestion so far, and welcome comment here in particular (though *please* first read the "Future Scope" section below).

## Backwards Compatibility

There are not expected to be any backwards compatiblity problems with this proposal. At the interpreter level, extra information is being added to certain SV values, without changing the meaning of any existing information currently stored. Code that is unaware of the new semantics will continue to see existing values unaltered.

Likewise at the Perl syntax level, the only new functionality being added consists of new functions in the `builtin` namespace (PPC 0009).

## Security Implications

Likewise, as this proposal only adds extra information to that being stored by SVs, it is not anticipated there are any security implications of doing so. There is not expected to be a security effect from exposing the fact that some SVs happen to express values of boolean intent.

## Examples

(See embedded code above)

## Prototype Implementation

I made an attempt at using `SvPVX()` tracking of the immortal constants and the `SvIsBOOL()` test macro, at

[https://github.com/leonerd/perl5/tree/stable-bool](github.com/leonerd/perl5/tree/stable-bool)

This was accepted and merged into core perl in time to be released as part of Perl version 5.35.4.

There is no attempt yet to provide the `true`, `false` or `isbool` builtin functions as the builtin function mechanism is not yet available. A workaround for `isbool` is provided by `Scalar::Util`.

## Future Scope

By its nature this proposal fits in the category of "give Perl values more typing information". As such it is likely to fit well with other ideas of a similar nature - such as clearer distinction between strings and numbers, between Unicode text and byte buffers, and other concepts. It may become possible to group these ideas together in a better way, to better solve such questions as how to add predicate-test functions in a way that is visible to Perl syntax.

It may also be possible to provide a lexically-guarded feature that alters the way that boolean values are stringified. It is noted that in Perl currently, a false value stringifies to the empty string, which leads to certain difficulties in debugging output. Perhaps under the `boolean` feature, or perhaps under its own unique feature name, boolean values could stringify differently:

```perl
use feature 'boolean';

my ($x, $y) = (true, false);
print "X=$x Y=$y\n";

__END__
X=true Y=false
```

## Rejected Ideas

One thing that is certainly impossible to do currently is change the stringification of existing boolean values. There is far too much code around - primarily in unit tests - which relies on the current stringification of boolean true and false values, and this cannot be modified. Doing so would break such tests as:

```perl
is( somefunc(), "", 'somefunc returns false' );
```

## Open Issues

## Copyright

Copyright (C) 2021, Paul Evans.

This document and code and documentation within it may be used, redistributed and/or modified under the same terms as Perl itself.
