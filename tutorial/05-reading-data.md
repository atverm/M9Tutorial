# 5 — Reading and writing data

Scientific data arrives as text files with conventions, and the
conventions are facts about the FILE, not properties a reader
should guess.  M9's CSV reader refuses to infer: the caller
declares each column's kind — real, integer, timestamp with a named
layout, text, or skip — and declares the file's missing-value
convention.  For comparison: handed a real flux-network file, a
fast modern dataframe library scanned the first thousand rows,
decided a column was integer, and stopped a hundred kilobytes in
when `-2.03` arrived.  The column was never integer; the file just
opens with round numbers.  Inference is a bet about the rows you
have not read yet.

The data for this chapter is a half-hour temperature series with
one gap, marked `-9999` — the file's own convention, stated in its
documentation, declared by the caller:

```csv data/obs.csv
TIMESTAMP,TA
202512010000,3.2
202512010030,3.0
202512010100,2.7
202512010130,-9999
202512010200,2.1
202512010230,1.8
202512010300,1.6
202512010330,1.7
```

```m9 C5Csv.m9
MODULE C5Csv ;

(* Chapter 5.  Reading data: the caller DECLARES what each column
   is; nothing is inferred from the first thousand rows.  The missing
   value is stated too, and becomes NaN -- which floats carry
   honestly and conversions refuse.

   The pool rule (docs/pools.md): storage that outlives a call takes
   the pool as a parameter, so "who frees this?" is answered in the
   signature.  This whole program uses one pool that dies with it.  *)

IMPORT Io ;
IMPORT Fmt ;
IMPORT Csv ;
IMPORT Time ;
IMPORT Stats ;

VAR
  pool  : POOL ;
  opt   : Csv.Options ;
  t     : PTR Csv.Table ;
  temp  : SLICE OF F32 ;
  stamp : SLICE OF Time.Instant ;
  buf   : ARRAY 64 OF F64 ;
  n, i  : I64 ;
  x     : F64 ;

BEGIN
  opt := Csv.Defaults () ;
  opt.missing := -9999.0 ;         (* the file's own convention *)
  opt.hasMissing := TRUE ;
  t := Csv.Open (pool, 'data/obs.csv', opt) ;
  Csv.SetStamp (t, 0, Csv.StampYmdHm) ;
  Csv.SetReal (t, 1) ;
  Csv.Parse (pool, t) ;

  stamp := Csv.ColStamp (t, 0) ;
  temp := Csv.ColF32 (t, 1) ;
  n := 0 ;
  FOR i := 0 TO LEN (temp) - 1 DO
    x := F64 (temp [i]) ;          (* widths convert EXPLICITLY *)
    IF x = x THEN                  (* NaN is the one value # itself *)
      buf [n] := x ;
      n := n + 1
    END
  END ;

  Io.Write ('rows ') ;
  Io.WriteI64 (Csv.Rows (t)) ;
  Io.Write (', valid ') ;
  Io.WriteI64 (n) ;
  Io.WriteLine ('') ;
  Io.Write ('window ') ;
  Io.Write (Time.Iso (pool, stamp [0], 0)) ;
  Io.Write (' .. ') ;
  Io.WriteLine (Time.Iso (pool, stamp [LEN (stamp) - 1], 0)) ;
  Io.Write ('mean TA ') ;
  Io.Write (Fmt.Fixed (pool, Stats.Mean (SLICE (buf, 0, n)), 3)) ;
  Io.WriteLine (' degC')
EXCEPT
| Csv.ParseError (msg, line, col) :
    Io.ErrLine ('CSV refused:') ;
    Io.ErrLine (msg) ;
    Io.Halt (1)
| Csv.RangeError (row, col) :
    Io.ErrLine ('CSV value out of range') ;
    Io.Halt (1)
| Io.IOError :
    Io.ErrLine ('cannot read data/obs.csv (run from examples/)') ;
    Io.Halt (1)
| ValueRange :
    Io.ErrLine ('a value did not fit') ;
    Io.Halt (1)
| IndexError :
    Io.ErrLine ('index out of range') ;
    Io.Halt (1)
| IndexError :
    Io.ErrLine ('index out of range') ;
    Io.Halt (1)
| Stats.TooFew :
    Io.ErrLine ('no valid readings') ;
    Io.Halt (1)
END C5Csv.
```

```output C5Csv
rows 8, valid 7
window 2025-12-01T00:00:00Z .. 2025-12-01T03:30:00Z
mean TA 2.300 degC
```

Walk the pipeline:

- **`opt.missing := -9999.0`** — the declared gap value becomes NaN
  on parse.  NaN is the honest in-band representation for "no
  measurement": floats carry it, every statistic refuses it
  (chapter 6), and the one comparison it satisfies — `x # x`,
  since NaN is the only value not equal to itself — makes the
  valid-data filter a single visible line.
- **`Csv.SetStamp (t, 0, Csv.StampYmdHm)`** — the timestamp layout
  is named, not sniffed, and parses into `Time.Instant`: seconds
  since the Unix epoch, UTC.  A time axis that is secretly local
  time is an hour of silent error for every user who assumed
  otherwise; time zones are facts, so the API makes them explicit.
- **`F64 (temp [i])`** — the file's reals are F32 (they have seven
  significant digits at most); statistics run in F64.  The widening
  is written, because chapter 2.
- **`Fmt.Fixed (pool, …, 3)`** — formatting allocates, so it takes
  the pool.  Which brings us to the last idea.

## The pool answers "who frees this?"

Every M9 allocation is carved from a named POOL, and a procedure
that keeps storage beyond its own frame takes the pool as a
parameter.  The rule of thumb, measured across this repository's
whole library: **the pool parameter appears exactly when the
allocation outlives the call.**  A signature with a pool is telling
you the result lives on and who owns it; a signature without one is
a promise that nothing was kept.  This program declares one pool at
the top; everything — the parsed table, the formatted strings —
dies with the program, and there is no free() to forget and no
garbage collector to wonder about.

On the writing side, `Io` deals in whole files (`WriteFile`,
`ReadFile`): a partial-read API is an invitation to the truncation
bugs this repository has already paid for, so the boundary is one
call, checked.

[← Previous: memory, pools and strings](04-memory.md) · [Next: simple math and statistics →](06-math-stats.md)
