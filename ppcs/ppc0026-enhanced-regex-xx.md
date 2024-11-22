# RFC - enhanced regex /xx

## Preamble

    Author:  Karl Williamson <khw@cpan.org>
    ID:      0026
    Status:  Draft

## Abstract

Let programmers improve the readability of regular expression patterns beyond
what is possible now.

## Motivation

Regular expression patterns were designed for concision rather than clarity.

The /x regular expression pattern modifier was created to enable adding
comments and white space to patterns to make them more readable.  It suffers
from not working for bracketed character classes, and silently compiling to
something unintended when the programmer forgets to mark literal white space,
and much worse, literal '#'.  This last silently swallows the rest of the line
that was supposed to be a part of the pattern.

I eventually added /xx to at least allow tabs and blanks inside bracketed
character classes.  This allows a very minor improvement in their readability.
I could not figure out a way to extend this to allow comments and multiple
lines inside such a class without making it even more likely that the pattern
would silently compile to something unintended.  But now, I think this RFC
fixes that.

## Specification

I propose adding a new opt-in feature.  Call it, for now, "feature
`enhanced_re_xx`".  Within its scope, the /xx modifier would change things so
that inside a bracketed character class [...], any vertical space would be
treated as a blank, essentially ignored.  Any unescaped '#' would begin a
comment that ends at the end of the line.

This would change the existing /x behavior where the portion of the line after
the '#' is parsed, looking for a potential pattern terminating delimiter.
Under this feature to terminate a pattern, do so before any '#' on a line.
If an unescaped terminating delimiter is found after a '#' on a line, a warning
would be raised.

And an unescaped '#' within a comment would raise a warning.  So

```
 $a[$i] =~ qr/ [ a-z         # We need to match the lowercase alphabetics
                 ! @ # . *   # And certain punctuation
                 0-9         # And the digits (which can only occur in $a[0])
               ]
             /xx;
```

would warn.

It might be that an unescaped '#' that isn't of the form \s+#\s+ should
warn to catch things like if the above example's second line were just

```
 !@#.*
```

Also, any comments inside [...] would check for an unescaped ']' on the same
line after a '#', and raise a warning if found.  So, something like

```
 $a[$i] =~ qr/ [ a-z  # . * ]
               [ A-Z ]
             /xx;
```

would warn.  Either escape the '#' or the ']' to suppress it, depending on what
your intent was.

I think these would catch essentially all unintended uses of '#' to mean
not-a-comment, but to be taken literally.

I can't think of anything to catch blanks/tabs being unintentionally ignored.

I also propose that unescaped '#' and vertical space inside bracketed character
classes under /xx be deprecated.  /xx has been available only since 5.26;
there's not a huge amount of code that uses it.  After the deprecation cycle,
the feature could become automatic, not opt-in, and /xx would have the new
meaning.  

Note there is no change to plain /x.

Copyright (C) 2022 Karl Williamson

This document and code and documentation within it may be used, redistributed
and/or modified under the same terms as Perl itself.

