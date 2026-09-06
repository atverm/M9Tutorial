# 15. Statistics: a regression, a test, and what a p-value means

Same file as chapter 14, the same two columns, and this time no time
series at all: 336 half-hours of net ecosystem exchange and air
temperature from one July week, asked three questions. Does NEE
depend on temperature? Do warm half-hours differ from cool ones? And
what does the "p" in the answer actually mean — shown by drawing it,
not by quoting it.

```m9 C15Stats.m9
MODULE C15Stats ;

(* Chapter 15: statistics on a week of eddy-covariance data -- a
   regression of net ecosystem exchange on air temperature, a test of
   whether warm and cool half-hours differ, and a thousand draws from
   a normal distribution to show what a p-value actually says.

   Same file as chapter 14 (the shipped week, or a path given on the
   command line); this time only two columns and no time series.  *)

IMPORT Io ;
IMPORT Fmt ;
IMPORT Csv ;
IMPORT Math ;
IMPORT Stats ;

CONST
  Default = 'data/fluxnet_week.csv' ;
  Trials = 1000 ;
  SeedValue = 20260906 ;       (* a fixed seed: the draws are reproducible *)

(* the half-hours where BOTH values are present, as the two F64
   slices the statistics take -- a gap in either drops the pair *)
PROCEDURE Pairs (VAR pool: POOL ; RO ta, nee: SLICE OF F32 ;
                 VAR x, y: SLICE OF F64) RAISES ValueRange =
VAR i, n : I64 ;
BEGIN
  n := 0 ;
  FOR i := 0 TO LEN (ta) - 1 DO
    IF NOT (Math.IsNaNF32 (ta[i]) OR Math.IsNaNF32 (nee[i])) THEN
      n := n + 1
    END
  END ;
  x := NEW (pool, F64, n) ;
  y := NEW (pool, F64, n) ;
  n := 0 ;
  FOR i := 0 TO LEN (ta) - 1 DO
    IF NOT (Math.IsNaNF32 (ta[i]) OR Math.IsNaNF32 (nee[i])) THEN
      x[n] := F64 (ta[i]) ;
      y[n] := F64 (nee[i]) ;
      n := n + 1
    END
  END
END Pairs ;

(* the y values whose x lies below the cut, and those at or above it *)
PROCEDURE Split (VAR pool: POOL ; RO x, y: SLICE OF F64 ; cut: F64 ;
                 VAR lo, hi: SLICE OF F64) =
VAR i, nl, nh : I64 ;
BEGIN
  nl := 0 ; nh := 0 ;
  FOR i := 0 TO LEN (x) - 1 DO
    IF x[i] < cut THEN nl := nl + 1 ELSE nh := nh + 1 END
  END ;
  lo := NEW (pool, F64, nl) ;
  hi := NEW (pool, F64, nh) ;
  nl := 0 ; nh := 0 ;
  FOR i := 0 TO LEN (x) - 1 DO
    IF x[i] < cut THEN lo[nl] := y[i] ; nl := nl + 1
    ELSE hi[nh] := y[i] ; nh := nh + 1 END
  END
END Split ;

PROCEDURE Abs (v: F64) : F64 =
BEGIN
  IF v < 0.0 THEN RETURN -v END ;
  RETURN v
END Abs ;

VAR
  pool : POOL ;
  path : STR ;
  opt : Csv.Options ;
  t : PTR Csv.Table IN pool ;
  cNee, cTa, c, i, k, hits : I64 ;
  ta, nee : SLICE OF F32 ;
  x, y, cool, warm, ysim : SLICE OF F64 ;
  reg, sim : Stats.Reg ;
  tt : Stats.Test ;
  fit : Stats.Fit ;
  st : Stats.Stream ;
  cut, observed : F64 ;
  what : STR ;
  ln : I64 ;

BEGIN
  BEGIN
    path := Default ;
    IF Io.ArgCount () > 1 THEN path := Io.Arg (pool, 1) END ;

    (* ---- the two columns, everything else skipped ---- *)
    opt := Csv.Defaults () ;
    opt.missing := -9999.0 ;
    opt.hasMissing := TRUE ;
    t := Csv.Open (pool, path, opt) ;
    cNee := Csv.Find (t, 'NEE_VUT_REF') ;
    cTa := Csv.Find (t, 'TA_F') ;
    IF (cNee < 0) OR (cTa < 0) THEN
      Io.ErrLine ('the file has no NEE_VUT_REF or no TA_F column') ;
      Io.Halt (1)
    END ;
    Csv.SetReal (t, cNee) ;
    Csv.SetReal (t, cTa) ;
    Csv.Parse (pool, t) ;
    ta := Csv.ColF32 (t, cTa) ;
    nee := Csv.ColF32 (t, cNee) ;
    Pairs (pool, ta, nee, x, y) ;
    Io.WriteLine (Fmt.I64Str (LEN (x)) + ' half-hours with both NEE and TA') ;

    (* ---- 1. the regression: NEE = intercept + slope * TA ---- *)
    reg := Stats.LinReg (x, y) ;
    Io.WriteLine ('') ;
    Io.WriteLine ('NEE on TA, least squares:') ;
    Io.WriteLine ('  slope      ' + Fmt.Fixed (reg.slope, 4)
                  + ' umol m-2 s-1 per degC  (stderr '
                  + Fmt.Fixed (reg.stderr, 4) + ')') ;
    Io.WriteLine ('  intercept  ' + Fmt.Fixed (reg.intercept, 3)
                  + ' umol m-2 s-1') ;
    Io.WriteLine ('  r          ' + Fmt.Fixed (reg.r, 3)
                  + '   r^2 ' + Fmt.Fixed (reg.r * reg.r, 3)) ;
    Io.WriteLine ('  p          ' + Fmt.Sci (reg.p, 2)
                  + '  (two-sided, against slope = 0)') ;

    (* ---- 2. warm half-hours against cool ones: Welch's t ---- *)
    cut := Stats.Median (x) ;
    Split (pool, x, y, cut, cool, warm) ;
    tt := Stats.TTest2 (warm, cool) ;
    Io.WriteLine ('') ;
    Io.WriteLine ('NEE when TA is above its median ' + Fmt.Fixed (cut, 2)
                  + ' degC against below it:') ;
    Io.WriteLine ('  warm mean  ' + Fmt.Fixed (Stats.Mean (warm), 3)
                  + '  (n ' + Fmt.I64Str (LEN (warm)) + ')') ;
    Io.WriteLine ('  cool mean  ' + Fmt.Fixed (Stats.Mean (cool), 3)
                  + '  (n ' + Fmt.I64Str (LEN (cool)) + ')') ;
    Io.WriteLine ('  t ' + Fmt.Fixed (tt.t, 2) + '  dof '
                  + Fmt.Fixed (tt.dof, 1) + '  p ' + Fmt.Sci (tt.p, 2)) ;

    (* ---- 3. what the p-value means, by drawing it ----
       Fit a normal to NEE, then make a thousand series with that mean
       and spread but NO relation to TA, and regress each one.  The
       fraction whose |slope| reaches the real one is a p-value
       obtained by counting instead of by a formula. *)
    fit := Stats.NormFit (y) ;
    st := Stats.Seed (SeedValue) ;
    Io.WriteLine ('') ;
    Io.WriteLine ('normal fit of NEE: mu ' + Fmt.Fixed (fit.mu, 3)
                  + ', sigma ' + Fmt.Fixed (fit.sigma, 3)) ;
    Io.Write ('three draws from it:') ;
    FOR i := 1 TO 3 DO
      Io.Write (' ' + Fmt.Fixed (fit.mu + fit.sigma * Stats.Normal (st), 3))
    END ;
    Io.WriteLine ('') ;

    observed := Abs (reg.slope) ;
    ysim := NEW (pool, F64, LEN (y)) ;
    hits := 0 ;
    FOR k := 1 TO Trials DO
      FOR i := 0 TO LEN (ysim) - 1 DO
        ysim[i] := fit.mu + fit.sigma * Stats.Normal (st)
      END ;
      sim := Stats.LinReg (x, ysim) ;
      IF Abs (sim.slope) >= observed THEN hits := hits + 1 END
    END ;
    Io.WriteLine ('in ' + Fmt.I64Str (Trials)
                  + ' series with that mean and spread and no relation to TA,'
                  + ' |slope| reached ' + Fmt.Fixed (observed, 4) + ': '
                  + Fmt.I64Str (hits) + ' times')
  EXCEPT
  | Csv.ParseError (whatP, lineP, colP) :
      what := whatP ; ln := lineP ;
      Io.ErrLine ('CSV: ' + what + ' at line ' + Fmt.I64Str (ln)) ;
      Io.Halt (1)
  | Csv.RangeError : Io.ErrLine ('CSV: a value out of range') ; Io.Halt (1)
  | Io.IOError : Io.ErrLine ('cannot read ' + path) ; Io.Halt (1)
  | Stats.TooFew : Io.ErrLine ('too few values for that statistic') ; Io.Halt (1)
  | Stats.BadArg : Io.ErrLine ('a statistic refused its argument') ; Io.Halt (1)
  | ValueRange : Io.ErrLine ('a value out of range') ; Io.Halt (1)
  END
END C15Stats.
```

## The regression

`Stats.LinReg` answers the five numbers `scipy.stats.linregress`
answers, and the repository holds it to scipy digit for digit. In
July, at this forest, every extra degree of air temperature moves NEE
by about −1.08 µmol m⁻² s⁻¹ — more negative, which in the flux
convention is more uptake: warm hours are bright hours and the canopy
is photosynthesising. Temperature explains about a fifth of the
variance (r² ≈ 0.19); the rest is light, humidity, the time of day
and the hour-to-hour noise of a turbulent measurement. The p-value,
3.5 × 10⁻¹⁷, says a slope this large is not something 336 unrelated
points produce by accident.

`Pairs` is the small procedure that makes this honest: it keeps only
the half-hours where *both* values are present, so a gap in either
column drops the pair rather than becoming a zero. This week has no
gaps; the full file does, and the program is written for the full
file.

## Warm against cool

`Stats.Median` splits the week at 15.08 °C, and `Stats.TTest2` — the
Welch form, which does not assume the two halves have the same spread
— compares NEE above and below it. The means are −7.9 and +1.4: warm
half-hours are net uptake, cool ones (the nights) net release. The
statistic is t ≈ −10 on about 300 degrees of freedom, p ≈ 10⁻²⁰. The
`dof` is fractional because Welch's correction makes it so; the
program prints it with one decimal for exactly that reason.

## What a p-value means, by counting

The last part is the one worth the chapter. `Stats.NormFit` fits a
normal distribution to NEE (mean −3.23, spread 9.69). `Stats.Seed`
starts a random stream from a fixed seed, and `Stats.Normal` draws
from it — reproducibly, so the three draws printed are the same three
on your machine, and the repository holds the stream bit for bit to
an independent implementation.

Then the program makes a thousand *fake* weeks: each has NEE drawn
from that normal — the right mean and spread — with no relation to
temperature whatsoever, and each is regressed on the real
temperatures. The question is how often a fake week produces a slope
as steep as the real one. The answer is zero times in a thousand.
That is a p-value obtained by counting, and it agrees with the
formula's 3.5 × 10⁻¹⁷ in the only way it can: you would need
something like 10¹⁷ fake weeks to see one.

```output C15Stats
336 half-hours with both NEE and TA

NEE on TA, least squares:
  slope      -1.0798 umol m-2 s-1 per degC  (stderr 0.1213)
  intercept  13.651 umol m-2 s-1
  r          -0.438   r^2 0.192
  p          3.54e-17  (two-sided, against slope = 0)

NEE when TA is above its median 15.08 degC against below it:
  warm mean  -7.908  (n 168)
  cool mean  1.446  (n 168)
  t -10.08  dof 303.5  p 8.42e-21

normal fit of NEE: mu -3.231, sigma 9.685
three draws from it: -7.926 10.014 -13.524
in 1000 series with that mean and spread and no relation to TA, |slope| reached 1.0798: 0 times
```

## The caveat a reviewer will raise

Both p-values assume the 336 half-hours are independent, and they
are not: a half-hour's NEE resembles the last half-hour's. The
formula assumes independence; the counted version inherits the same
assumption, because the fake weeks are white noise. The honest
reading is that the effect is real and large, and that the exact
number of zeros in the p-value overstates the evidence. A proper
treatment thins the series or models the autocorrelation — which is
a different chapter, and the point of this one is that the program
states what it assumed, so the reviewer can see it.
