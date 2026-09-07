# 14. A big CSV file, and a CF NetCDF file from two of its columns

The file is real: ICOS's FLUXNET half-hourly product for the
Hyltemossa forest station in Sweden (SE-Htm), 244 columns, one row
every half hour from January 2018 to September 2025 — 135,840 rows,
186 MB. We want two of those columns, `NEE_VUT_REF` (the net ecosystem
exchange of CO₂) and `TA_F` (air temperature), as a time series in a
NetCDF file that any CF-aware tool will open correctly.

The program below reads the whole file, keeps those two columns and
the timestamp, writes the NetCDF, reads it back, and proves the round
trip lost nothing. Given no argument it reads a one-week excerpt
shipped beside it (3–9 July 2023, 336 rows, the same 244 columns);
given a path it reads that. The timings in this chapter are from the
full file.

```m9 C14Flux.m9
MODULE C14Flux ;

(* Chapter 14: a big CSV file, read in a flash, and two of its columns
   written to a CF-compliant NetCDF file as a time series.

   The file is ICOS's FLUXNET half-hourly product for Hyltemossa
   (SE-Htm): 244 columns, one row per half hour.  Given no argument
   the program reads the one-week excerpt shipped beside it; given a
   path it reads that instead -- the full 186 MB, 135,840-row file is
   what the chapter's timings are measured on.

   Three things this program says out loud that a script would leave
   implicit: which columns it wants and what they ARE (the other 241
   are never parsed), what value means "missing", and that the
   timestamps are LOCAL STANDARD TIME, one hour ahead of UTC.       *)

IMPORT Io ;
IMPORT Fmt ;
IMPORT Csv ;
IMPORT Frame ;
IMPORT Time ;
IMPORT Math ;
IMPORT Stats ;

CONST
  Default = 'data/fluxnet_week.csv' ;
  OutPath = 'sehtm.nc' ;
  HalfHour = 1800 ;                   (* the file's resolution, seconds *)
  Utc1 = 3600.0 ;                     (* SE-Htm stamps are UTC+1 *)

(* the values that are actually there: NaN is the missing value, and
   a mean over a column with gaps is a mean over what is present *)
PROCEDURE Present (VAR pool: POOL ; RO v: SLICE OF F32) : SLICE OF F64
  RAISES ValueRange =
VAR
  out : SLICE OF F64 ;
  i, n : I64 ;
BEGIN
  n := 0 ;
  FOR i := 0 TO LEN (v) - 1 DO
    IF NOT Math.IsNaNF32 (v[i]) THEN n := n + 1 END
  END ;
  out := NEW (pool, F64, n) ;
  n := 0 ;
  FOR i := 0 TO LEN (v) - 1 DO
    IF NOT Math.IsNaNF32 (v[i]) THEN
      out[n] := F64 (v[i]) ;
      n := n + 1
    END
  END ;
  RETURN out
END Present ;

(* two columns are the same column when every value is the same
   value -- and a gap equals a gap, which `=` on NaN never says *)
PROCEDURE Same (RO a, b: SLICE OF F32) : BOOL =
VAR i : I64 ;
BEGIN
  IF LEN (a) # LEN (b) THEN RETURN FALSE END ;
  FOR i := 0 TO LEN (a) - 1 DO
    IF Math.IsNaNF32 (a[i]) THEN
      IF NOT Math.IsNaNF32 (b[i]) THEN RETURN FALSE END
    ELSIF a[i] # b[i] THEN
      RETURN FALSE
    END
  END ;
  RETURN TRUE
END Same ;

PROCEDURE Stamp (VAR pool: POOL ; sec: I64) : STR RAISES ValueRange =
VAR t : Time.Instant ;
BEGIN
  t.t := F64 (sec) ;
  RETURN Time.Iso (pool, t, 0)
END Stamp ;

VAR
  pool : POOL ;
  path : STR ;
  opt : Csv.Options ;
  t : PTR Csv.Table IN pool ;
  cNee, cTa, c, parsed : I64 ;
  t0, t1 : Time.Instant ;
  fr : PTR Frame.Fr IN pool ;
  ts, back : PTR Frame.Ts IN pool ;
  stamps : SLICE OF I64 ;     (* not `time`: that name is libc's *)
  nee, ta : SLICE OF F32 ;
  neeOk, taOk : SLICE OF F64 ;
  (* a handler's payload binders carry no type into the checker yet,
     so they are copied into declared locals before they meet `+` *)
  what, op, detail : STR ;
  ln, st : I64 ;

BEGIN
  BEGIN
    path := Default ;
    IF Io.ArgCount () > 1 THEN path := Io.Arg (pool, 1) END ;

    (* ---- 1. the CSV: say what the columns are, then parse ---- *)
    opt := Csv.Defaults () ;
    opt.missing := -9999.0 ;          (* FLUXNET's gap marker *)
    opt.hasMissing := TRUE ;
    opt.utcOffset := Utc1 ;           (* subtracted from every stamp *)
    t := Csv.Open (pool, path, opt) ;

    Csv.SetStamp (t, 0, Csv.StampYmdHm) ;      (* TIMESTAMP_START *)
    cNee := Csv.Find (t, 'NEE_VUT_REF') ;
    cTa := Csv.Find (t, 'TA_F') ;
    IF (cNee < 0) OR (cTa < 0) THEN
      Io.ErrLine ('the file has no NEE_VUT_REF or no TA_F column') ;
      Io.Halt (1)
    END ;
    (* every OTHER column stays Skip -- scanned past, never parsed,
       never carried.  Nothing needs saying about the 241 of them:
       a column nobody declared is a column nobody gets. *)
    Csv.SetReal (t, cNee) ;
    Csv.SetReal (t, cTa) ;

    t0 := Time.Now () ;
    Csv.Parse (pool, t) ;
    t1 := Time.Now () ;
    (* the timing is reported only for a file named on the command
       line: the shipped week parses in a millisecond, and a number
       that changes from run to run has no place in gated output *)
    IF Io.ArgCount () > 1 THEN
      Io.ErrLine ('parsed ' + Fmt.I64Str (Csv.Rows (t)) + ' rows in '
                  + Fmt.Fixed (Time.Elapsed (t0, t1), 3) + ' s')
    END ;

    (* counted, not asserted: the columns that were actually parsed *)
    parsed := 0 ;
    FOR c := 0 TO Csv.Cols (t) - 1 DO
      IF Csv.KindCodeAt (t, c) # Csv.KindSkip THEN parsed := parsed + 1 END
    END ;
    Io.WriteLine ('rows ' + Fmt.I64Str (Csv.Rows (t))
                  + ', columns in the file ' + Fmt.I64Str (Csv.Cols (t))
                  + ', parsed ' + Fmt.I64Str (parsed)) ;

    (* ---- 2. a frame, then a time series on the half-hour grid ---- *)
    fr := Frame.FromCsv (pool, t) ;
    stamps := Frame.ColI64 (fr, 'TIMESTAMP_START') ;
    Io.WriteLine ('from ' + Stamp (pool, stamps[0]) + ' to '
                  + Stamp (pool, stamps[LEN (stamps) - 1]) + ' UTC') ;

    (* what a CF reader will find beside each variable *)
    Frame.SetMeta (pool, fr, 'NEE_VUT_REF',
                   'net ecosystem exchange of CO2, VUT reference',
                   'surface_upward_mole_flux_of_carbon_dioxide',
                   'umol m-2 s-1') ;
    Frame.SetMeta (pool, fr, 'TA_F',
                   'air temperature, gap-filled',
                   'air_temperature', 'degC') ;

    ts := Frame.NewTs (pool, fr, stamps, HalfHour, Frame.ConvStart (),
                       'SE-Htm FLUXNET INTERIM half-hourly, L2') ;

    (* ---- 3. what is in the two columns ---- *)
    nee := Frame.ColF32 (fr, 'NEE_VUT_REF') ;
    ta := Frame.ColF32 (fr, 'TA_F') ;
    neeOk := Present (pool, nee) ;
    taOk := Present (pool, ta) ;
    Io.WriteLine ('NEE_VUT_REF: ' + Fmt.I64Str (LEN (neeOk)) + ' of '
                  + Fmt.I64Str (LEN (nee)) + ' present, mean '
                  + Fmt.Fixed (Stats.Mean (neeOk), 3) + ' umol m-2 s-1') ;
    Io.WriteLine ('TA_F:        ' + Fmt.I64Str (LEN (taOk)) + ' of '
                  + Fmt.I64Str (LEN (ta)) + ' present, mean '
                  + Fmt.Fixed (Stats.Mean (taOk), 3) + ' degC') ;

    (* ---- 4. write the NetCDF, then read it back and compare ---- *)
    Frame.WriteTsNc (pool, ts, OutPath) ;
    back := Frame.TsFromNc (pool, OutPath) ;
    Io.WriteLine ('wrote ' + OutPath + ', read back '
                  + Fmt.I64Str (Frame.Rows (Frame.TsFrame (back))) + ' rows') ;
    IF Same (nee, Frame.ColF32 (Frame.TsFrame (back), 'NEE_VUT_REF'))
       AND Same (ta, Frame.ColF32 (Frame.TsFrame (back), 'TA_F')) THEN
      Io.WriteLine ('both columns identical after the round trip: yes')
    ELSE
      Io.WriteLine ('both columns identical after the round trip: NO')
    END
  EXCEPT
  | Csv.ParseError (whatP, lineP, colP) :
      what := whatP ; ln := lineP ;
      Io.ErrLine ('CSV: ' + what + ' at line ' + Fmt.I64Str (ln)) ;
      Io.Halt (1)
  | Csv.RangeError : Io.ErrLine ('CSV: a value out of range') ; Io.Halt (1)
  | Io.IOError : Io.ErrLine ('cannot read ' + path) ; Io.Halt (1)
  | Frame.SizeError : Io.ErrLine ('frame: size') ; Io.Halt (1)
  | Frame.Duplicate : Io.ErrLine ('frame: duplicate column') ; Io.Halt (1)
  | Frame.Disorder : Io.ErrLine ('frame: time not increasing') ; Io.Halt (1)
  | Frame.BadArg : Io.ErrLine ('frame: bad argument') ; Io.Halt (1)
  | Frame.Unknown : Io.ErrLine ('frame: no such column') ; Io.Halt (1)
  | Frame.WrongType : Io.ErrLine ('frame: wrong column type') ; Io.Halt (1)
  | NetCDF.Error (opP, detailP, statusP) :
      (* op, the library's own message, and its status NUMBER -- the
         checker does not verify a handler's payload order, so it is
         read off the EXCEPTION declaration, not remembered *)
      op := opP ; detail := detailP ; st := statusP ;
      Io.ErrLine ('netcdf: ' + op + ': ' + detail + ' (status '
                  + Fmt.I64Str (st) + ')') ;
      Io.Halt (1)
  | Stats.TooFew : Io.ErrLine ('nothing present to average') ; Io.Halt (1)
  | ValueRange : Io.ErrLine ('a value out of range') ; Io.Halt (1)
  END
END C14Flux.
```

## What it says out loud

A script would do this in four lines and leave three things implicit.
M9 makes you state them, and each one is a place the implicit version
goes wrong.

**Which columns, and what they are.** `Csv.SetStamp` for the
timestamp, `Csv.SetReal` for the two values, and nothing at all for
the other 241: a column nobody declares stays `Skip`, scanned past
and never parsed. That default is not a convenience, it is a
promise, and the promise was broken once: `Csv.Open` used to start
every column as a real number whatever the definition said, and the
first version of this program found out by writing a **136 MB**
NetCDF file with 246 variables in it. The default is what the
definition says now, and the file is 5.4 MB and holds exactly
`time`, `time_bnds`, `TIMESTAMP_START`, `TA_F` and `NEE_VUT_REF`.
The `parsed 3` on the
first output line is *counted* from the table after the declarations,
not asserted.

**What "missing" means.** FLUXNET writes `-9999`. `opt.missing` says
so, and from then on a gap is a NaN — which is what `Present` tests
for before averaging, and what `Same` treats as equal to itself when
comparing the round trip, because `NaN = NaN` is false and a naive
comparison would report every gap as a difference.

**What time zone the stamps are in.** FLUXNET timestamps are *local
standard time*, one hour ahead of UTC for Sweden, and the file does
not say so anywhere. `opt.utcOffset := 3600.0` subtracts the hour from
every stamp as it is parsed, so the series is UTC before anything
downstream sees it. The first line of output shows the consequence:
the week that starts at `202307030000` in the file starts at
`2023-07-02T23:00:00Z` in the series. Chapter 10 met the same trap
from the other side — a CF time axis with no zone that *was not* UTC
— and the honest fix is to convert on the way in, once, and state it.

## In a flash

Parsing the full 186 MB with three columns declared takes **0.345 s**
on this laptop — 565 MB per second, and about 130 MB/s of that is the
part that actually parses numbers. With all 244 columns parsed it is
0.78 s. `Csv` reads the file in one piece and scans it in place; the
time is the scan, and skipping a column costs only the scan.

The NetCDF write and the read-back are `Frame.WriteTsNc` and
`Frame.TsFromNc`, and the `SetMeta` calls before them are what make
the file CF: a `standard_name` and a unit per variable, on top of the
time axis, its bounds and the resolution `NewTs` was told. The
repository's own gate runs `cfchecks` over files written this way and
requires zero errors.

```output C14Flux
rows 336, columns in the file 244, parsed 3
from 2023-07-02T23:00:00Z to 2023-07-09T22:30:00Z UTC
NEE_VUT_REF: 336 of 336 present, mean -3.231 umol m-2 s-1
TA_F:        336 of 336 present, mean 15.634 degC
wrote sehtm.nc, read back 336 rows
both columns identical after the round trip: yes
```

## What to notice

Every value survives the round trip — both columns are compared
element by element after reading the file back, and the answer is
`yes`. That is a cheap check to write and it is the one that matters:
a file format is a promise about what comes back, not about what went
in.

This chapter's program links `libnetcdf`, which the repository's
continuous integration does not have; the gate skips it out loud
there rather than passing quietly, and runs it everywhere the library
exists.
