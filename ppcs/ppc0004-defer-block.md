# Deferred block syntax

## Preämble

    Author:  Paul Evans <PEVANS>
    Sponsor:
    ID:      0004
    Status:  Implemented

## Abstract

Add a new control-flow syntax written as `defer { BLOCK }` which enqueues a block of code for later execution, when control leaves its surrounding scope for whatever reason.

## Motivation

Sometimes a piece of code performs some sort of "setup" operation, that requires a corresponding "teardown" to happen at the end of its task. Control flow such as exception throwing, or loop controls, means that it cannot always be written in a simple style such as

```perl
{
  setup();
  operation();
  teardown();
}
```

as in this case, if the `operation` function throws an exception the teardown does not take place. Traditional solutions to this often rely on allocating a lexical variable and storing an instance of some object type with a `DESTROY` method, so this runs the required code as a result of object destruction caused by variable cleanup. There are no in-core modules to automate this process, but the CPAN module `Scope::Guard` is among the more popular choices.

It would be nice to offer a native core syntax for this common behaviour. A simple `defer { BLOCK }` syntax removes from the user any requirement to think about storing the guard object in a lexical variable, or worry about making sure it really does get released at the right time.

This syntax has already been implemented as a CPAN module, [`Syntax::Keyword::Defer`](https://metacpan.org/pod/Syntax::Keyword::Defer). This PPC formalizes an attempt implement the same in core.

## Rationale

((TODO - I admit I'm not very clear on how to split my wording between the Motivation section, and this one))

The name "defer" comes from a collection of other languages. Near-identical syntax is provided by Swift, Zig, Jai, Nim and Odin. Go does define a `defer` keyword that operates on a single statement, though its version defers until the end of the containing function, not just a single lexical block. I did consider this difference, but ended up deciding that a purely-lexical scoped nature is cleaner and more "Perlish", overriding the concerns that it differs from Go.

## Specification

A new lexically-scoped feature bit, requested as

```perl
use feature 'defer';
```

enables new syntax, spelled as

```perl
defer { BLOCK }
```

This syntax stands alone as a full statement; much as e.g. a `while` loop does. The deferred block may contain one or multiple statements.

When the `defer` statement is encountered during regular code execution, nothing immediately happens. The contents of the block are stored by the perl interpreter, enqueued to be invoked at some later time, when control exits the block this `defer` statement is contained within.

If multiple `defer` statements appear within the same block, the are eventually executed in LIFO order; that is, the most recently encountered is the first one to run:

```perl
{
  setup1();
  defer { say "This runs second"; teardown1(); }

  setup2();
  defer { say "This runs first"; teardown2(); }
}
```

`defer` statements are only "activated" if the flow of control actually encounters the line they appear on (as compared to `END` phaser blocks which are activated the moment they have been parsed by the compiler). If the `defer` statement is never reached, then its deferred code will not be invoked:

```perl
while(1) {
  defer { say "This will happen"; }
  last;
  defer { say "This will *NOT* happen"; }
}
```

((TODO: It is currently not explicitly documented, but naturally falls out of the current implementation of both the CPAN and the in-core versions, that the same LIFO stack that implements `defer` also implements other things such as `local` modifications; for example:

```perl
our $var = 1;
{
  defer { say "This prints 1: $var" }

  local $var = 2;
  defer { say "This prints 2: $var" }
}
```

((I have no strong thoughts yet on whether to specify and document - and thus guarantee - this coïncidence, or leave it unsaid.))

Non-local control flow (`next/last/redo`, `goto`, `return`) is not permitted to cross the boundary of the `defer` block

```perl
foreach my $item (@things) {
  defer { last; }  ## This is NOT permitted
}
```

Attempts to do this will throw an exception, likely worded something about

```
Attempting to 'last' out of a 'defer' block is not permitted at FILE line LINE.
```

Of course, nonlocal control flow is permitted entirely *within* the `defer` block; we only prohibit control flow that attempts to leave the block:

```perl
{
  defer {
    foreach my $i ( 1 .. 10 ) {
      last if $i == 5;  ## This is permitted
    }
  }
}
```

The one exception to this (pardon the pun) is that exceptions thrown from within the `defer` block propagate out to the caller.

For example, in

```perl
sub f {
  defer { die "This is thrown\n"; }
}

f();
```

the caller will see the exception being thrown in the same way as if it appeared in a regular (non-`defer`red) block.

It is currently unspecified what happens if a `defer` block throws an exception while unwinding the stack because of an exception.

```perl
{
  defer { die "Exception A\n"; }
  die "Exception B\n";
}
```

In this case, the caller will definitely see *an* exception thrown from the code, but there is currently no guarantee whether that will be A, B, or some third kind of thing that is a combination of the two. This is intentionally left as scope for future expansion; at such time perl ever gains true object-like core exceptions, then it can be represented by some kind of "double exception" condition.

## Backwards Compatibility

The new syntax `defer { BLOCK }` is guarded by a lexical feature guard. A static analyser that is not aware of that feature guard would get confused into thinking this is indirect call syntax; though this is no worse than basically any other feature-guarded keyword that controls a block (e.g. `try/catch`).

For easy compatibility on older Perl versions, the CPAN implementation already mentioned above can be used. If this PPC succeeds at adding it as core syntax, a `Feature::Compat::Defer` module can be created for it in the same style as [`Feature::Compat::Try`](https://metacpan.org/pod/Feature::Compat::Try).

## Security Implications

None foreseen.

## Examples

A couple of small code examples are quoted above. Further, from the docs of `Syntax::Keyword::Defer`:

```perl
use feature 'defer';

{
  my $dbh = DBI->connect( ... ) or die "Cannot connect";
  defer { $dbh->disconnect; }

  my $sth = $dbh->prepare( ... ) or die "Cannot prepare";
  defer { $sth->finish; }

  ...
}
```

## Prototype Implementation

CPAN module `Syntax::Keyword::Defer` as already mentioned.

In addition, I have a mostly-complete branch of bleadperl (somewhat behind since I haven't updated it for the 5.34 release yet) at

  https://github.com/leonerd/perl5/tree/defer

I'm not happy with the way I implemented it yet (don't look at how I abused an SVOP to store the deferred optree) - it needs much tidying up and fixing for the various things I learned while writing the CPAN module version.

## Future Scope

If this PPC becomes implemented, it naturally follows to enquire whether the same mechanism that powers it could be used to add a `finally` clause to the `try/catch` syntax added in Perl 5.34. This remains an open question: while it doesn't any new ability, is the added expressive power and familiarity some users will have enough to justify there now being two ways to write the same thing?

Additionally, once `defer` exists and if core exceptions ever gain some sort of metadata or object-like representation beyond simple strings, then the double-exception case mentioned above can be addressed.

## Rejected Ideas

On the subject of naming, this was originally called `LEAVE {}`, which was rejected because its semantics don't match the Raku language feature of the same name. It was then renamed to `FINALLY {}` where it was implemented on CPAN. This has been rejected too on grounds that it's too similar to the proposed `try/catch/finally`, and it shouldn't be a SHOUTY PHASER BLOCK. All the SHOUTY PHASER BLOCKS are declarations, activated by their mere presence at compiletime, whereas `defer {}` is a statement which only takes effect if dynamic execution actually encounters it. The p5p mailing list and the CPAN module's RT queue both contain various other rejected naming ideas, such as UNWIND, UNSTACK, CLEANUP, to name three. 

Another rejected idea is that of conditional enqueue:

```perl
defer if (EXPR) { BLOCK }
```

as this adds quite a bit of complication to the grammar, for little benefit. As currently the grammar requires a brace-delimited block to immediately follow the `defer` keyword, it is possible that other ideas - such as this one - could be considered at a later date however.

## Open Issues

Design-wise I don't feel there are any remaining unresolved questions.

Implementation-wise the code still requires some work to finish it off; it is not yet in a merge-ready state.

## Copyright

Copyright (C) 2021, Paul Evans.

This document and code and documentation within it may be used, redistributed and/or modified under the same terms as Perl itself.
