# 0 — Installing and running the compiler

There are two routes to a working `m9c`, and both end at the same
place.  What they need is deliberately small: **the only compiler
required is gcc.**  M9 compiles to C11, the generated C for the
toolchain itself is checked into the repository as the bootstrap,
and a CI gate builds the whole thing on a machine where every other
compiler has been replaced by a script that fails loudly — so the
claim is tested, not asserted.

## Route 1: the Debian package

Download
[`m9_0.3.0_amd64.deb`](https://github.com/atverm/M9Tutorial/raw/main/m9_0.3.0_amd64.deb)
from the [M9Tutorial repository](https://github.com/atverm/M9Tutorial)
— which also holds every example in these chapters — and:

    sudo apt install ./m9_0.3.0_amd64.deb

This puts `m9c` in `/usr/bin`, the runtime header in
`/usr/include/m9`, the runtime archive at `/usr/lib/libm9rt.a`, and
the standard library — as M9 SOURCE, because a readable library is
part of the point — in `/usr/lib/m9`, which the compiler searches
without any variable being set.  The per-module reference pages land
in `/usr/share/doc/m9/modules`, the language report beside them, and
`man m9c` works.

The package is built on Ubuntu 26.04 for amd64, and a deb names
its distribution's library versions — on another release, build the
package there (`dpkg-buildpackage -us -uc -b`) or use route 2.

## Route 2: from source

    git clone <the m9 repository> && cd m9repo
    ./build.sh                 # needs gcc, nothing else

Two artifacts appear: `out/m9c` and `out/libm9rt.a`.  To use the
compiler in place, tell it where the library and runtime sources
live (installed via route 1, neither variable is needed):

    export PATH=$PWD/out:$PATH
    export M9LIBRARY=$PWD/corpus
    export M9RUNTIME=$PWD/runtime

`./build.sh DESTDIR` also installs the route-1 layout under a
prefix, which is exactly how the package itself is assembled.

## The first program

    cat > Hello.m9 <<'M9'
    MODULE Hello ;
    IMPORT Io ;
    BEGIN
      Io.WriteLine ('it works')
    END Hello.
    M9
    m9c --make -o hello Hello.m9
    ./hello

`m9c` checks, generates C, and drives gcc; `--make` also builds any
imported module whose object is missing or stale, deepest first.
Without `--make`, a missing library object is not an error dump —
it is NAMED, with the exact command that produces it, before the C
compiler runs.  The compiler supplies the include paths, the module
objects, the runtime and `-lm` on its own; `-v` prints the composed
gcc line when you want to see precisely what it did, and anything
you place after `--` replaces the supplied flags entirely, because
m9c is not a build system and does not pretend to be one.

The flags you will actually use:

    m9c --make -o prog Main.m9    compile and link a program
    m9c --make -c Mod             compile one module to Mod.o + Mod.h
    m9c -g -o prog Main.m9        debuggable build (keeps the C;
                                  gdb breaks by function, walks
                                  file:line)
    m9c --doc Mod.m9              render the module's reference page
                                  from its own definition comments
    m9c --version                 say which m9c this is

A refused program prints its diagnostics to stderr, with line and
column, and writes nothing — there is no output to half-trust.

## The editor: syntax and docstrings in VS Code

The repository ships a VS Code extension with the M9 grammar
(highlighting), HOVER DOCUMENTATION and dot-completion.  Hovers and
completion are fed by `m9c --doc` — the same generated pages you
read in `/usr/share/doc/m9/modules` — never by a private re-scan of
the source, so what the editor shows you is what the compiler
believes.  (The repository refuted the second-scanner design with
its own corpus: a code generator holding eighteen string literals
containing `(*` defeats any scanner that does not lex strings.)

From the Debian package, the extension is installed at
`/usr/share/m9/vscode-m9`; VS Code loads per-user extensions, so
link it once:

    ln -s /usr/share/m9/vscode-m9 \
          ~/.vscode/extensions/atverm.m9-lang-0.2.0

From a source tree, link `tools/vscode-m9` to the same place.
Restart VS Code; `.m9` files get the grammar, and hovering
`Io.WriteLine` shows its contract.  If your modules import things
outside the default search path, the `m9.includePaths` setting
names the extra directories, and everything documented in chapter 3
— your own definition comments included — appears in your own
hovers.

## Checking the installation

    m9c --version                 m9c 0.3.0
    man m9c                       the reference, options and the
                                  supplied-flags contract
    ls /usr/share/doc/m9/modules  the standard library, one page
                                  per module

[Next: hello, and why M9 exists →](01-hello.md)
