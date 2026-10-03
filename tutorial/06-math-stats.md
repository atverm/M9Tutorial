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
IMPORT Faults ;
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
| Faults.BadArg :
    Io.ErrLine ('bad argument') ; Io.Halt (1)
| ValueRange :
    Io.ErrLine ('a sample that does not vary') ; Io.Halt (1)
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
reaches a statistic — and when a sample is nothing but gaps?

```m9 C6Nan.m9
MODULE C6Nan ;

(* Chapter 6.  A NaN in a sample is a MISSING VALUE: the mean of
   [1, NaN, 3] is 2, over the two values that are there -- and the
   library says how many that was.  A skip nobody can see changes n
   behind your back; a skip with its count beside it is a statement
   about your data.  A sample of nothing but gaps is not answered
   at all: it is refused, by that same count.                      *)

IMPORT Io ;
IMPORT Stats ;
IMPORT Fmt ;

VAR
  xs, gaps : ARRAY 3 OF F64 ;
  m : F64 ;
  i : I64 ;

BEGIN
  xs [0] := 1.0 ;
  xs [1] := 0.0 / 0.0 ;             (* a gap, as data has *)
  xs [2] := 3.0 ;
  m := Stats.Mean (xs) ;
  Io.WriteLine ('mean = ' + Fmt.Fixed (m, 3) + ' over ' +
                Fmt.I64Str (Stats.Count (xs)) + ' of ' +
                Fmt.I64Str (LEN (xs)) + ' values') ;
  FOR i := 0 TO 2 DO gaps [i] := 0.0 / 0.0 END ;
  m := Stats.Mean (gaps) ;
  Io.WriteLine ('mean = ' + Fmt.Fixed (m, 3))
EXCEPT
| Stats.TooFew (got, need) :
    Io.WriteLine ('nothing but gaps: ' + Fmt.I64Str (got) +
                  ' values, and a mean needs ' + Fmt.I64Str (need))
| ValueRange :
    Io.WriteLine ('formatting failed')
END C6Nan.
```

```output C6Nan
mean = 2.000 over 2 of 3 values
nothing but gaps: 0 values, and a mean needs 1
```

The gap is skipped, and counted out.  There are two answers a
library can give here that someone regrets: a NaN mean (correct
IEEE, useless science), and the silently "helpful" skip, which
changes n without telling you and turns "a third of my sample is
missing" into a confident narrow confidence interval.  `Stats` gives
neither.  It takes every statistic over the values that are there —
measured series have gaps, and a mean that refuses them is only ever
called behind a filter someone wrote by hand — and it puts the count
in your hands: `Stats.Count (xs)` beside a mean, `n` in the answer of
a regression, a fit or a test, `Stats.RollingCount` beside a rolling
mean.  For two samples taken together a *pair* counts when both its
values are present.  And when nothing is left to count, the answer
is not a NaN but a refusal by name, `TooFew`, carrying the number it
found — the second line above.

So the rule for your own code is one line long: **print n beside
every statistic of measured data.**  The library will always tell
you what it was.

*(Until 0.14 a NaN in a sample raised `ValueRange`, and skipping was
the caller's loop — chapter 5's filter is exactly that loop, and it
still works.  The rule changed because the loop hid the very thing
the refusal was meant to keep in view: what keeps it in view now is
the count.)*

One more time: a checked build runs within a few percent of the
same code with every check stripped.  A language does not have to
choose between honest and fast.

[← Previous: reading and writing data](05-reading-data.md) · [Next: timeseries →](07-timeseries.md)
