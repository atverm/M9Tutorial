# Scientific programming in Modula-9 — first steps

M9 is a Wirth-family language for scientific computing, designed so
the result can be **trusted and reproduced** — by your reviewer, by
your future self, by whoever inherits the code.  You get exactly
what is written, and when what is written is wrong, the compiler
tells you and refuses to run it, rather than letting the program go
quietly on and publish a number no one can reproduce.  The extra
effort of stating clearly what you want, in the code, pays off later
where it matters most: in the review, in the re-run, in the audit
two years on.

Every feature in the language cites a real failure it makes
uncompilable — the failures live in the repository as programs that
must not compile.  This tutorial works the same way: **every example
is a complete program**, compiled and executed by the repository's
own gate (`runtime/test/tutdiff.sh`), its output compared byte for
byte with what these pages print.  The refusals are real too: the
examples named `X*` MUST fail to compile, with the diagnostic the
text quotes.  A tutorial that cannot disagree with the compiler is
the only kind worth reading.

These pages teach the language by using it. The language itself is
defined in one place — **[the M9 report](https://github.com/atverm/m9c/blob/main/docs/M9-report.md)**, the specification,
where every rule is stated with the failure that forced it. This
tutorial never contradicts it; when you want the rule rather than the
worked example, that is the document to open.

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
12. [Building strings](12-building-strings.md)
13. [Procedures: calls, returns, and the parameter modes](13-procedures-modes.md)
14. [A big CSV file, and a CF NetCDF file from two of its columns](14-big-csv.md)
15. [Statistics: a regression, a test, and what a p-value means](15-statistics.md)
16. [Threads for computation, and what the cores actually give you](16-threads-compute.md)
17. [Living in the real world: parameters, external programs, and pipelines](17-real-world.md)

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
