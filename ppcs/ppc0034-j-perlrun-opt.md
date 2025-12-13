# A short and accurate title

## Preamble

    Author:  Philipp Böschen <perl@philipp.boeschen.me>
    Sponsor:
    ID:      0034
    Status:  Draft

## Abstract

This PPC suggests the implementation of a new perlrun option `-j` that works in conjunction with `-p` or `-n` to automatically feed the stdin that is parsed into `JSON::PP::decode_json($_);`
A working example can be found at: https://github.com/Perl/perl5/pull/22718

## Motivation

Ultimately this change would make Perl another step further to being extremely useful in adhoc sysadmin tasks, a role that is currently covered by, usually, the `jq` tool.
I personally have found that `jq`, while useful for quick extracts, becomes very cumbersome if you have to do conditional logic, which is where `perl -jnle` would shine since usually a simple `grep` or `map` can work wonders on getting good data quickly.
JSON has largely become ubiquitous with modern API handling, so I feel like it's not overly specific to let Perl "natively" handle it.

As the [PRs](https://github.com/Perl/perl5/pull/22718) discussion shows, there are of course alternatives, but none of them seem to be as accessible as a `-j` flag, especially if we are thinking of where `jq` is often applied:
- On cloud-init scripts where Perl is present but not really considered
- In a bare bones debugging environment it gets often pulled in, Perl would already be there

## Rationale

I have tested the aformentioned [PR](https://github.com/Perl/perl5/pull/22718) for a while at work and privately and found it quite useful in place of `jq`. It has one less dependency on my existing debugging kit and performs reasonably well.

Any other proposed solutions always come with the catch that now I have to bring said solution along with me to the debugging target, be it a source filter or just a whole different tool written in Perl itself.

There are some thoughts on the "-p" interaction for things that ingest JSON and don't want to produce a new JSON from it, I personally am unsure of what the ideal here would be right now.

## Specification

This would be an extension of https://perldoc.perl.org/perlrun.
```pod
=item -j

causes Perl to consume the lines that "-n" provides to be fed into C<JSON::PP::decode_json($_);> to give access to line delimited JSON feeds as Perl hashes directly. This will fail with the usual errors from C<JSON::PP::decode_json> if the line that is read is not valid JSON.
When used with both "-n" and -p", "-j" calls C<print JSON::PP::encode_json($_);> to properly print out JSON again.
```
## Backwards Compatibility

This change can be backported into all versions that have a working `JSON::PP` module.

## Security Implications

This would open up a similar security exposure as any perl scripts have that use `JSON::PP`. Of course if this feature gets widely adopted into server bootstraps the criticality of `JSON::PP` becomes more vital over time.

## Examples

Structured logging is the obvious use case for this feature so you can for example force `journald` into line by line JSON mode and then parse it like so to list all root processes logs:
```
sudo journalctl -f -o json | ./perl -njle 'if ( $_->{"_GID"} == 0 ) {print $_->{"MESSAGE"}}'
```

This would work similarly for most structured logging applications that have somewhat all agreed on JSON being the format of choice.

A quick example from parsing audit logs for Hashicorp Vault:

```
tail -f /var/log/vault/audit.log | perl -jnle 'if ( $_->{request}{client_id} eq "some_client" ) { print "$_->{request}{operation},$_->{request}{path}\n" }'
update,auth/token/create
...
```

## Prototype Implementation

https://github.com/Perl/perl5/pull/22718 implements the full PPC while also passing all tests.

## Future Scope

There should be no additions would be that `JSON::PP` starts supporting different things.
Some work could be done on formatting the output in various ways, like supporting pretty printing.

## Rejected Ideas

As mentioned above there has been some discussion around writing a custom Perl application/module that does similar things or using a source filter. All of these solutions fall a bit short on the portability of being in the main Perl runtime which is the allure of this flag since it would remove one more dependency from running systems.

## Open Issues


## Copyright

Copyright (C) 2024, Philipp Böschen

This document and code and documentation within it may be used, redistributed and/or modified under the same terms as Perl itself.

