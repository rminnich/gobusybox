# Go Busybox

Go Busybox compiles many independent Go commands into a single binary that
dispatches to the right one at runtime. It does this by rewriting each
command's source into a side-effect-free package and generating a new main
program around them.

## Language

### The binary and its commands

**Busybox**:
A single binary containing many Go commands, which selects one to run from its
invocation arguments.
_Avoid_: bb (that is the default output filename, not the concept), multibinary

**Command**:
A `package main` that becomes one entry in a busybox. A directory only
qualifies as a command if its package is named `main` and it has at least one
Go file surviving the build constraints.
_Avoid_: binary, executable, tool, applet

**Dispatch**:
Choosing which command to run, from `argv[0]` (usually via a symlink to the
busybox) or, when `argv[0]` is not a registered name, from `argv[1]`.
_Avoid_: routing, multiplexing

**Shellbang**:
A script whose interpreter line names the busybox and a command, e.g.
`#!/bbin/bb #!/bbin/date`. Plan 9 passes these arguments differently from Unix.
_Avoid_: shebang, hashbang

### The transformation

**Rewrite**:
The source-to-source transformation that turns a command into an importable
package with no global side effects. Only commands are rewritten; the packages
they import are copied unchanged.
_Avoid_: transform, compile, codegen

**Registered main**:
The function a command's `main` is renamed to, so that calling it becomes an
ordinary function call rather than program entry.
_Avoid_: entrypoint, real main

**Registered init**:
The generated function that calls a command's extracted initializers in the
order Go would have run them.
_Avoid_: setup, bootstrap

**Busybox init**:
One of the numbered functions holding the body of an original `init`, or the
assignment hoisted out of one global variable declaration.
_Avoid_: initN, init function

**Init order**:
The sequence in which a command's global variable initializers must run,
derived from type-checking rather than from source order. Preserving it is why
the set of files handed to the type checker is always sorted.
_Avoid_: dependency order

**Registry**:
The runtime map from command name to its registered init and registered main.
Each rewritten command adds itself to the registry from a generated `init`.
_Avoid_: table, dispatch map

**bbmain**:
The generated package holding the registry and the run loop. It is deliberately
dependency-free, so that importing it cannot pull anything into every command.
_Avoid_: runtime, bblib

### The generated module

**Generated tree**:
The directory Go Busybox writes, containing the generated main program, bbmain,
and every rewritten command and copied dependency. It is a self-contained Go
module and can be built on its own.
_Avoid_: temp dir, output dir, scratch dir

**Busybox module**:
The module the generated tree declares, named so that it can never collide with
the module of any command being compiled into it.
_Avoid_: bb module, fake module

**Vendor manifest**:
The generated list of which module each vendored package is attributed to.
It and the generated `go.mod` are written together from one package list,
because the go tool rejects a vendor directory that disagrees with its
`go.mod`.
_Avoid_: modules.txt, vendor list

**Synthetic module**:
A module invented for a package that has none of its own, so that it can be
recorded in the vendor manifest. Packages found in GOPATH mode each get one.
_Avoid_: fake module, placeholder module

**Synthetic version**:
The version recorded for a module whose real version is unknown or
inapplicable. A vendored build never resolves it, so it only has to be
syntactically valid and consistent.
_Avoid_: dummy version, v0.0.0

**Offline build**:
A compilation that reaches neither the network nor the module cache. Vendoring
everything into the generated tree is what preserves this property for inputs
that already had it.
_Avoid_: hermetic build, air-gapped build

### Finding commands

**Build environment**:
The Go toolchain settings used to load and compile packages: `GOOS`, `GOARCH`,
build tags, module mode, which compiler binary to run.
_Avoid_: env, Environ (ambiguous with the lookup environment)

**Lookup environment**:
The settings used to turn user-supplied patterns into package paths, namely the
search path and the u-root source shortcut. It decides *which* commands are
built, not *how*.
_Avoid_: env, Env (ambiguous with the build environment)

**Pattern**:
One user-supplied argument naming commands. It may be a file system path, a Go
package path, a glob of either, or an exclusion.
_Avoid_: arg, target, spec

**Exclusion**:
A pattern prefixed with `-`, removing matching commands from the set the other
patterns produced.
_Avoid_: negation, filter, ignore

**Search path**:
A list of directory prefixes that are each prepended to a pattern and tested,
so that commands from several repositories can be named without repeating their
common prefix.
_Avoid_: GBB_PATH, include path

**Workspace mode**:
Using a Go workspace to span several modules, so that commands from different
modules can be compiled together. This is the supported way to build across
modules from local checkouts.
_Avoid_: multi-module mode
