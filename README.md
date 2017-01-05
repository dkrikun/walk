walk(1) is a fast, lightweight, general purpose build and task execution system.

Heavily inspired by `make` and [redo](https://github.com/apenwarr/redo).

## What can it build?

* Frontend applications (e.g. sass -> css -> compress).
* C++ applications.
* Infrastructure.
* Anything that has dependencies.

## Features

* Blazingly fast parallel builds.
* Build shared dependencies once.
* Only build targets if their dependencies changed.

## DAG

At the core of **walk** is a Directed Acyclic Graph (DAG). DAG's are a magical data structure that allow you to easily express dependency trees. You'll find DAG's everywhere; in GIT, languages, infrastructure tools, etc. **walk** provides a general UNIX utility to express a DAG as a set of targets (files) that depend on each other.

## Basics

### Targets & Rules

Similar to make, **walk** has the concept of "targets". A target is simply a file that can be built from some dependencies. However, unlike make (and like redo), how targets are built (rules) are defined as executable files instead of in a Makefile. This provides for unlimited flexibility to define how something is built, and what it depends on, by composing existing tools.

For example, if I wanted to describe how to build a binary called "hello" from "hello.c", I could do so with a bash script called "hello.walk":


```bash
#!/bin/bash

deps() {
  echo hello.c
}

compile() {
  gcc -Wall -o hello $(deps)
}

case $1 in
  deps) deps ;;
  exec) compile ;;
esac
```

Executing `walk hello` will build the target:

```$
$ walk hello
hello
```

This is the core of how walk works, and you can use it to build up very large dependency trees.

### Phases

**walk** has two phases:

1. **Plan Phase**: In this phase, **walk** executes all the `.walk` files with `deps` as the first argument. `.walk` files are expected to print a newline delimited list of files that the target depends on, relative to the target. Internally, **walk** builds a graph of all of the targets and their dependencies.
2. **Exec Phase**: In this phase, **walk** executes all of the `.walk` files with `exec` as the first argument. `.walk` files are expected to build the given target.

By separating these phases, **walk** can build a compact dependency graph, and perform fast parallel builds.

### Arguments

When **walk** executes a `.walk` file, it executes it with the following positional arguments:

1. `$1`: The phase (**deps** or **exec**).
2. `$2`: The name of the target to build (e.g. `hello.o`).

### Conditional Execution

By design, walk does not try to perform any kind of conditional execution of targets (e.g. if the file modification time has changed, like make). Conditional execution is left to the `.walk` file in the **exec** phase. For example, if I wanted to only build `hello` if `hello.c` has changed, I can do so like so:

```bash
#!/bin/bash

name=hello

deps() {
  echo hello.c
}

# Returns true if any of the dependencies are newer than the object file.
changed() {
  for d in $(deps); do
    if [[ "$d" -nt "$name" ]]; then
      return 0
    fi
  done
  return 1
}

compile() {
  gcc -Wall -o "$name" $(deps)
}

case $1 in
  deps) deps ;;
  exec)
    if changed; then
      compile
    fi
    ;;
esac
```

Different targets may require different conditional execution behaviors. For example, I may want to perform a hash of all of the contents of all of the dependencies instead.

### Output

**walk** follows the design trope of UNIX, that Silence is Golden. By default, only stderr from `.walk` files is shown, and **walk** outputs the targets that it has built. This allows it to be easily composed with other tools:

```
walk | grep test
```

You can show stdout from `.walk` files by using the `--stdout` flag.
