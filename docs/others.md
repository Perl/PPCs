(June 2021)

# What can we learn from others.

This is a process for features, not for bug fixes. Obviously, it's
unclear where the line is, and do we need it for trivial features?
TCL's view was:

> We divide projects into two general categories: bug fixes and feature changes. In general, if a project requires manual entries to be updated then it is a feature change; when in doubt, a project is a feature change

If we don't learn from the mistakes documented in https://www.perl.com/pub/2000/11/perl6rfc.html/ we're doomed to repeat them.

## So what can we learn from others?

"Python Enhancement Proposals" and "TCL Improvement Proposals" diverge from the same source in Aug 2000. I assume that PEPs were the original, and TIPs the fork because I can't find a common third source that both share.

Hence these both predate Distributed Version Control Systems, easy efficient reliable free web hosting, and the common use of code review (or at least effective tools to do this remotely and in different timezones).

TIPs seem to assume a small core team who are equally competent (and maybe not optimised for bad ideas or inexperienced contributors), whilst PEPs assume a Benevolent Dictator For Life (ie Guido) and then are structured to delegate work from him. PEPs seem to rely on a lot of work being done/doable by PEP editors.

The PEP process has evolved (both before and after Guido's retirement).  TIPs resolutely have not, and still mention SourceForge despite TCL now using Fossil. The Python Core Team is about 100, and the PEP process in now implemented in terms of of their steering committee or a delegated person.  The TIP process only involves the core team, and the approvals process is "TYANNOTT: Two Yesses And No No's Or Two Thirds" (two approvals and no objections, or a full vote is needed)

PHP uses RFCs tracked in wiki, and then voting by core team of about 40.

PEPs must have a sponsor - this is someone on the core team.
TIPs must have two people on the core team in favour.

* https://wiki.php.net/rfc/howto
* https://www.python.org/dev/peps/pep-0001/
* https://core.tcl-lang.org/tips/doc/trunk/tip/0.md
* https://core.tcl-lang.org/tips/doc/trunk/tip/2.md

PEPs and TIPs are both used both for code improvements and process improvements. This conflation seems awkward - it requires a special status ("Active") for process related PEPs/TIPs, and many of the sections needed for feature changes are irrelevant for process changes. It also means that the *official* document for the process itself is the change document for the process, not the procedure document. I think this is a bad idea and we should not imitate it. PHP's RFCs only cover code. I like this better, but I think we can "borrow" more of the detail from the Python and TCL approach than from PHP's.
