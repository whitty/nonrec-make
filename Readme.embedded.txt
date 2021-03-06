# embedded-nonrec-make

## Justification

I picked up nonrec-make as the basis for a make-based project for work
on a Linux hosted project targeting a mixed embedded Linux, plus
bare-metal microcontroller project.

The basic pieces of nonrec-make form a nice core:

 * Declarative language
 * Support for building multiple `BUILD_MODE`s using out-of-tree obj
   directories
 * Some abstraction of platform
 * In-built support for -j parallel builds

But with a bit of extra work we can have a nice platform with:

 * Built-in host-based unit testing in code and script tests
 * Better dependency tracking
 * Simple "convention-based" rules to avoid having to keep Rules.mk
   files up to date
 * Support for cross-testing
 * Helpers for maintaining cross-platform code

So after having learnt a lot for the work project I'm back to re-write
everything from scratch, under a free software license, getting a
chance to learn from some mistakes, and a chance to do things better
the second time.

## Incompatibility with nonrec-make

### `$^` dependencies

Existing `target_CMD` command example uses `$^` to for its link
command-line.  Since we now track Makefiles as dependencies of normal
targets `$^` includes makefiles that will confuse the linker.  Use
`$(filter)` to only pass .o files.

Specifically (if you use `=` rules) you can use `$(DEP_OBJS)` to only
pass .o files.

```
cli$(EXE)_DEPS = cli.o cli_dep.o
cli$(EXE)_CMD = $(LINK.c) $(DEP_OBJS) $(LDLIBS) -o $@
```

# Approach

## Makefile dependencies

Make works best when it has exactly the correct correct dependencies.
And when developing you want reliability of those dependencies.  Since
the `Rules.mk` files define the build commands if those files change
potentially they should require the rebuilding of targets definied in
those files.

One approach is the "kbuild" approach that generates build scripts for
each output file.  This approach works because if a the script for
generating the output file changes, then the output file needs to be
rebuilt.  This has the downside of requiring those files to be
regenerated on each run, slowing down startup.

Instead of following the build-script approach we instead add a simple
dependency on the makefiles that define.  Since make is a global
system and nonrec-make has two passes this could mean every makefile
could be a dependency of any file.  Instead we take the pragmatic
assumption and add dependencies on:

 * The Rules.mk file for the current $(d)
 * All Rules.mk files toward the root of the repository
 * The basic nonrec-make files themselves (everything loaded before the first Rules.mk)

## Separation of output files

All outputs *must* be kept to target specific obj/target
subdirectories otherwise interference between BUILD_MODEs would occur.

Thus `clean_extra` is deprecated

## Testing

Testing should be a first-class target of any usable build system.
You are writing tests for all of your code - right?  And especially
for embedded code, using good software design (ie appropriate
abstraction) significant algorithm coverage can be acheived on your
big fast host CPU.

### Testing stages

In reverse order:

1. The test target itself - building this causes the test to run.  If the test succeeds the target is built and make ends.  If the test fails the target must not be rebuilt.
2. The test must be re-run if the test program itself is rebuilt - thus the test target depends on it.  Conversely running the test should cause the program to be rebuilt if necessary
3. The test must be re-run if any input files change (they could change the results) thus the test target depends on any input files.  Conversely if any test files need generation they will be built as required.

Test environment

An open question is whether a test should be run in the directory of
the source file `$(d)`, or the context of the built binary
`$(OBJPATH)`.

`$(d)` is more natural if test input is in the source directory.
`$(OBJPATH)` has the benefit that tests don't need to be told their
`$(OBJPATH)` if they need to reference generated output.

The natural location differs for binary vs script (no generated files)
tests.


### Cross testing

1. How are programs run on target?  ssh? serial console? debugger? flash?
2. Test dependencies - test files need to be made available on the remote target.  network?  shares?  scp/rsync?
3. How is test-success determined?  Does the target support processes? exit codes?


# TODO

## Daytona compatiblity

LINKORDER? DEPENDS? Tests vs binaries?

## Host builds for embedded cross targets

Sometimes you need to build a host-tool to run to build a cross output.
