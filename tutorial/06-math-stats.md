# 6 — Simple math and statistics

`Stats` is the library this project uses for its own science, and
its numbers come with a pedigree: every procedure is gated against
numpy and scipy — the p-values digit for digit — with the expected
values checked into the repository, because a gate that regenerates
what it compares against cannot fail.  When the tutorial prints a
p-value below, that number has an oracle behind it.

```m9 C6Stats.m9
MODULE C6Stats ;

(* Chapter 6.  Statistics with real p-values, verified against
   scipy digit for digit in the library's own gate -- which is why a
   tutorial can print them without hedging.  Two synthetic samples:
   y depends on x almost linearly, and the two groups differ.       *)

IMPORT Io ;
IMPORT Fmt ;
IMPORT Stats ;

PROCEDURE P (RO label: STR ; v: F64 ; dec: I64) =
BEGIN
  Io.Write (label) ;
  Io.Write (Fmt.Fixed (v, dec)) ;
  Io.WriteLine ('')
EXCEPT
| ValueRange :
    Io.ErrLine ('formatting failed') ;
    Io.Halt (1)
END P ;

VAR
  x, y : ARRAY 8 OF F64 ;
  a, b : ARRAY 6 OF F64 ;
  i    : I64 ;
  reg  : Stats.Reg ;
  tst  : Stats.Test ;

BEGIN
  FOR i := 0 TO 7 DO
    x [i] := F64 (i) ;
    y [i] := 2.0 + 0.5 * F64 (i)
  END ;
  y [3] := y [3] + 0.4 ;             (* one imperfect point *)
  P ('mean y    ', Stats.Mean (y), 4) ;
  P ('std y     ', Stats.Std (y), 4) ;
  P ('median y  ', Stats.Median (y), 4) ;
  P ('p90 y     ', Stats.Percentile (y, 90.0), 4) ;

  reg := Stats.LinReg (x, y) ;
  P ('slope     ', reg.slope, 4) ;
  P ('intercept ', reg.intercept, 4) ;
  P ('r         ', reg.r, 4) ;
  P ('p         ', reg.p, 8) ;

  a [0] := 5.1 ; a [1] := 4.9 ; a [2] := 5.3 ;
  a [3] := 5.0 ; a [4] := 5.2 ; a [5] := 4.8 ;
  FOR i := 0 TO 5 DO
    b [i] := a [i] + 0.6                (* shifted group *)
  END ;
  tst := Stats.TTest2 (a, b) ;
  P ('Welch t   ', tst.t, 4) ;
  P ('p         ', tst.p, 6)
EXCEPT
| Stats.TooFew :
    Io.ErrLine ('sample too small') ; Io.Halt (1)
| Stats.BadArg :
    Io.ErrLine ('bad argument') ; Io.Halt (1)
| ValueRange :
    Io.ErrLine ('a NaN reached the statistics') ; Io.Halt (1)
| Overflow :
    Io.ErrLine ('overflow') ; Io.Halt (1)
END C6Stats.
```

```output C6Stats
mean y    3.8000
std y     1.2212
median y  3.9500
p90 y     5.1500
slope     0.4952
intercept 2.0667
r         0.9933
p         0.00000074
Welch t   -5.5549
p         0.000242
```

What the library gives you, in the order a working analysis meets
them: moments (`Mean`, `Std`, and the population forms), order
statistics (`Median`, `Percentile` — numpy's linear interpolation
rule, so your quartiles match your colleague's notebook), a normal
fit, least-squares regression with scipy.linregress's five numbers,
and both t-tests (`TTest2` is Welch's — unequal variances assumed,
which is the safe default for real measurements).  The p-values are
real two-sided probabilities computed through the incomplete beta
function, not lookup-table approximations.

For reproducible synthetic data, `Stats.Seed` gives a deterministic
`Stream` with uniform, integer, normal, exponential and log-normal
draws — bit-for-bit reproducible, seed in, same sequence out, on
every machine.

## The NaN policy, and why it is a policy

Chapter 5 turned a declared gap into NaN.  What happens when a NaN
reaches a statistic?

```m9 C6Nan.m9
MODULE C6Nan ;

(* Chapter 6.  A NaN in a sample RAISES -- the mean of [1, NaN, 3]
   is not a number, and a library that answers one anyway (or
   silently skips the gap) has decided your science for you.  The
   skipping, when you mean it, is the caller's one visible line.    *)

IMPORT Io ;
IMPORT Stats ;
IMPORT Fmt ;

VAR
  xs : ARRAY 3 OF F64 ;
  m  : F64 ;

BEGIN
  xs [0] := 1.0 ;
  xs [1] := 0.0 / 0.0 ;             (* a gap, as data has *)
  xs [2] := 3.0 ;
  m := Stats.Mean (xs) ;
  Io.WriteLine ('mean = ' + Fmt.Fixed (m, 3))
EXCEPT
| ValueRange :
    Io.WriteLine ('Stats.Mean refused the NaN (ValueRange)')
| Stats.TooFew :
    Io.WriteLine ('sample too small')
END C6Nan.
```

```output C6Nan
Stats.Mean refused the NaN (ValueRange)
```

It raises.  The alternatives are both answers someone regrets: a
NaN mean (correct IEEE, useless science) or the silently "helpful"
skip, which changes n without telling you and turns "a third of my
sample is missing" into a confident narrow confidence interval.
Skipping is often what you want — chapter 5's filter loop is
exactly that — but it must be the CALLER's visible decision, with
the count in the caller's hands.  A library that decides for you
has decided your science.

One more time: a checked build runs within a few percent of the
same code with every check stripped.  A language does not have to
choose between honest and fast.

[← Previous: reading and writing data](05-reading-data.md) · [Next: timeseries →](07-timeseries.md)
