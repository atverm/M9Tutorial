# Scientific programming in Modula-9 — first steps

M9 is **not designed for human convenience**.  It is designed to be
verbose, clear, reliable and auditable.  You get exactly what is
written, and when what is written is wrong, the compiler tells you
and refuses to run it.  That makes it a good language for code
written by AI agents, a good language for human programmers — and
the best language we know how to build for the **human reviewer**,
who has to certify work she did not write.  The extra effort of
stating clearly what you want, in the code, pays off enormously
later: in the review, in the re-run, in the audit two years on.

Every feature in the language cites a real failure it makes
uncompilable — the failures live in the repository as programs that
must not compile.  This tutorial works the same way: **every example
is a complete program**, compiled and executed by the repository's
own gate (`runtime/test/tutdiff.sh`), its output compared byte for
byte with what these pages print.  The refusals are real too: the
examples named `X*` MUST fail to compile, with the diagnostic the
text quotes.  A tutorial that cannot disagree with the compiler is
the only kind worth reading.

## Chapters

0. [Installing and running the compiler](00-install.md)
1. [Hello, and why M9 exists](01-hello.md)
2. [Strong typing](02-strong-typing.md)
3. [Definition and implementation](03-definition-implementation.md)
4. [Memory: pools, slices and strings](04-memory.md)
5. [Reading and writing data](05-reading-data.md)
6. [Simple math and statistics](06-math-stats.md)
7. [Timeseries](07-timeseries.md)
8. [Data through zarr](08-zarr.md)
9. [Preparing data for plotting](09-plotting.md)
10. [A real dataset, end to end](10-real-data.md)
11. [Threads: waiting in parallel](11-threads.md)

## Running the examples yourself

Chapter 0 covers installation (a Debian package, or `./build.sh`
from source with nothing but gcc) and the VS Code extension.  The
examples and the package are also public at
[github.com/atverm/M9Tutorial](https://github.com/atverm/M9Tutorial).
With the compiler installed:

    cd docs/tutorial/examples
    m9c --make -o hello C1Hello.m9   # compile, resolving imports
    ./hello

Chapters 8 and 9 bind C libraries (blosc for zarr; the SVG formatter
shim), so their build lines name those — each chapter shows its own.
