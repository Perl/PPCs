# Adding Optional Strictures and Warnings Classifications to Perl

## Preamble

    Author:  Curtis "Ovid" Poe <curtis.poe@gmail.com>
    Sponsor:
    ID:      0032
    Status:  Draft

## Abstract

This PPC proposes adding mechanisms for optional strictures and warnings to
Perl that would not be enabled by the standard `use strict` and `use warnings`
pragmas. This allows for introducing new strict checks and warnings without
affecting existing code that uses these pragmas. As a concrete example, this
PPC proposes adding `use strict 'strings'` as the first such optional
stricture, which would prevent implicit stringification of references, similar
to the functionality provided by the CPAN module `stringification`.

## Motivation

The current `strict` and `warnings` pragmas in Perl effectively enable all
available strictures and warnings when used in their recommended forms (`use
strict` and `use warnings`). This creates a challenge when wanting to
introduce new strict checks or warnings that might be too strict or noisy for
general use but valuable for specific cases. Currently, such functionality
must be implemented as separate pragmatic modules (like `indirect`,
`multidimensional`, or `bareword::filehandles`), leading to inconsistent
interfaces and documentation scattered across different locations.

The specific case of implicit reference stringification illustrates this
problem well. When references are implicitly stringified (e.g., `"$ref"`),
they produce output like `ARRAY(0x1234567)` which is rarely useful and often
indicates a programming error. However, this behavior cannot be prohibited
under the current `strict` pragma as it would break existing code.

## Rationale

Adding support for optional strictures and warnings provides several benefits:

1. It allows for introducing new strict checks and warnings without breaking
   backward compatibility
2. It provides a consistent interface for enabling such checks (`use strict
   'feature'` rather than `no feature`)
3. It groups related functionality under the established `strict` and
   `warnings` pragmas
4. It provides a clear path for experimental features to graduate into core
   Perl

The stringification case demonstrates these benefits. Rather than using a
separate `no stringification` pragma, the functionality can be enabled with
`use strict 'strings'`, which is more consistent with existing Perl syntax and
more discoverable through the standard documentation.

## Specification

### Optional Strictures

Add support for optional strict modes that are not enabled by a bare `use
strict` statement. These would be enabled explicitly:

```perl
use strict 'strings';  # Enable reference stringification checks
```

The first such optional stricture would be 'strings', which prevents implicit
stringification of references. When enabled, the following operations would
die with an error if attempted on a non-object reference:

- String interpolation (`"$ref"`)
- String concatenation (`$ref . "foo"`)
- Case conversion (`lc`, `lcfirst`, `uc`, `ucfirst`)
- Pattern matching (`$ref =~ m//`)
- String operations (`split`, `join`)
- Output operations (`print`, `say`)

Error messages would be specific to the operation being performed, e.g.:

```
Attempted to stringify a reference at line X
Attempted to concat a reference at line Y
```

### Optional Warnings

Add a new warnings classification for optional warnings that are not
enabled by `use warnings` or `use warnings 'all'`. These would require
explicit enabling:

```perl
use warnings 'optional';  # Enable all optional warnings
use warnings 'optional::specific';  # Enable specific optional warning
```

## Backwards Compatibility

This proposal is explicitly designed to maintain backward compatibility:

- Existing code using `use strict` or `use warnings` will continue to work
  unchanged
- New strictures and warnings must be explicitly enabled
- The mechanism allows for gradual adoption of new features
- Existing modules like `stringification` can be updated to use the new
  mechanism while maintaining their current interface

## Security Implications

The proposed changes improve security by allowing developers to catch more
potential programming errors. The stringification case in particular can help
prevent information leaks where reference addresses might be accidentally
exposed in output, though that seems unlikely to be a significant security
issue.

## Examples

Here's a trivial example:

```perl
use strict 'all', 'strings';
use warnings;

my $data = [ sensitive_info() ];
log("Processing $data");  # Dies
```

## Prototype Implementation

The [`stringification` module on
CPAN](https://metacpan.org/release/PEVANS/stringification-0.01_004/view/lib/stringification.pm)
provides a prototype for the reference stringification checks. The
implementation can be adapted to work as a core feature, using similar
op-hooking techniques.

## Future Scope

1. Additional optional strictures could include:
   - `strict 'direct'` (current `indirect` module)
   - `strict 'filehandles'` (current `bareword::filehandles` feature)

2. Warning modes for optional strictures:
   - `use strict 'strings=warn'` to warn rather than die

## Rejected Ideas

1. Adding new strictures to the default `strict` mode:
   - Would break backward compatibility
   - Would make `strict` too restrictive for general use

2. Using negative pragmas (`no feature`):
   - Less consistent with Perl's existing pragma system
   - Makes features harder to discover

## Open Issues

1. The PPC specifically mentions "non-object" stringification. Should this be
   more general and apply to all references?

2. How should string overloading be handled? Presumably, stringification
   should be allowed for objects that overload stringification. Hence, the
   earlier restriction on "non-object" stringification, but it's unclear if
   that's the best approach.

3. Should there be a way to enable all optional strictures at once?

   ```perl
   use strict 'optional';  # ?
   ```

See also [Grinnz's feature suggestion on this
matter](https://github.com/Perl/perl5/issues/18543).

## Copyright

Copyright (C) 2025, Curtis "Ovid" Poe.

This document and code and documentation within it may be used, redistributed and/or modified under the same terms as Perl itself.
