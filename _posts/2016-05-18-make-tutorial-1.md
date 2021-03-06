---
layout: post
title:  "Writing a simple Makefile"
categories: blog
---

# Introduction

I often write small programs to prototype and test out ideas and constructs in isolation, before integrating or expanding them into bigger projects.

When programming in a scripting language such as Python, it's as easy as just writing the script and then invoking the interpreter on it: `python main.py`. This is the case even if main.py imports code from other files foo.py and bar.py.

In a compiled language such as C or C++, there is the additional overhead of compiling and linking each file, with non-trivial sequences of commands such as `gcc -g -Wall main.c foo.c bar.c -o main && ./main`. This command will actually recompile all of the source files, even if they weren't modified since the last time they were compiled, so it's not particularly efficient.

# Enter make

make is a build automation tool that provides an easy way to achieve timestamp-based conditional recompilation, and interpreter-like behavior even for compiled languages. That means you can have your project efficiently compiled and running with a single command, `make`.

The original make tool has just recently turned 40, and was first included in one of the earliest versions of UNIX, back in 1976. Several versions of make now exist out there, such as Microsoft's nmake and BSD's pmake, but the most common is by far GNU make, as it is standard in both Linux and OS X. It is the version that will be described in this tutorial.

Its working principle is quite simple, you create a list of **rules**, place them in a file, and then call `make` with an optional target name. When no target name is given, make will try to generate the first target on the list. By default, it will look for files named Makefile or makefile, other filenames can be specified using the `-f filename` command-line option.

A Makefile **rule** consist of a **target** <- **prerequisites** dependency relation, and a list of commands, called a **recipe**, to generate the target from the prerequisites. They are written in the form:

```make
target ... : prerequisites ...
    recipe
    ...
#^^^recipe lines must be preceded by an actual tab character    
```

Internally, make builds a representation of the dependency relations in the form of a [directed acyclic graph](https://en.wikipedia.org/wiki/Directed_acyclic_graph), a data structure similar to a tree, but where each node can have more than one parent. The nodes of the graph are then traversed in post-order, and if any of a target's prerequisites is determined to be newer than the target itself, the target is regenerated by running the corresponding recipe.

# Simplifying

With all that said, you don't actually need a Makefile to compile a single-file program. If your file is called foobar.c, you can just call `make foobar`, and it will call the C compiler to compile it for you. If your file instead ends in .cpp or .cc, make will call the C++ compiler, and analogously for Fortran 77 .f files, Pascal .p files, and Assembly .s files (AT&T syntax).

To compile multi-file programs, you can use very simple one-line Makefiles of the form:

```make
main: main.c foo.c bar.c
```
If you run make and examine the command that it performs, you'll see something like `cc main.c foo.c bar.c -o main`. The intermediate object files are not kept, so this cannot be used to selectively recompile only changed files.

So what is going on, how does make know the recipes to generate these targets?

# Implicit rules

The key is make's [implicit rules](https://www.gnu.org/software/make/manual/html_node/Implicit-Rules.html#Implicit-Rules), which allow writing very succinct yet powerful Makefiles.

For instance, the following one-line Makefile supports timestamp-based recompilation of object files, and should satisfy the needs of most simple C programs.

```make
main: main.o foo.o bar.o
```

However, it's not at all obvious how or why it works, since its behavior is entirely defined by implicit rules. If the relevant rules are made explicit, the following is obtained:

```make
main: main.o foo.o bar.o

# implicit rules
%: %.o
    $(CC) $(LDFLAGS) $(TARGET_ARCH) $^ $(LOADLIBES) $(LDLIBS) -o $@

%.o: %.c
    $(CC) $(CFLAGS) $(CPPFLAGS) $(TARGET_ARCH) -c -o $@ $<
```

If you have only limited experience with Makefiles, the above can seem quite daunting at first, so let's dissect it into manageable parts.

# Pattern rules
Let's start by examining the rule `%.o: %.c`.

The presence of a `%` character in a target indicates a [pattern rule](https://www.gnu.org/software/make/manual/html_node/Pattern-Rules.html#Pattern-Rules), and works much like a conventional wildcard character like * in the shell. The portion of the target that is matched by the wildcard is called the **stem**. Subsequent use of `%` in the prerequisites list is replaced by the same stem.

In this context, `%.o` matches with `main.o`, `foo.o`, and `bar.o`. The corresponding prerequisite `%.c` is then replaced by `main.c`, `foo.c`, and `bar.c`, respectively.

The other rule `%: %.o` works similarly. Since the target is only the wildcard character `%`, this is a [match-anything rule](https://www.gnu.org/software/make/manual/html_node/Match_002dAnything-Rules.html#Match_002dAnything-Rules), and will apply to anything not already covered by other rules. But in certain circumstances, there can be [multiple rules for the same target](https://www.gnu.org/software/make/manual/html_node/Multiple-Rules.html): in this case, the rule `main: main.o foo.o bar.o` gives additional prerequisites to the implicit `%: %.o` rule, which has the actual recipe.

# Variables

make supports the use of variables in Makefiles, which can be used to avoid repetition, improve clarity, and increase maintainability. Variables are expanded by using the dollar sign `$` operator, followed by the variable name in parentheses. If the variable name is only a single character, the parentheses can be omitted. To use an actual $ character, write `$$` (useful for shell variables, such as `$$PATH`).

The [info function](https://www.gnu.org/software/make/manual/html_node/Make-Control-Functions.html#Make-Control-Functions) can be used to check the value of a certain variable: `$(info text)` will write `text` to the standard output (more on functions in a later post).

```make
# variable definitions
V = Hello
Var = World

$(info $V)     # Hello
$(info $(Var)) # World
$(info $Var)   # Helloar, common pitfall
```
Recipes often make use of [implicit variables](https://www.gnu.org/software/make/manual/html_node/Implicit-Variables.html#Implicit-Variables), such as `CC`, `CFLAGS`, etc. Most of these either have reasonable defaults, such as `CC = cc`,or are empty and meant to be defined in the Makefile itself.

The following is an example of setting these variables for a project requiring a library called mylib:

```make
# C compiler
CC = gcc
# preprocessor flags
CPPFLAGS = -I./mylib/include
# compiler flags
CFLAGS = -g -Wall
# linker flags
LDFLAGS = -L./mylib/lib
# linked libraries
LDLIBS = -lmylib
# LOADLIBES is a deprecated and badly spelled alternative to LDLIBS
# target architecture
TARGET_ARCH = -march=native
```

Variables can also be defined at the command-line when calling make, for instance: `make CC=clang`, and these will override any corresponding definitions within the Makefile.

# Automatic variables
Additionally, make also offers [automatic variables](https://www.gnu.org/software/make/manual/html_node/Automatic-Variables.html#Automatic-Variables), which are defined on a per-rule basis. Of these, the most useful are:

* `$@` - target name, useful to specify the output name in the command.
* `$<` - first prerequisite, useful when the others are auxiliary, and not needed within the recipe.
* `$^` - list of prerequisites, useful to link multiple files.
* `$*` - the stem with which a pattern rule matches, useful to create derived filenames.

```make
main: main.o foo.o bar.o
    $(info $@) # main
    $(info $<) # main.o
    $(info $^) # main.o foo.o bar.o
    $(CC) $^ -o $@
#   cc main.o foo.o bar.o -o main

foo.o: foo.c foo.h bar.h
    $(info $@) # foo.o
    $(info $<) # foo.c
    $(info $^) # foo.c foo.h bar.h
    $(CC) $<
#   cc foo.c
```

It is now possible to correctly interpret the implicit rules used by make:

```make
# adds extra prerequisites to the implicit %: %.o rule
main: main.o foo.o bar.o

# implicit rules

# note that there must be an object file with the same name as the executable,
# otherwise this rule doesn't apply
# matches with main: main.o
%: %.o
#   links all the prerequisite objects $^ into the target executable $@
    $(CC) $(LDFLAGS) $(TARGET_ARCH) $^ $(LOADLIBES) $(LDLIBS) -o $@

# matches with main.o: main.c
#              foo.o: foo.c
#              bar.o: bar.c
%.o: %.c
#   compiles the prerequisite source file $< into the target object file $@
    $(CC) $(CFLAGS) $(CPPFLAGS) $(TARGET_ARCH) -c -o $@ $<
```

# Extra functionality

A typical Makefile also provides targets for cleaning intermediate files, and sometimes running the program. These targets represent actions, rather than actual filenames, and are therefore considered [phony targets](https://www.gnu.org/software/make/manual/html_node/Phony-Targets.html#Phony-Targets). Phony targets should be indicated by listing them as prerequisites of the special `.PHONY` target.

The recipes themselves are quite simple, clean should remove all the products of compilation, `$(RM) *.o main`, (note that `RM = rm -f`), and run should run the executable, `./main`.

To have make recompile a source file when one of its included headers changes, these need to be explicitly listed as prerequisites for the corresponding object file. In this example, it is assumed that all the source files include foobar.h.

Adding this extra functionality leads to a pretty general purpose Makefile:

```make
CFLAGS = -g -Wall

run: main
    ./main

main: main.o foo.o bar.o

main.o foo.o bar.o: foobar.h

clean:
    $(RM) *.o main

.PHONY: run clean
```
Since `run` is the first target, this allows compiling and running with a single command, `make`.

# Caveats

As is, the above Makefile will only work correctly with C and assembly programs. This is because the `%: %.o` linking rule uses the C compiler specifically, and won't by default link the libraries needed for other languages. There are various ways to solve this, either list the libraries as a linker flag, `LDLIBS = -lstdc++`, or redefine `CC = g++` (although then the symbol `CC` loses its meaning as the C compiler), or make the rule explicit, and replace `$(CC)` with the relevant compiler:

```make
main: main.o foo.o bar.o
    $(CXX) $(LDFLAGS) $(TARGET_ARCH) $^ $(LOADLIBES) $(LDLIBS) -o $@
```

As a project grows, it quickly becomes unmanageable to list and maintain every object file, with the right filename, in the Makefile, along with their header dependencies. This can and should be automated, by having make and gcc do the heavy lifting, tracking the source files and their dependencies.

# Closing notes

In this post I covered some of make's basic functionality, by detailing the inner workings of one of its most underused features, implicit rules. But I have barely scratched the surface. make is an extremely powerful tool, with a flexible declarative language supporting conditionals, string operations, parallel execution, and various other features which can be leveraged for your purposes, even beyond building programs.

In a future post I will cover advanced features, such as functions, substitution references, automatic dependency generation, different types of variables, secondary expansion, the use (and abuse) of recursive make, and techniques for debugging a Makefile. These will lead to a more maintainable Makefile, usable in large projects.
