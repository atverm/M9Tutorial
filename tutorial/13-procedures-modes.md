# 13. Procedures: calls, returns, and the parameter modes

Every chapter so far has called procedures; this one is about what a
procedure heading PROMISES.  In M9 the heading is the whole contract
between a caller and a body, and the parameter modes are the words
it is written in: who may read, who may write, who keeps what.  The
checker holds the body to the heading, so a reviewer can read the
heading and believe it without opening the body.

```m9 C13Modes.m9
MODULE C13Modes ;

(* Chapter 13: procedures -- how a call passes its arguments, how a
   function answers on every path, and what each parameter MODE
   promises.  The modes are the whole story: a mode is a promise the
   compiler holds the body to, and the reviewer can read it off the
   heading without opening the body.

     value   a copy for a scalar, a shared read-only borrow for a
             pointer: the body may look, not change
     VAR     the caller's own variable: the body may change it
     RO      a read-only borrow, and the reader knows nothing
             changes even though the thing is big
     KEPT    the body KEEPS the argument beyond the call -- it must
             say so, or the retention is refused                    *)

IMPORT Io ;
IMPORT Fmt ;
IMPORT Math ;

TYPE
  Point = RECORD
    x, y : F64 ;
  END ;
  PointRef = PTR Point ;     (* NEW wants a type NAME for its elements *)

(* --- VAR: the body changes the caller's variable, and says so --- *)
PROCEDURE Scale (VAR p: Point ; f: F64) =
BEGIN
  p.x := p.x * f ;
  p.y := p.y * f
END Scale ;

(* --- RO: a read-only borrow.  The heading promises p is unchanged;
   the body could not write p.x if it tried. --- *)
PROCEDURE Norm (RO p: Point) : F64 RAISES ValueRange =
BEGIN
  RETURN Math.Sqrt (p.x * p.x + p.y * p.y)
END Norm ;

(* --- a function answers on EVERY path, or it is refused.  The
   ELSE is not decoration: without it there is a path that reaches
   END with no RETURN, and the checker names it. --- *)
PROCEDURE Sign (v: F64) : I64 =
BEGIN
  IF v > 0.0 THEN
    RETURN 1
  ELSIF v < 0.0 THEN
    RETURN -1
  ELSE
    RETURN 0
  END
END Sign ;

(* --- "maybe an answer" is OPT, never a null pointer: the caller
   cannot use the result without first asking IS SOME --- *)
PROCEDURE Nearest (RO pts: SLICE OF PointRef ; RO q: Point ; within: F64)
  : OPT PTR Point RAISES ValueRange =
VAR
  best : OPT PTR Point ;
  bestD, d : F64 ;
  i : I64 ;
  delta : Point ;
BEGIN
  best := NONE ;
  bestD := within ;
  FOR i := 0 TO LEN (pts) - 1 DO
    delta.x := pts[i].x - q.x ;
    delta.y := pts[i].y - q.y ;
    d := Norm (delta) ;
    IF d < bestD THEN
      bestD := d ;
      best := SOME (pts[i])           (* a PTR is wrapped, never coerced *)
    END
  END ;
  RETURN best
END Nearest ;

(* --- KEPT: this procedure stores its argument in module state, so
   the string outlives the call.  Without KEPT on s the checker
   refuses the assignment: a borrowed argument may not reach module
   state undeclared (par 4.1).  With it, the caller can read off the
   heading that what it passes will be held on to. --- *)
VAR lastLabel : STR ;

PROCEDURE Remember (KEPT s: STR) =
BEGIN
  lastLabel := s
END Remember ;

PROCEDURE Show (RO name: STR ; RO p: Point) RAISES ValueRange =
BEGIN
  Io.WriteLine (name + ' = (' + Fmt.Fixed (p.x, 2) + ', '
                + Fmt.Fixed (p.y, 2) + ')  norm ' + Fmt.Fixed (Norm (p), 3))
END Show ;

VAR
  pool : POOL ;
  a, origin : Point ;
  pts : SLICE OF PointRef ;
  i : I64 ;

BEGIN
  BEGIN
    a.x := 3.0 ; a.y := 4.0 ;
    Show ('a', a) ;

    Scale (a, 0.5) ;                    (* VAR: a itself changes *)
    Show ('a scaled by 0.5', a) ;

    Io.WriteLine ('sign of -2.5: ' + Fmt.I64Str (Sign (-2.5))
                  + ', of 0.0: ' + Fmt.I64Str (Sign (0.0))
                  + ', of 7.0: ' + Fmt.I64Str (Sign (7.0))) ;

    (* three points on the plane, and a query that has a neighbour
       within 2, and one that has none *)
    pts := NEW (pool, PointRef, 3) ;
    FOR i := 0 TO 2 DO
      pts[i] := NEW (pool, Point) ;
      pts[i].x := F64 (i) * 5.0 ;
      pts[i].y := F64 (i) * 5.0
    END ;
    origin.x := 6.0 ; origin.y := 4.0 ;
    IF Nearest (pts, origin, 2.0) IS SOME p THEN
      Io.WriteLine ('nearest to (6, 4) within 2: (' + Fmt.Fixed (p.x, 2)
                    + ', ' + Fmt.Fixed (p.y, 2) + ')')
    ELSE
      Io.WriteLine ('nothing within 2 of (6, 4)')
    END ;
    origin.x := 30.0 ; origin.y := 30.0 ;
    IF Nearest (pts, origin, 2.0) IS SOME p THEN
      Io.WriteLine ('nearest to (30, 30) within 2: (' + Fmt.Fixed (p.x, 2)
                    + ', ' + Fmt.Fixed (p.y, 2) + ')')
    ELSE
      Io.WriteLine ('nothing within 2 of (30, 30)')
    END ;

    Remember ('the label that outlives its call') ;
    Io.WriteLine ('remembered: ' + lastLabel)
  EXCEPT
  | ValueRange : Io.ErrLine ('a value out of range')
  END
END C13Modes.
```

```output C13Modes
a = (3.00, 4.00)  norm 5.000
a scaled by 0.5 = (1.50, 2.00)  norm 2.500
sign of -2.5: -1, of 0.0: 0, of 7.0: 1
nearest to (6, 4) within 2: (5.00, 5.00)
nothing within 2 of (30, 30)
remembered: the label that outlives its call
```

## The modes

**Value** — `f: F64` in `Scale`, `v: F64` in `Sign`.  A scalar is
copied; the body has its own.  A POINTER passed by value is not
copied into anything the body may change: it is a shared borrow,
readable and not writable, and not passable onward as `VAR` (report
par 4.1).  The first refusal below shows that rule firing.

**VAR** — `Scale (VAR p: Point ; f: F64)`.  The body has the
caller's own variable and may change it, and the heading says so.
After `Scale (a, 0.5)` the caller's `a` is different, and the
reader knew it would be from the `VAR` alone.

**RO** — `Norm (RO p: Point)`.  A read-only borrow: the body could
not write `p.x` if it tried.  Use it for anything big enough that
you would not want it copied and whose caller must be sure it is
unchanged — a record, a slice, a matrix.  (`RO` on a record is
copied by the current generator — a known cost, not a semantic
difference; the promise holds either way.)

**KEPT** — `Remember (KEPT s: STR)`.  This body stores its argument
in a module variable, so the string is held beyond the call.  That
is a retention, and par 4.1 requires it to be declared: without
`KEPT` on `s` the checker refuses the assignment, naming the
parameter and the word that would fix it — the second refusal
below.  With it, a caller reads off the heading that what it passes
will be kept, which changes what it may pass (a slice into a pool
that dies at the next line would be a mistake, and now a visible
one).

**OWN**, not used here, moves ownership of an owned pointer INTO the
procedure; the caller's name is dead afterwards.  Chapter 4 and
report par 4.2 cover it.

## A function answers on every path

`Sign` has three branches and three `RETURN`s.  Remove the `ELSE`
and the checker refuses the procedure: there is a path that reaches
`END` without answering, and in Pascal or C that path would deliver
whatever happened to be in the result register — a garbage number
that looks as real as any other and travels straight into your
results.  M9 will not compile a function that can fall off its end.

## "Maybe" is a type, not a null pointer

`Nearest` answers `OPT PTR Point`: a point, or `NONE`.  Inside, a
found pointer is wrapped with `SOME (pts[i])` — a `PTR` never
becomes an `OPT PTR` by itself, any more than an `I32` becomes an
`I64`.  Outside, the caller cannot touch the result except through
the guard:

    IF Nearest (pts, origin, 2.0) IS SOME p THEN ... ELSE ... END

`p` exists only inside the THEN, and only when there is something
for it to be.  There is no way to write the faith-based dereference
that turns an empty result into a crash — the guard is the only door
to the value, so the case where there is nothing cannot be forgotten.

## Enumerations: a type that is a fixed set of names

Much of what a scientist records is categorical, not numeric: a
quality flag, an instrument state, a land-cover class.  Written as
bare integers -- `0` good, `1` suspect, `2` bad -- the meaning lives
in a comment somewhere and the compiler cannot help when a `3` slips
in or two tables disagree on what `1` meant.  An **enumeration** makes
the set of values a type:

```m9 C13Kinds.m9
MODULE C13Kinds ;

(* Chapter 13.  An ENUMERATION is a type that is a fixed, named set of
   values -- a quality flag, an instrument state, a category.  It is a
   case record with no payload, so the total-CASE rule from this
   chapter applies: every value is covered, no ELSE, and a value added
   to the type later breaks every CASE that forgot it -- at compile
   time, not in a run.  NAME turns a value into its own identifier
   text, and an ARRAY indexed by the type tallies one slot per value
   with no ordinals written down anywhere. *)

IMPORT Io ;
IMPORT Fmt ;

TYPE Quality = (Good, Suspect, Bad) ;

PROCEDURE Keep (q: Quality) : BOOL =
  (* which readings survive quality control.  Total over Quality: add
     a fourth flag and this procedure will not compile until it says
     what to do with it. *)
BEGIN
  CASE q OF
  | Good : RETURN TRUE
  | Suspect : RETURN TRUE
  | Bad : RETURN FALSE
  END
END Keep ;

VAR
  pool  : POOL ;
  flags : SLICE OF Quality ;        (* eight readings, each flagged *)
  tally : ARRAY Quality OF I64 ;    (* one count per quality, indexed
                                       BY the type: no ordinals *)
  q : Quality ;
  i, kept : I64 ;

BEGIN
  flags := NEW (pool, Quality, 8) ;
  flags[0] := Quality.Good ;    flags[1] := Quality.Good ;
  flags[2] := Quality.Suspect ; flags[3] := Quality.Bad ;
  flags[4] := Quality.Good ;    flags[5] := Quality.Suspect ;
  flags[6] := Quality.Good ;    flags[7] := Quality.Bad ;

  FOR q := Quality.Good TO Quality.Bad DO tally[q] := 0 END ;
  kept := 0 ;
  FOR i := 0 TO 7 DO
    tally[flags[i]] := tally[flags[i]] + 1 ;
    IF Keep (flags[i]) THEN kept := kept + 1 END
  END ;

  FOR q := Quality.Good TO Quality.Bad DO
    Io.WriteLine (NAME (q) + ': ' + Fmt.I64Str (tally[q]))
  END ;
  Io.WriteLine ('kept ' + Fmt.I64Str (kept) + ' of 8')
END C13Kinds.
```

```output C13Kinds
Good: 4
Suspect: 2
Bad: 2
kept 6 of 8
```

An enumeration is a case record with no payload, so the total-CASE
rule from `Sign` above applies to it directly: `Keep` covers every
`Quality`, needs no `ELSE`, and the day someone adds a fourth flag it
stops compiling until it says what to do with the new one -- the
opposite of an integer `switch` that silently falls through.  `NAME`
turns a value into its own identifier text, so a report reads `Good`,
`Suspect`, `Bad` without a hand-written table that can drift from the
type.  `FOR q := Quality.Good TO Quality.Bad` walks the members in
order, and `ARRAY Quality OF I64` gives one tally slot per value,
indexed by the value itself -- `tally[flags[i]]`, with no ordinal
written down and no way to index past the end, because the index *is*
the type.

## What the checker refuses

The shared-borrow rule, on a pointer passed by value:

```m9 X13Write.m9
MODULE X13Write ;

(* EXPECT-ERROR: cannot write through a value parameter *)
(* Chapter 13, a program that must NOT compile.  A pointer passed by
   VALUE is a shared borrow: the body may read what it points at and
   may not change it.  In C this would be a silent write into the
   caller's data through a copied pointer; here the mode on the
   heading says "look, do not touch", and the checker holds the body
   to it.  Write `VAR p: PTR Point` if you mean to change it.       *)

IMPORT Io ;

TYPE
  Point = RECORD
    x, y : F64 ;
  END ;

PROCEDURE Zero (p: PTR Point) =
BEGIN
  p.x := 0.0 ;               (* refused: p is a value parameter *)
  p.y := 0.0
END Zero ;

VAR
  pool : POOL ;
  q : PTR Point IN pool ;

BEGIN
  q := NEW (pool, Point) ;
  Zero (q) ;
  Io.WriteLine ('never compiled')
END X13Write.
```

```refusal X13Write
20:3 X13Write.Zero: cannot write through a value parameter: p is a shared borrow (take VAR, par 4.1)
21:3 X13Write.Zero: cannot write through a value parameter: p is a shared borrow (take VAR, par 4.1)
```

And the retention rule, on a borrowed string that reaches module
state:

```m9 X13Keep.m9
MODULE X13Keep ;

(* EXPECT-ERROR: undeclared retention: borrowed s reaches module state -- declare KEPT s (par 4.1) *)
(* Chapter 13, a program that must NOT compile.  Remember stores its
   argument in a module variable, so the string is KEPT beyond the
   call -- and the heading does not say so.  A caller reading
   `RO s: STR` is entitled to believe the call borrows s and lets it
   go; the body breaks that promise, and the checker names the
   parameter and the word that would fix it.                         *)

IMPORT Io ;

VAR lastLabel : STR ;

PROCEDURE Remember (RO s: STR) =
BEGIN
  lastLabel := s             (* refused: s is borrowed, not kept *)
END Remember ;

BEGIN
  Remember ('a label') ;
  Io.WriteLine (lastLabel)
END X13Keep.
```

```refusal X13Keep
17:13 X13Keep.Remember: undeclared retention: borrowed s reaches module state -- declare KEPT s (par 4.1)
```

Both diagnostics name the parameter and the mode that would make
the program legal.  Neither is a style complaint: the first is a
write into the caller's data through a copied pointer, which is a
classic silent C bug, and the second is a dangling reference waiting
for the caller to free what the module still points at.

[← Previous: building strings](12-building-strings.md)

[Next: a big CSV file, and a CF NetCDF file from two of its columns →](14-big-csv.md)
