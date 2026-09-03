# 7 — Timeseries

A column of numbers with timestamps is not yet a timeseries.  Three
facts are usually lost between the instrument and the analysis: the
RESOLUTION (is this half-hourly data, or irregular samples?), the
CONVENTION (does 01:00 label the period that starts then, ends
then, or is centred on it?), and the ZONE.  `Frame` makes all three
part of the data.  A `Ts` is a frame plus its time axis, its
resolution in seconds, and its start/end/mid convention; time is
I64 seconds since the epoch, UTC only — a frame cannot be built
from local-time stamps, because the hour that repeats every autumn
is not a bug you want in your averaging windows.

The frame itself is typed per column — F64, F32, I64, I32, I16,
BYTE, BOOL, text — and each numeric column carries its own missing
value inside the type.  (A BOOL column carries none, deliberately:
a null that silently became FALSE is the exact lie a flags column
must refuse, so gaps in booleans are an error with the column
named.)

## Averaging, honestly

```m9 C7Series.m9
MODULE C7Series ;

(* Chapter 7.  A timeseries is a frame plus facts most tables lose:
   the resolution, whether a stamp names the START, END or MIDDLE of
   its period, and UTC-only time.  Averaging states HOW per column
   -- a mean of the timestamps would be nonsense, so the caller says
   First there and Mean for the data, and the windows are aligned to
   the epoch exactly as polars' group_by_dynamic aligns them.       *)

IMPORT Io ;
IMPORT Fmt ;
IMPORT Csv ;
IMPORT Time ;
IMPORT Frame ;

VAR
  pool   : POOL ;
  opt    : Csv.Options ;
  t      : PTR Csv.Table ;
  fr     : PTR Frame.Fr ;
  ts, hr : PTR Frame.Ts ;
  hows   : SLICE OF Frame.How ;
  tm     : SLICE OF I64 ;
  ta     : SLICE OF F64 ;
  i      : I64 ;
  stamp  : Time.Instant ;

BEGIN
  opt := Csv.Defaults () ;
  opt.missing := -9999.0 ;
  opt.hasMissing := TRUE ;
  t := Csv.Open (pool, 'data/obs.csv', opt) ;
  Csv.SetStamp (t, 0, Csv.StampYmdHm) ;
  Csv.SetReal (t, 1) ;
  Csv.Parse (pool, t) ;

  fr := Frame.FromCsv (pool, t) ;
  ts := Frame.NewTs (pool, fr, Frame.ColI64 (fr, 'TIMESTAMP'),
                     1800, Frame.ConvStart (), 'half-hourly TA') ;

  hows := NEW (pool, Frame.How, 2) ;
  hows [0] := Frame.HowFirst () ;      (* TIMESTAMP *)
  hows [1] := Frame.HowMean () ;       (* TA        *)
  hr := Frame.Average (pool, ts, 3600, hows, 1) ;

  tm := Frame.TsTime (hr) ;
  ta := Frame.ColF64 (Frame.TsFrame (hr), 'TA') ;
  FOR i := 0 TO LEN (tm) - 1 DO
    stamp.t := F64 (tm [i]) ;
    Io.Write (Time.Iso (pool, stamp, 0)) ;
    Io.Write ('  ') ;
    Io.WriteLine (Fmt.Fixed (ta [i], 3))
  END
EXCEPT
| Csv.ParseError (msg, line, col) :
    Io.ErrLine ('CSV refused:') ; Io.ErrLine (msg) ; Io.Halt (1)
| Csv.RangeError (row, col) :
    Io.ErrLine ('CSV value out of range') ; Io.Halt (1)
| Io.IOError :
    Io.ErrLine ('cannot read data/obs.csv (run from examples/)') ;
    Io.Halt (1)
| Frame.WrongType :
    Io.ErrLine ('column type mismatch') ; Io.Halt (1)
| Frame.Unknown :
    Io.ErrLine ('no such column') ; Io.Halt (1)
| Frame.BadArg :
    Io.ErrLine ('bad argument') ; Io.Halt (1)
| Frame.SizeError :
    Io.ErrLine ('sizes disagree') ; Io.Halt (1)
| Frame.Disorder :
    Io.ErrLine ('time is not ascending') ; Io.Halt (1)
| Frame.Duplicate :
    Io.ErrLine ('duplicate column') ; Io.Halt (1)
| ValueRange :
    Io.ErrLine ('a value did not fit') ; Io.Halt (1)
| IndexError :
    Io.ErrLine ('index out of range') ; Io.Halt (1)
| Overflow :
    Io.ErrLine ('overflow') ; Io.Halt (1)
END C7Series.
```

```output C7Series
2025-12-01T00:00:00Z  3.100
2025-12-01T01:00:00Z  2.700
2025-12-01T02:00:00Z  1.950
2025-12-01T03:00:00Z  1.650
```

The output is the hand-checkable answer: 3.100 is the mean of 3.2
and 3.0; the 01:00 window had one valid reading and one gap, so it
answers 2.700 — the gap EXCLUDED and the count rule (`minCount`)
in the caller's hands, per chapter 6's policy; nothing invents a
value for the missing half hour.

Two design points worth pausing on:

- **`hows` says what "average" means PER COLUMN.**  A mean of the
  timestamps would be nonsense; flags want ALL/ANY (`HowLo` /
  `HowHi` over booleans); counters want `HowSum`; text wants
  `HowFirst` or refusal.  One word per column, visible at the call.
- **Windows are epoch-aligned** — [00:00, 01:00), [01:00, 02:00) —
  the same rule as polars' `group_by_dynamic`, and the library's
  gate holds the two to EXACT agreement on shared samples.  Your
  M9 resample and your colleague's polars resample land on the
  same rows to the last bit, or the repository's build fails.

The rest of the timeseries toolkit follows the same shape:
`MakeContiguous` fills missing PERIODS with each column's declared
missing value (and refuses disorder, and refuses to invent booleans);
the netCDF writer (chapter 8's sibling) emits CF-1.8-clean files
with the time zone stated in the units and the window bounds
recorded, because this project has been bitten by files that omit
both; and a Parquet reader/writer covers the interchange case with
pyarrow as its gate.

[← Previous: simple math and statistics](06-math-stats.md) · [Next: data through zarr →](08-zarr.md)
