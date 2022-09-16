This repository is for *Requests For Comments* - proposals to change the Perl language.

Right now, we're [trialling the process](docs/process.md). If you would like to submit a feature request, please [email an *elevator pitch*](mailto:perl5-porters@perl.org) - a short message with 4 paragraphs:

1. Here is a problem
2. Here is the syntax that I'm proposing
3. Here are the benefits of this
4. Here are potential problems

and if a "paragraph" is 1 sentence, great.

That will be enough to make it obvious whether the idea is

0) actually a **bug** - a change we'd also consider back porting to maintenance releases (so should be a opened as [*Report a Perl 5 bug*](https://github.com/Perl/perl5/issues/new/choose))
0) worth drafting an RFC for
0) better on CPAN first
0) "nothing stops you putting it on CPAN, but it doesn't seem viable"

You don't need to subscribe to the list to send an idea (or see updates). By keeping all discussion there during the trial, we can see if the process works as hoped, and fix the parts that don't.

Please **don't** submit ideas as *issues* or *PRs* on this repository. (We can disable issues, but not PRs). Please follow the instructions above.

These files describe the process we are trialling

* [motivation.md](docs/motivation.md) - why do we want to do something
* [process.md](docs/process.md) - the initial version of the process
* [template.md](docs/template.md) - the RFC template
* [future.md](docs/future.md) - how we see the process evolving
* [others.md](docs/others.md) - what others do (or did), and what we can learn from them

## RFCs by status

### Shipped

*present in a shipped release of Perl*

| ID | Title | Version |
|----|-------|---------|
|0001|[Multiple-alias syntax for foreach](rfcs/rfc0001.md)|5.36.0|
|0004|[Deferred block syntax](rfcs/rfc0004.md)|5.36.0|
|0008|[Stable SV Boolean Type](rfcs/rfc0008.md)|5.36.0|
|0009|[Namespace for Builtin Functions](rfcs/rfc0009.md)|5.36.0|
|0011|[Command-line flag for slurping](rfcs/rfc0011.md)|5.36.0|
|0016|[A built-in for getting index-and-value pairs from a list](rfcs/rfc0016.md)|5.36.0|
|0017|[built-in functions for checking the original type of a scalar](rfcs/rfc0017.md)|5.36.0|

### Implemented

*docs, tests and implementation*

| ID | Title |
|----|-------|

### Accepted

*we think that this plan looks viable*

| ID | Title |
|----|-------|
|0006|[New module loading function](rfcs/rfc0006.md)|
|0012|[Configure option for not including taint support](rfcs/rfc0012.md)|
|0013|[Support overloaded objects in join(), substr() builtins](rfcs/rfc0013.md)|

### Provisional

*we think that this idea is worth implementing*

| ID | Title |
|----|-------|

<!-- If some RFCs are "Deferred", they should be in a second table here -->

### Exploratory

*we think that this idea is worth exploring*

| ID | Title |
|----|-------|
|0002|Re-implement number to string conversion|
|0007|source::encoding to declare encoding of source|

### Rejected

*we don't think that this is a good idea - here's why...*

| ID | Title |
|----|-------|
|0003|[Issue a warning "-np better written as -p"](rfcs/rfc0003.md)|
|0005|Everything slices|
