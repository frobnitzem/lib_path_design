# Design Document for Installation of Dependent Packages

## Preamble

Monolithic applications, including systems software stacks,
are giving way to library-oriented architectures including:

* [Object-Based Design](https://en.wikipedia.org/wiki/Policy-based_design)
* [Type Interfaces](https://gobyexample.com/interfaces)
* [Module Interfaces](https://ocaml.org/learn/tutorials/modules.html#Interfaces-and-signatures)
* [Type Traits](https://docs.scala-lang.org/tour/traits.html)
* [Type Hints](https://docs.python.org/3/library/typing.html#callable)

At the system-level, all these library approaches require
robust support for resolving paths of installed components of
software packages, including:

* static libraries (resolved at link time)
* shared libraries (resolved at link and run-time)
* header files (resolved at compile time)
* data files, potentially containing interpreted code (resolved at run-time)

During installation, it has become standard practice for packages
that export the above kinds of components to declare their exports.

Down-stream packages that import these kinds of components
have two problems:

1. They must both a) find filesystem paths to and b) statically type-check their imports.
2. They must also export their own functionality to potential down-stream packages.

This document argues for the following conventions:

1. A set of well-known paths and data formats must be established for storing location, name and type information about exports.  For example (package A) exports to downstream (package B).
2. A generic import functionality should be available to understands these paths. For example (package B) inspects well-known paths to learn about the layout of (package A).
3. A generic "export" functionality should be available to create these paths.  For example, (package B) calls export to store its information.
4. The "export" function MUST be transitive, in the sense that downstream packages (package C) will import the same paths and types (package A) seen by its upstream (package B) at compile-time.

The last convention is essential to ensure that compile-time checks do not
need to be repeated.  In the case that multiple compile-time
configurations for (package B) are to be made available, each
configuration of (package B) should be given its own name and path
(e.g. package B1, package B2, etc.).

The alternative would require downstream packages (e.g. package C) to
internally model about all the potential options of package B during compile-time.
For an example, see the installation logic for the blaspp spack package,
which must carefully inspect the exact blas library for integer size,
threading support, etc...

With current technology, it has been found to be much more robust to use
separate, stand-alone install paths for every installed package version.
This permits packages to store configuration options
statically, and to statically reference their upstream
installation locations.  Importantly, downstream packages
can rely on a static configuration for their dependencies,
and describe themselves accordingly.


## Use Cases


## Autotools Idiom

Before rpaths were added to ELF libraries, the GNU project introduced libtool.
Libtool's central idea is to provide `dependency_libs` and `inherited_linker_flags`
for each library so that its symbols can be resolved.

Libtool is now used only as a part of the autotools build process so that
libraries can be used from the build directory.

Because ELF libraries can contain full paths to their dependencies,
the challenge moves to build time.  How does a package take a list
of dependency packages and find their library, data,
and include files at build time?

GNU Autotools solves this by relying on [pkgconfig](https://autotools.io/pkgconfig/file-format.html).
Pkgconfig files are placed in a well-known location (/usr/lib/pkgconfig or /usr/share/pkgconfig)
during install, and reference the absolute system paths of their
installed directories.  They also include information on the specific library file
names and compile flags (commonly pthreads or openmp).

This idiom accomplishes the lookup problem in the preferred way, by
storing static, full-paths at install time.  However, it does not
have facilities for dealing with multiple compilers and the differences
in build flags they engender.  Neither does it provide a consistent
package naming or "enabled options" scheme.  This means two packages
with the same name cannot co-exist.


## Spack Idiom

[Spack](https://spack.readthedocs.io/) is a package manager system that solves
the dependency lookup problem by using three conventions:

1. All packages are installed into a unique, hash-based prefix.

2. All packages have a unique name and an extended "spec" that specifies all configured build options for the package.

3. Once installed, packages files are not to be modified (all its files are static or constant).

These conventions address the resolution of named dependencies to installation
libraries, include directories, and data locations.  They ensure that every
package can find all its dependencies at build time.  They can also query the
dependencies of those dependencies - since everything was installed statically.

However, spack does not deal specifically with propagating build flags
to dependencies.  Instead, it relies on a separate "spec" for compiler and
system architecture that it translates into appropriate options
to `configure` or `cmake` for every package it compiles.

This works well enough in practice, because `configure` and `cmake`
have the ability to create `inherited_linker_flags` based on
system checks run during the build process.

Problems can arise, however.  Upstream packages might
require special linker flags, for example requiring
openmp or accelerator-specific options.
Also, for header-only libraries, the package might require
special preprocessor definitions.  Downstream packages
need to include these to compile, link and run properly.

Package versions and specs, combined with static attributes
that spack can query, are designed to help downstream
packages add the correct information to their build steps.
However, these extra attributes are often un-documented
and vary from one package to another.


## CMake Idiom

Cmake has attempted to propagate preprocessor and linker flags
to downstream packages using `target`s.  A CMake target
is a set of built files that can be installed, combined
with many `target attributes` that are needed either
to build the object (PRIVATE scope), use the object
(INTERFACE scope) or both (PUBLIC scope).

Target attributes can include things such as
`link-libraries` (libtool's `dependency_libs`),
`include-directories`, `compile-options` (libtool's `inherited_linker_flags`
and things like C++ standard and GPU architecture combined),
or targets-as-dependencies.

Setting a target as a dependency using cmake's `target_link_libraries`
can add link-libraries, include-directories, and compile-options together.

In order to propagate all that information downstream,
cmake introduces the idea of exported targets.  These
are a set of files installed in a package-local directory.
There are [many potential, canonical locations](https://cmake.org/cmake/help/latest/command/find_package.html#search-procedure) to save these export info files.

However, cmake assumes its typical use-case is to build
packages for a package manager to later
install into global locations like `/usr`,
`/usr/local`, or `/opt`.  Because of this, its exported
targets attempt to contain only relative paths.
The idea is that `make install` can install all files
in `/tmp`, and those can be moved into a package
management format like `rpm` or `dpkg`.  Then they
can later be installed to their final, global
location.

Support for this relocation feature means that
installed packages can be moved as a group
(or mounted at different locations) and still
resolve each other at compile-time.
Being global is important here, since
then all installed packages can use a single
`CMAKE_PREFIX_PATH`.

Multiple prefix paths are supported,
but a *different prefix path for each
dependency breaks transitivity*.

The problem occurs when package `C` knows
it needs package `B`.  Package `B` knows it
needs package `A`.  If `B` and `A` are in different
paths, `C` is forced to include all installation
paths.  This is impossible unless it knows about
all of `B`-s transitive dependencies.

A similar problem occurs with python.
Since python import statements don't reference
an absolute path for their dependencies,
the environment must contain correct
locations of all python imports of `B`
at the time `C` is run.

Pkg-config files usually include absolute
paths to their dependencies, so do not have
the same issue (in most cases).


## Proposed Link Idiom

It is possible to achieve the 4 conventions set out in
this design document using a combination of spack
with cmake and pkgconfig.  Either of the following
two solutions work:

1. Encode full paths to dependency packages in cmake exports.

2. Require spack to populate `CMAKE_PREFIX_PATH` with
   a transitive list of dependent modules during cmake
   configuration steps.

The first is strongly preferred, for the following reasons:

1. It matches the pkgconfig convention.

2. It is discoverable by downstream builds not using spack.

3. It has a greater chance of preventing linking errors,
   since it is much harder to link with incompatible upstream
   packages.

