# 0 — Installing and running the compiler

There are two routes to a working `m9c`, and both end at the same
place.  What they need is deliberately small: **the only compiler
required is gcc.**  M9 compiles to C11, the generated C for the
toolchain itself is checked into the repository as the bootstrap,
and a CI gate builds the whole thing on a machine where every other
compiler has been replaced by a script that fails loudly — so the
claim is tested, not asserted.

## Route 1: the install package

The [release page](https://github.com/atverm/m9c/releases/tag/v0.4.1)
of the compiler's repository, [m9c](https://github.com/atverm/m9c),
carries one package per distribution, x86-64, each built ON that
distribution from the same source tarball (the
[M9Tutorial repository](https://github.com/atverm/M9Tutorial) holds
every example in these chapters):

| distribution | package | install |
|---|---|---|
| Ubuntu 24.04 LTS | `m9_0.4.1-1_amd64.ubuntu24.04.deb` | `sudo apt install ./m9_0.4.1-1_amd64.ubuntu24.04.deb` |
| Ubuntu 26.04 LTS | `m9_0.4.1-1_amd64.ubuntu26.04.deb` | `sudo apt install ./m9_0.4.1-1_amd64.ubuntu26.04.deb` |
| Debian 13 | `m9_0.4.1-1_amd64.debian13.deb` | `sudo apt install ./m9_0.4.1-1_amd64.debian13.deb` |
| Fedora 43 | `m9-0.4.1-1.fc43.x86_64.rpm` | `sudo dnf install ./m9-0.4.1-1.fc43.x86_64.rpm` |
| Rocky 9 (RHEL 9, Alma 9) | `m9-0.4.1-1.el9.x86_64.rpm` | `sudo dnf install ./m9-0.4.1-1.el9.x86_64.rpm` |
| Arch | `m9-0.4.1-1-x86_64.pkg.tar.zst` | `sudo pacman -U ./m9-0.4.1-1-x86_64.pkg.tar.zst` |

Each package comes with a `.receipt` beside it — the distribution it
was built on, its sha256, the tarball it came from, the gcc that
built it, and the line proving the installed compiler compiled and
ran a program with an empty environment.  What is on the page is
what came back from the build machine, unchanged.

Any of them puts `m9c` in `/usr/bin`, the runtime header in
`/usr/include/m9`, the runtime archive at `/usr/lib/libm9rt.a`, and
the standard library — as M9 SOURCE, because a readable library is
part of the point — in `/usr/lib/m9`, which the compiler searches
without any variable being set.  The per-module reference pages land
in `/usr/share/doc/m9/modules`, the language report beside them, and
`man m9c` works.

The two debs are different files — a deb names its distribution's
library versions — so take the one for your distribution.  On any
other distribution or release, route 2 is a two-line build.

## Route 2: from source

    git clone https://github.com/atverm/m9c && cd m9c
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

## Other editors: the language server

For editors that speak the Language Server Protocol — Neovim, Helix,
Emacs with eglot, Kate — the repository carries `m9lsp`, a language
server written in M9 itself (`corpus/Lsp.m9`).  It publishes
diagnostics on open and on save, and it holds the same rule as the
VS Code extension: it never lexes M9 privately.  Every diagnostic is
the compiler's own — the server runs `m9c --check` (check, print,
write nothing) and republishes the messages with their line and
column, so an editor squiggle can never disagree with the build.

The server is not a second binary to install: `Lsp.m9` ships in the
standard library, and you build it with the compiler you already
have — one command, anywhere you have write permission:

    m9c --make -o m9lsp Lsp

This needs m9c 0.4.1 or newer (`m9c --version` says; the 0.4.0
packages predate `--check` and the library module by a few hours).
From a source build (route 2), any current checkout works.

Wire it as a stdio server for `.m9` files.  Helix
(`~/.config/helix/languages.toml`):

    [language-server.m9lsp]
    command = "m9lsp"

    [[language]]
    name = "modula9"
    scope = "source.modula9"
    file-types = ["m9"]
    roots = []
    language-servers = ["m9lsp"]

Neovim (0.11+):

    vim.lsp.config['m9lsp'] = {
      cmd = { 'm9lsp' },
      filetypes = { 'modula9' },
    }
    vim.lsp.enable('m9lsp')
    vim.filetype.add({ extension = { m9 = 'modula9' } })

The server finds `m9c` on `PATH` (or `$M9LSP_M9C` names one), and
the compiler's own search rules apply — `$M9LIBRARY`, the installed
library, and the checked file's directory — so a file whose imports
sit beside it needs no configuration at all.  Checks run against
the file on disk: save to see fresh diagnostics.  In VS Code, keep
using the extension above; it already has hovers and completion,
which the server does not yet.

## Checking the installation

    m9c --version                 m9c 0.4.1
    man m9c                       the reference, options and the
                                  supplied-flags contract
    ls /usr/share/doc/m9/modules  the standard library, one page
                                  per module

[Next: hello, and why M9 exists →](01-hello.md)
