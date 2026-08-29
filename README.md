# Scientific programming in Modula-9 — first steps

M9 (Modula-9) is a Wirth-family language for scientific computing,
designed for machine-written, human-audited code.  It is **not
designed for human convenience**: it is verbose, clear, reliable and
auditable.  You get exactly what is written — and when what is
written is wrong, the compiler tells you and refuses to run it.
Every import is named, every numeric width is exact, every error a
procedure can raise is in its signature, every allocation says which
pool owns it, and the checks are part of what a program *means*:
there is no build in which they are absent.  That makes it a good
language for code written by AI agents, for human programmers — and
above all for the human reviewer who has to certify work she did not
write.

**The tutorial is live at
[tutorial.modula9.net](https://tutorial.modula9.net)** — every code
example on those pages is editable and runs, sandboxed, on the
server; what you read here is the same material, gated by the same
tests.

The tutorial service is, of course, **written entirely in M9** —
the HTTP server, the compile-and-run sandbox pipeline, the whole
backend — with a little JavaScript on the frontend for the editable
cells.  Serving the tutorial with the language it teaches is the
claim made structural.

This repository holds the tutorial's example programs, the full
tutorial text ([`tutorial/`](tutorial/README.md), the same chapters
the site serves), and the M9 compiler package.

## License

Code — the examples, the tutorial text, the M9 compiler in the
package — is licensed under the **GNU GPL v3** (see
[LICENSE](LICENSE)).  The measurement data in
`examples/data/icos-obspack.zarr` remains **CC BY 4.0** by the ICOS
data licence ([doi:10.18160/JZ2X-GZGU](https://doi.org/10.18160/JZ2X-GZGU));
cite ICOS when you use it.

## Install

The [release page](https://github.com/atverm/M9Tutorial/releases/tag/v0.3.0)
carries one package per distribution — Ubuntu 24.04 and 26.04,
Debian 13, Fedora 43, Rocky 9 (RHEL 9 family) and Arch, x86-64 —
each built on
that distribution from the one attached source tarball and made to
compile and run an M9 program there before it was published; the
`.receipt` beside each says on what, with which gcc, and its sha256.
gcc is the only compiler any of them needs.  For Ubuntu 24.04:

    sudo apt install ./m9_0.3.0_amd64.ubuntu24.04.deb

(`dnf install ./m9-0.3.0-1.fc43.x86_64.rpm`, `./m9-0.3.0-1.el9.x86_64.rpm`,
`pacman -U ./m9-0.3.0-1-x86_64.pkg.tar.zst` for the others; chapter 0
of the tutorial has the whole table.)  This installs `m9c` (the compiler), the runtime, the standard
library as readable M9 source in `/usr/lib/m9`, per-module reference
pages in `/usr/share/doc/m9/modules`, `man m9c`, and a VS Code
extension (syntax highlighting, hover documentation, completion) at
`/usr/share/m9/vscode-m9` — link it once with:

    ln -s /usr/share/m9/vscode-m9 \
          ~/.vscode/extensions/atverm.m9-lang-0.2.0

The package is built on Ubuntu 26.04 for amd64.  On another release
the C library versions will not match — the compiler itself needs
nothing but gcc, so building from source is the route there.

## The first program

    git clone https://github.com/atverm/M9Tutorial
    cd M9Tutorial/examples
    m9c --make -o hello C1Hello.m9
    ./hello

`m9c` checks the program, generates C11, and drives gcc; `--make`
also builds any imported library module.  A refused program prints
its diagnostics with line and column and writes nothing.

## The examples

Every example is a **complete program**, and each lands one idea:

| file | the idea |
|---|---|
| `C1Hello.m9` | a module's body is the program; the root `EXCEPT` block is where failure is decided |
| `C1Bounds.m9` | a deliberate run off the end of an array: the access raises `IndexError` instead of reading past it |
| `C2Widths.m9` | exact widths; conversions are calls that RAISE when the value does not fit — `I64 (NaN)` included |
| `C2Wrap.m9` | checked `+` raises Overflow, always; wraparound is a different operator, `+%`, visible and greppable |
| `Temps.m9` | a DEFINITION is a checked contract: exceptions with payloads, complete RAISES lists |
| `C3Use.m9` | the client's side: the handler that names the module's exception and binds its payload |
| `C4Mem.m9` | memory made visible: pools own storage, slices view it, VAR says who writes, strings are slices of CHAR — with the docstring convention modelled |
| `C5Csv.m9` | reading data with **declared** column kinds and a declared missing value — nothing inferred |
| `C6Stats.m9` | mean, percentiles, regression and Welch's t-test with real p-values (gated digit-for-digit against scipy) |
| `C6Nan.m9` | a NaN in a sample RAISES; skipping gaps is the caller's one visible line |
| `C7Series.m9` | timeseries with resolution and time convention as data; per-column averaging rules, epoch-aligned windows |
| `C8Zarr.m9` | a zarr store over HTTP: checked shapes, NaN fills for deleted chunks, ownership that makes use-after-close uncompilable |
| `C9Plot.m9` | figures as deterministic SVG strings — a plot you can `cmp` |
| `C10Icos.m9` | the capstone: nine years of real ICOS CO2 (Hyltemossa, 150 m) from the Carbon Portal's zarr service — QC by the station's flags, a Fourier fit of trend + seasonal cycle, plotted.  Point it at `https://zarr.icos-cp.eu/icos-obspack.zarr` and it reads the live service; `examples/data/icos-obspack.zarr` is a raw byte mirror of the same arrays for offline runs (ICOS ObsPack, CC BY 4.0, doi:10.18160/JZ2X-GZGU) |

The four `X*.m9` files must **not** compile — each carries an
`EXPECT-ERROR` line naming the diagnostic the compiler must give:
implicit conversion refused, an undeclared RAISES, a signature that
drifts from its definition, a promised procedure never implemented,
a pool-interior pointer escaping the frame that owns it.

`expect/` holds the exact output of every runnable example; the
upstream repository's CI compiles and runs all of them on every
commit and compares byte for byte, so these programs cannot drift
from the compiler you just installed.

Chapter 7 wants a local zarr store to read: any HTTP server over a
zarr v2 directory works, and the example takes the URL as its first
argument.

Chapters 7 and 8 bind C libraries (blosc; the SVG number formatter),
so their build lines name those — see the comments in the files.
