# 8 — Data through zarr

Large scientific arrays live in chunked stores: the array is cut
into fixed-size chunks, each compressed and addressable by its
coordinates, with small JSON metadata describing shape, dtype and
fill.  Zarr is the convention the ICOS Carbon Portal serves its
data with, over plain HTTPS — and `ZarrStore` is a complete reader
for it: metadata parsed and validated at open, blosc decompression,
a chunk cache, and every index checked against the shape the store
itself declared.

The store in this chapter is a local replica of the repository's
test store, and the sandbox that runs these examples online has no
network on purpose -- so the replica is served *inside* the
sandbox, on its own private loopback, started beside your cell and
gone when it exits.  Real chunked data, zero egress, and a tutorial
that cannot fail with the portal's next maintenance window.  Run
locally, serve the store yourself:
`python3 -m http.server 18931 --directory <dir-holding-co2.zarr>`.  Against a live portal the only change is
the URL — the TLS client underneath verifies certificate and
hostname, proven against the standard set of deliberately broken
hosts.

```m9 C8Zarr.m9
MODULE C8Zarr ;

(* Chapter 8.  Zarr over HTTP: a chunked array store is opened by
   URL, sliced by index, and every byte that arrives is checked --
   the shape from .zarray, the compression, the fill value.  The
   store here is local (the sandbox has no network by design); the
   same call with a portal URL is how the ICOS data is read.

   Ownership: Open answers a SHARED handle and Close consumes it --
   the checker refuses a use after Close, so a leaked store is not a
   silent bug but an uncompilable one.                              *)

IMPORT Io ;
IMPORT Fmt ;
IMPORT ZarrStore ;

VAR
  pool : POOL ;
  st   : SHARED PTR ZarrStore.Store ;
  a    : PTR ZarrStore.Array ;
  idx  : ARRAY 2 OF I64 ;
  url  : STR ;
  x    : F64 ;

PROCEDURE At (VAR arr: PTR ZarrStore.Array ; i, j: I64) : F64
  RAISES ZarrStore.IOError =
VAR ix : ARRAY 2 OF I64 ;
BEGIN
  ix [0] := i ;
  ix [1] := j ;
  RETURN ZarrStore.GetF64 (arr, ix)
END At ;

BEGIN
  IF Io.ArgCount () > 1 THEN
    url := Io.Arg (pool, 1)
  ELSE
    url := 'http://127.0.0.1:18931'
  END ;
  st := ZarrStore.Open (url) ;
  a := ZarrStore.OpenArray (st, 'co2.zarr') ;

  Io.Write ('shape ') ;
  Io.WriteI64 (ZarrStore.Extent (a, 0)) ;
  Io.Write (' x ') ;
  Io.WriteI64 (ZarrStore.Extent (a, 1)) ;
  Io.WriteLine ('') ;

  Io.WriteLine ('co2[0,0]   = ' + Fmt.Fixed (At (a, 0, 0), 10)) ;
  Io.WriteLine ('co2[99,49] = ' + Fmt.Fixed (At (a, 99, 49), 10)) ;
  x := At (a, 10, 5) ;
  IF x = x THEN
    Io.WriteLine ('co2[10,5]  = ' + Fmt.Fixed (x, 10))
  ELSE
    Io.WriteLine ('co2[10,5]  is a gap (deleted chunk, NaN fill)')
  END ;

  ZarrStore.CloseArray (a) ;
  ZarrStore.Close (st)
EXCEPT
| ZarrStore.IOError :
    Io.ErrLine ('store unreachable -- is the local server running?') ;
    Io.Halt (1)
| ZarrStore.FormatError (what) :
    Io.ErrLine ('not a zarr v2 store') ;
    Io.Halt (1)
| ValueRange :
    Io.ErrLine ('a value did not fit') ;
    Io.Halt (1)
| IndexError :
    Io.ErrLine ('index out of range') ;
    Io.Halt (1)
END C8Zarr.
```

```output C8Zarr
shape 100 x 50
co2[0,0]   = 394.9816047539
co2[99,49] = 403.8924951332
co2[10,5]  is a gap (deleted chunk, NaN fill)
```

Three things are new here, and each is the language keeping a
promise from an earlier chapter:

- **The gap is DATA.**  Chunk (2,1) of this store was deleted;
  zarr's rule is that a missing chunk reads as the declared fill
  value, NaN here.  The read succeeds, the gap arrives as NaN, and
  chapter 5's `x = x` test spots it.
- **Ownership is in the signature.**  `Open` answers a
  `SHARED PTR Store` — a counted handle — and `Close (OWN s)`
  CONSUMES it: after the `Close` line, any further use of `st` is
  a compile error naming what killed it and where.  The
  use-after-close bug class is not tested away here; it is
  uncompilable.
- **The foreign boundary is declared.**  Blosc and the socket layer
  are C libraries, bound in `FOR "C"` units where every procedure
  states its C name and its thread discipline (`[SERIAL]` /
  `[REENTRANT]`) — the compiler emits the serialisation, so a
  global-state C library cannot be raced by accident.  The honest
  consequence: linking this program names those libraries.  With
  the store served locally (`python3 -m http.server 18931
  --directory /tmp/m9stores`), the build is:

      m9c --make -c C8Zarr.m9
      gcc C8Zarr.o ZarrStore.o Json.o Http.o DynStr.o Io.o Fmt.o \
          m9rt.c tcpshim.c tlsshim.c -l:libblosc.so.1 \
          -lssl -lcrypto -lm -o co2

  `m9c` is deliberately not a build system; what it DOES supply by
  default (include paths, the module closure, the runtime) it will
  show you with `-v`.

[← Previous: timeseries](07-timeseries.md) · [Next: preparing data for plotting →](09-plotting.md)
