# Namespacing proposal

This document outlines a proposal to improve the namespacing of
compilation units in OCaml. It aims to solve a number of known problems
whilst keeping full backwards compatibility.

## Motivation

Compilation units in OCaml currently live in a flat namespace. This
means that all the compilation units in a program must have a unique
name. Since this is a very burdensome requirement a number of
work-arounds have arisen:

1. The `-pack` compiler option can be used to pack up multiple
compilation units into a single unit. The names of the original units
are not exposed to the rest of the program and so need not be
unique. Due to the nature of OCaml's module system, packed modules can
only be linked in their entirety, which can dramatically increase the
size of executables. Packed modules are also a bit inflexible. It is
difficult for multiple libraries to install their contents within a
single packed module, or to use them craft a custom namespace for a
particular application.

2. Module aliases, in combination with the `-open` and `-no-alias-deps`
compiler options, can be used to give modules long unique names and then
expose them with short names within a hierarchical namespace. Build
systems such as jbuilder do this automatically in order to give the
illusion that compilation units have a hierarchical namespace. However,
implementing this system is a lot of work for these build systems and
even more work for tools like merlin and odoc to try to hide what is
going on under the hood. Even with all this hard work artefacts of the
process, e.g the hidden long unit names, do tend to occasionally leak
out to the user. Module aliases do not suffer from the linking issues
of packed modules. They are noticeably more flexible than packed modules
but it is still difficult for multiple libraries to install their units
within a single namespace.

The aim of this proposal is to replace these ad hoc mechanisms with more
direct support in the compiler. This should address some of their
short-comings and simplify the view presented to users.

An additional long term aim is to address the over-abundance of names
within the OCaml ecosystem. Currently a library with a single module has
at least 4 names associated with it:

1. The name of its module
2. The name of its .cmxa archive file
3. The name of its ocamlfind library
4. The name of its opam package

Ideally all these names are identical, but it would still be good to get
rid of some of them. This proposal aims to remove 2 immediately and to
lay the groundwork towards eliminating 3, at least for the majority of
packages.

## Design

The general approach of this proposal is to have the namespace of units
reflect the directory structure in which those units are found. Note
that this refers to the installation directories where the compiled
artefacts are found (`.cmi`, `.cmo`, .cmx`, etc.) rather than the
source directories.

This approach should provide a system that is easy to manage and fairly
flexible. It does not require the final name of a unit to known in
advance: moving a unit to a different position in the namespace simply
requires moving its compiled files to a different position in the
directory hierarchy.

### Removing `.cma` files

Since we are going to use the directory structure to assign names to
compilation units we need to make this structure available to both
linking and compilation. This requires moving away from using
`.cma`/`.cmxa` files for linking and instead passing the installation
directories to the linking command.

This can be addressed by allowing the `-I` option to be used with
linking commands, where `-I foo` is equivalent to passing a `.cmxa` file
containing all the `.cmx` files in the `foo` directory.

A directory containing the `.cmx` files of a library is already
required in order to support cross-module inlining, so using `-I` for
linking should be strictly less work.

In order to completely replace `.cmxa` files with `-I` we must also
provide alternatives for some additional features of `.cmxa` files.

#### `-cclib` and `-ccopt`

`-cclib` and `-ccopt` options can be added to `.cmxa` files so that they
are included when linking anything using that library. This feature can
probably be replaced by having the same functionality for `.cmx`
files. Possibly some support for deduplicating these options would be
necessary as well.

#### `-linkall`

The `-linkall` option can be used when creating a `.cmxa` file to force
the linking of all modules from the library even if some modules are not
transitive dependencies of the executable.

The main use case for this feature is when some modules in the library
depend indirectly on some other modules -- usually via
side-effects. This use case can instead be supported with a new
`-requires` option: adding `-requires Foo` when compiling the `bar` unit
will ensure that `Foo` is always linked into any executable that
includes `Bar`.

### `-P` option

Currently, other compilation units are made available to the compiler
using the `-I` option. For example,

    ocamlopt -c -I foo bar.ml

where `foo` is a directory containing the files `a.cmi` and `b.cmi` will
give the units described by `a.cmi` and `b.cmi` the names `A` and `B`
respectively, and make them available for use in `bar.ml`.

In order to support hierarchical namespacing for compilation units, a
second option `-P` should be added. Given the same foo directory as
above:

    ocamlopt -c -P foo bar.ml

will instead give the units described by `a.cmi` and `b.cmi` the names
`Foo.A` and `Foo.B` respectively and make a module `Foo` available for
use in `bar.ml`. `Foo` will consist of two module aliases `A` and `B`
pointing to `Foo.A` and `Foo.B`.

#### Sub-directories

In order to support nested unit hierarchies the `-P` option will also
make sub-directories available as sub-modules. For example, if the `foo`
directory contains a sub-directory `baz` which in turn contains units
`c.cmi` and `d.cmi` then these units will be given the names `Foo.Baz.C`
and `Foo.Baz.D` respectively, and the `Foo` module will contain a
sub-module `Baz` which consists of module aliases `C` and `D` pointing
to `Foo.Baz.C` and `Foo.Baz.D`.

#### Linking

As with `-I`, `-P` is used to make units available for linking as well
as compilation. For example,

    ocamlopt -P foo bar.cmx

where the `foo` directory contains `a.cmx` and `b.cmx` would make those
units available for linking as `Foo.A` and `Foo.B` respectively.

#### Relative naming

The names given to units found by `-I` or `-P` are relative rather than
absolute. If a unit is given the name `Foo.A` and refers to another unit
as `B` then we will look for this second unit relative to `Foo.A`. This
means that we first look for `Foo.B` and only if no such unit exists do
we look for `B`. Similarly, if `Foo.A` refers to another unit as `Bar.C`
then we will first look for `Foo.Bar.C` and only if that does not exist
do we look for `Bar.C`.

If you consider the absolute namespace of units as a tree:

              |--A
              |
      |--Foo--|--B
      |       |       |--C
      |       |--Bar--|
    --|               |--D
      |
      |--Baz--E
      |
      |--F


then `-I` and `-P` are used to present the view of that tree from a
particular unit. For example, when compiling unit `Foo.A` above you
should use:

 - an `-I` option to make the `Foo.B` module available as `B`

 - a `-P` option to make `Foo.Bar.C` and `Foo.Bar.D` available as
   `Bar.C` and `Bar.D`

 - a `-P` option to make `Baz.E` available as `Baz.E`

 - an `-I` option to make `F` available as `F`

and for the final linking you should use:

 - a `-P` option to make `Foo.A`, `Foo.B`, `Foo.Bar.C` and `Foo.Bar.D`
   available with those names.

 - a `-P` option to make `Bar.E` available with that name

 - an `-I` option to make `F` available with that name

When the linker sees that `Foo.A` refers to a module `Bar.C` it will
look for `Foo.Bar.C` and find it, and when it sees that `Foo.A` refers
to `F` it will first look for `Foo.F`, which will fail at which point it
will look for `F` and find it.

The obvious question arises: "What if there is a `Foo.F` available at
linking which was not available when compiling `Foo.A`?". This
inconsistency would be detected using `.cmi` digests and produce an
error at link time: much like inconsistent compilation environments do
currently. In exchange for these potential inconsistencies we are able
to have relative naming with two simple command-line options -- being
more precise about the shape of the tree would require a more complex
set of options.

### `OCAML_NAMESPACES` environment variable

In addition to `-P` we propose an additional mechanism for creating
hierarchical namespaces: the `OCAML_NAMESPACES` environment
variable. This variable would have the form:

    OCAML_NAMESPACES=name1:/path/to/dir1;name2:/path/to/dir2

If `/path/to/dir1` contained the directory `foo`
(i.e. `/path/to/dir1/foo`) which in turn contained the compiled
interfaces `a.cmi` and `b.cmi` then these would be given the names
`name1:Foo.A` and `name1:Foo.B` respectively, and a module `Foo` would
be made available which contained module aliases `A` and `B` pointing to
`name1:Foo.A` and `name1:Foo.B`.

#### Sub-directories

As with `-P`, `OCAML_NAMESPACES` will also make sub-directories
available as sub-modules. For example, if `/path/to/dir1/foo1` contains
a sub-directory `baz` which in turn contains a unit `c.cmi` then that
unit will be given the name `name1:Foo.Baz.C` and the `Foo` module will
contain a sub-module `Baz` which consists of a module alias `C` pointing
at `name1:Foo.Baz.C`.

#### Linking

As with `-I` and `-P`, `OCAML_NAMESPACES` is used to make units
available for linking as well as compilation. For example, with

    OCAML_NAMESPACES=name1:/path/to/dir1

where the `/path/to/dir1/foo` directory contains `a.cmx` and `b.cmx`
would make those units available for linking as `name1:Foo.A` and
`name1:Foo.B` respectively.

#### Absolute naming

Unlike `-P`, the names given to units found via `OCAML_NAMESPACES` are
absolute. All references to `name1:Foo.A` in all units are considered to
refer to the same unit, which is always expected to be found at
`/path/to/dir1/foo/a.cmi`.

### Symbol names

Whilst this proposal makes OCaml's compilation unit namespace
hierarchical, the underlying symbol namespace used by system linkers is
assumed to be flat. As such we must still be able to create unique
symbol names for every compilation unit. To support separate compilation
these unique names should be based on information available when
creating the unit's `.cmi` file.

This is done by using the combination of the units original name
(i.e. `A` for units described by files named `a.cmi`) and the digest of
its `cmi` file.

This is very likely to be unique, but we also propose providing a
`-salt` command-line option which can be used when creating a `.cmi`
file to add additional data to be included in the symbol name. For
example,

    ocamlopt -c -salt foo a.mli

and

    ocamlopt -c -salt bar a.mli

would always produce different symbol names even if both `a.mli`s were
identical.

## Use cases

This proposal is primarily designed to support three uses cases.

1. Existing libraries
2. Libraries using `-P` with ocamlfind
3. Libraries using `OCAML_NAMESPACES`

### Existing libraries

Existing libraries expect to be used with`-I` during compilation, and
via a `.cmxa` file during linking. They rely on ocamlfind to use the
appropriate `-I` and `.cmxa` options, as well as to provide similar
options for their dependencies. Since units handled with `-I` and
`.cmxa` are all treated as having top-level names under this proposal,
these libraries will behave exactly as they did before (including the
requirement that their unit names be sufficiently unique).

### Libraries using `-P` with ocamlfind

Instead of being used with `-I` and `.cmxa` files, under this proposal
libraries can be designed to be used with `-P` during both compilation
and linking.

In this scheme, the units in the library would be compiled using `-I` to
refer to the other units in the library, and using `-P` to refer to any
sub-libraries. The units would then all be installed in a directory
named after the library.

Using such a library would require passing this installation directory
using a `-P` option. ocamlfind would be relied on for passing this `-P`
option to the compiler, as well as to provide similar options for the
library's dependencies.

### Libraries using `OCAML_NAMESPACES`

Under this proposal libraries could avoid the need to use ocamlfind and
be designed for use with the `OCAML_NAMESPACES` environment variable for
both compilation and linking.

In this scheme, the units in the library would be compiled using `-I` to
refer to the other units in the library, and using `-P` to refer to any
sub-libraries. The units would then all be installed in a directory
named after the library directly beneath a known namespace directory. In
particular, systems like opam could provide:

    OCAML_NAMESPACES=opam:~/.opam/4.05.0/lib/

or perhaps:

    OCAML_NAMESPACES=opam:~/.opam/4.05.0/lib/ocaml/site-packages/

and the `foo` library would install its units into
`~/.opam/4.05.0/lib/foo`.

Libraries installed in this way are immediately available as `Foo` so
there is no need for ocamlfind to provide options for them at compiling
or linking. Similarly, it can remove the need to track dependencies: if
library `foo` was found via `OCAML_NAMESPACES` when compiling library
`bar`, then it will also be available via `OCAML_NAMESPACES` when
compiling/linking things that depend on `bar`.

## Open questions

There remain various open questions around this proposal.

### Representing multiple implementations/ocamlfind predicates

OCaml allows link-time selection between multiple implementations of a
compilation unit. By default, this proposal assumes that all compilation
units have a single implementation found in the same location as the
interface. This is a reasonable default, but it is also desirable to
allow users at link time to choose between different implementations.

Currently this choice is often accomplished via ocamlfind predicates --
a very powerful mechanism to let the user or dependent libraries control
which compiler options are used for a library and thus select between
different implementations (or even different interfaces).

One approach to this issue with our proposal would be to allow
additional sub-directories within a library (or sub-library) whose names
are marked in some way -- for example they might be prefixed with `@` --
which are included into their parent directory when a particular
command-line option is given.

For example, if the `foo` directory looked like:

    @bar -- a.cmx
    |     | b.cmx
    |
    @baz -- a.cmx
    |     | b.cmx
    |
    a.cmi
    b.cmi

then you could compile against the `foo` library with:

    ocamlopt -c -P foo c.ml

and link against the `bar` implementation with:

    ocamlopt -variant bar -P foo c.cmx

It might also be a good idea to support a `-requires-variant` option, so
that:

    ocamlopt -c -requires-variant bar -P foo c.ml

that would enforce the use of the `bar` variant for anything that linked
`c.cmx`.

This essentially provides a simpler, less powerful, version of
predicates. It is hard to know what proportion of the uses of predicates
in the wild it is sufficient to handle without doing a thorough review
of these uses on opam, but it seems plausible that it would handle a
majority of cases.

### What should be in the initial scope?

In the proposal as described all libraries installed within an
`OCAML_NAMESPACES` directory are available in the initial scope without
needing to specify them on the command-line. Some people have raised
concerns about this making code bases harder to understand since the
build system would no longer require a list of dependencies.

It is not clear how much of a concern this is, since in most
environments the list of dependencies is already in the library's opam
file. However, it does seem a good idea to support a mode which does not
automatically introduce the libraries into the initial scope.

In particular, there could be an option `-no-namespaces`, along with an
extension to OCaml's existing support for paths like `+foo` in `-I`,
such that:

    ocamlopt -no-namespaces -P +foo c.ml

would make the `foo` library from `OCAML_NAMESPACES` available in the
initial scope, but none of the other libraries from `OCAML_NAMESPACES`.

The default could also be the other way around, requiring a
`-namespaces` option to bring all the namespaced libraries into the
initial scope.

I think that `-namespaces` would definitely be the right default for the
REPL as the need to `#require` libraries to use them is a constant
pitfall for beginners.

### Including values in "package" modules

Many libraries have a top-level module containing values as well as
sub-modules. The design above forces libraries using `-P` or
`OCAML_NAMESPACES` to only include sub-modules in their top-level module.

There are two similar ways to add support for values in top-level
modules:

1. If a library directory contains a specially named unit then `include`
   the contents of that unit, after the module aliases, in the module
   corresponding to that directory.

2. If a library directory contains a specially named unit then `include`
   the contents of that unit as the entirety of the module corresponding
   to that directory.

With the first approach, the library author does not need to include
aliases to the other units in the library, which is especially useful if
the list of other units may not be known when that unit is compiled.

With the second approach, the library author must list the other units
of the library, but can also expose the units under different names than
they are given within the library.

It is not immediately clear which approach is better, although it is
worth noting that jbuilder takes the second approach in its emulation of
namespaces.

## Implementation

The proposal outlined here is relatively simple and, in principle, it
should be fairly easy to implement. However, OCaml is not very precise
with how it handles compilation unit names and names more
generally. There are quite a few places that make implicit assumptions
about the form of compilation unit names, and there are no type-level
distinctions between different kinds of name to make these places easy
to locate. For this reason the implementation has been split into two
separate patches. The first patch will make OCaml's handling of names
more precise without changing any of OCaml's external behaviour. The
second will then implement this proposal, taking advantage of the
increased precision to make its correctness more obvious.
