# User-specific `@INC` Directories

## Preamble

    Author:  Paul Evans <PEVANS>
    Sponsor:
    ID:      TODO
    Status:  Exploratory

## Abstract

Make it easier for non-root users to install and use perl modules in a standard location within their home directory.

## Motivation

It is common for users to want to install additional Perl modules in a non-systemwide location - either for their own use perhaps in their home directory, or for a specific project or application in a location dedicated to that purpose. By default, the perl interpreter provided by the operating system does not look for modules in any paths writable by a non-root user, so initially this would seem to be impossible. 

Configuration options do exist in both the interpreter and installer tools to select other locations - typically some path within the user's `$HOME` or local to a specific project - thus allowing additional modules to be installed without them appearing system-wide or needing permission from the root user to do so. However, the out-of-the-box new user experience is poor, often requiring things like edits to `.profile` or other configuration changes.

This proposal aims to make the common case of users wishing to install their own modules work much smoother, especially for novice users, by reducing the number of hoops they must jump through in order to achieve this.

## Background Research

Before launching into a full specification of how we aim to solve these problems in Perl, we will take a brief detour into some other language ecosystems that are doing similar things in this area. This overall problem area is not unique to Perl and is something others have looked into before. We can gain some insight into looking for common patterns.

### Python

Python's approach is mostly spelled out by [PEP-0370](https://peps.python.org/pep-0370/). It specifies that, by default, the interpreter will look in a given path within the user's home directory when loading modules. Additionally, it specifies a command-line flag for its module installer tool (called `pip`) to request that it write into this per-user path.

```
$ pip install --user ...
```

The specific path that is chosen is, naturally, dependent on the type of operating system. The path also encodes the version number of the Python interpreter, so that modules built for one specific version don't collide with another.

On UNIX-like systems (e.g. Linux) it is within the `.local` directory of the user's home, at

```
$HOME/.local/lib/pythonNN/site-packages/
```

While MacOS X is mostly considered UNIX-like, it picks a different path there

```
$HOME/Library/Python/NN/lib/python/site-packages/
```

On Windows it uses the `%APPDATA%` environment variable, using

```
%APPDATA%\Python\PythonNN\site-packages\
```

where here "NN" encodes the version number of the Python interpreter (e.g. "python26" for version 2.6, or "python311" for version 3.11). 

### Ruby

Ruby follows a similar theme to Python. Its installer tool, `gem`, also has a command-line flag:

```
$ gem install --user ...
```

Likewise, the installation path lives in the user's home directory, typically for UNIX-like platforms, including Mac OS X

```
$HOME/.gem/ruby/$VER/
```

On Windows it goes in `%USERPROFILE%` directly, not using `%APPDATA%`

```
%USERPROFILE%\.local\share\gem\ruby\$VER\
```

where here "$VER" contains the Ruby version number, e.g. "4.0.0".

### Raku

Raku's installation tool, `zef`, appears to work in the opposite way. It picks a per-user installation path by default, but has an option to ask it not to (presumably, so the root user can make system-wide installations).

Modules get installed in the `$HOME/.raku/` directory within the user's home, though I haven't managed to find out any further detail on the structure within this location; whether it is subdivided by version numbers, or other such.

### npm

Node seems to take a different approach than these other languages. Rather than storing modules in a per-user location, it instead defaults to using a `node_modules` directory in the current path, installing modules in what is presumed to be the top-level of a particular project. While there is a "global" flag for its installer tool, this tries to act on a system-wide basis, and thus would presumably have the same "must be root" permissions problems that other systems try to solve with a per-user location.

```
npm install -g ...
```

On UNIX platforms this picks the system-wide installation directory, but on Windows it uses a path in `%APPDATA%`:

```
%APPDATA%\npm\node_modules
```

I haven't been able to find any description of a standard layout for what goes inside here, whether it encodes an interpreter version number. Nor any suggestion of what a per-user path might look like.

## Rationale

(explain why the (following) proposed solution will solve it)

## Specification

A solution to this problem requires work in two areas - the perl interpreter when loading modules, and the module installer tools when installing them.

### The `perl` Interpreter

Primarily, if this mechanism is in use, the perl interpreter needs to look for the presence of extra directories within the user's home, and if present, insert them in the right place in the `@INC` list. In particular, they should be inserted *after* any project-specific overrides that might be found in the `$PERL5LIB` environment variable, but before any system-wide ones that appear in configuration defaults.

On a UNIX-like platform, the relevant paths added would be:

```
$HOME/.local/lib/perl/$PERLVER
$HOME/.local/lib/perl
```

((TODO: For Mac OS X we should work out whether to just follow the same UNIX-like path, or to use something in `$HOME/Library/Perl/...`))

On Windows, the relevant paths added would be:

```
%APPDATA%\Perl\$PERLVER
%APPDATA%\Perl
```

((TODO: I have little knowledge and experience of how non-UNIX platforms ought to behave. Folks more familiar with Mac OS X, Windows, VMS, and others may wish to adjust these suggestions or make new ones.))

The phrase "if this mechanism is in use" is important here, as for various reasons, we should not always apply it. Two new command-line flags would allow users to force the decision one way or the other:

```
$ perl --home-libdir -MFoo -e '...'

$ perl --no-home-libdir -MFoo -e '...'
```

In the absence of either of these overrides, a compile-time default shall apply. This would be set at `Configure` time, by either of

```
$ ./Configure -Dusehomelibdir

$ ./Configure -Uusehomelibdir
```

It is important that tools such as `perlbrew` make sure to disable this option when building a perl to be installed directly in a user's home directory, because in this situation its `@INC` dir paths are already writable by the installing user, and the shared home directory mechanism should not be used.

((TODO: Some note about a transition-period way to set defaults, while this mechanism is still new. Maybe an env var? But that's still annoying and not much improved on the current setup. meh.))

Finally, for security reasons, this mechanism should only be used when `sudo`-like conditions are not present; i.e. when the real and effective user IDs are the same (`getuid() == geteuid()`) and likewise for group ID. This prevents software that runs in an elevated context from having its behaviour overridden by any modules the invoking user has placed in their home directory.

Special consideration needs to be put into how to ensure that child processes created by calling `exec()` on `$^X` also set the correct module include directories. The preëxisting mechanisms of setting environment variables were inherited by child processes. As a test, the following example should print the same directory list twice:

```perl
say join ",", @INC;

exec( $^X, '-E', 'say join ",", @INC' );
```

It is possible this could be acheived with the existing environment variables, but care should be taken to ensure it creates the list in the right order, and does not duplicate any elements.

### Module Installer Tools

While changes to the perl interpreter and any tooling bundled with it are within the scope of this PPC process, we cannot strictly require any changes in behaviour of third-party installer tools that are available from CPAN. From their perspective, what follows here is merely a recommendation, a suggestion on what they ought to do to properly interact with the proposed perl behaviour.

Where possible, tools that are primarily controlled by commandline arguments should accept a `--user` option to request that a module install (or other operation) is performed within the standard user home directory. For backward compatibility, existing tools should probably not change their default behaviour, though newly-written tools might decide to adopt this as a default. If this option becomes default, such tooling should also recognise a `--no-user` or similar option to disable it.

To avoid duplication of code and allow installer tools to easily determine the correct location, it is likely that perl would provide this information in a standard way - perhaps from the `Config` or `ExtUtils::InstallPaths` modules. This would need to take account of things like the `./Configure -Dusehomelibdir` option, so that it continues to do the right thing on `perlbrew` installs or other situations.

The core-provided `CPAN.pm` and its associated `cpan` commandline tool don't really operate in this way, so it remains an open design question to determine the best way for users to request a home-directory install.

## Backwards Compatibility

From the perspective of code that needs to handle modules being present in a variety of locations, this proposal does not introduce anything new. Already the perl interpreter allows customisation of module install paths, so installers and loaders have to be able to cope with various situations.

## Security Implications

At first glance this proposal might seem similar to Perl's previous "dot in INC" behaviour, with the same security issues it causes. However, a key difference here is that the added paths are not relative to the current working directory, but fixed within the user's home. Thus there is not the same trouble of perl picking up extra directories from possibly-untrusted locations such as recently downloaded and unpacked archives. If an attacker is in a position to write changes to files directly within the user's home directory, then all manner of attacks become possible and this proposal does not introduce any new angles that would not have been present already.

Additionally, the real-vs-effective ID check ensures that security-sensitive software that runs in an elevated context is not influenced by files the user may have provided to alter its behaviour.

## Examples

## Prototype Implementation

## Future Scope

## Rejected Ideas

## Open Issues

 * How to do `cpan --user Module`, or the `cpan> install Module` in its shell.

 * How do installer tools communicate this request to EU:MM or M:B-style control scripts inside the module distribution?

 * Should Mac OS X use the `Library/` path name, or just follow the same as other UNIX-likes?

 * What should home-directory paths look like on VMS?

 * What should build/install systems do with executable scripts intended for `$PATH`?

 * What should build/install systems do with manpages? Almost certainly these are not of interest to user-specific homedirectory installs and we should probably just skip them; but we should consider it first.

 * Allow project-specific library directories too

## Copyright

Copyright (C) 2026, Paul Evans.

This document and code and documentation within it may be used, redistributed and/or modified under the same terms as Perl itself.
