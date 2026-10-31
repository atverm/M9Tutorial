# 13. Procedures: calls, returns, and the parameter modes

Every chapter so far has called procedures.  This one is about what
a procedure heading PROMISES.  In M9 the heading is the whole
contract between a caller and a body, and the parameter modes are
the words it is written in: who may read, who may write, who keeps
what.  The checker holds the body to the heading, so a reviewer can
read the heading and believe it without opening the body.

```m9 C13Modes.m9
MODULE C13Modes ;

(* Chapter 13 covers procedures: how a call passes its arguments, how
   a function answers on every path, and what each parameter MODE
   promises.  The modes are the whole story.  A mode is a promise the
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
   heading that what it passes will be held on to.  RO stands beside
   it because the caller passes a literal, and a literal is lent only
   to a parameter that promises not to write it (par 2.4). --- *)
VAR lastLabel : STR ;

PROCEDURE Remember (RO KEPT s: STR) =
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

    (* Make three points on the plane.  One query below has a
       neighbour within 2, and one has none. *)
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

**Value** parameters are `f: F64` in `Scale` and `v: F64` in `Sign`.
A scalar is copied; the body has its own.  A POINTER passed by value
is not copied into anything the body may change: it is a shared
borrow, readable and not writable, and not passable onward as `VAR`
(report par 4.1).  The X13Write refusal below shows that rule firing.

**VAR** appears in `Scale (VAR p: Point ; f: F64)`.  The body has
the caller's own variable and may change it, and the heading says
so.  After `Scale (a, 0.5)` the caller's `a` is different, and the
reader knew it would be from the `VAR` alone.

**RO** appears in `Norm (RO p: Point)`.  It is a read-only borrow:
the body could not write `p.x` if it tried.  Use it for anything big
enough that you would not want it copied and whose caller must be
sure it is unchanged: a record, a slice, a matrix.  (The current
generator copies an `RO` record.  That is a known cost, not a
semantic difference; the promise holds either way.)

**KEPT** appears in `Remember (RO KEPT s: STR)`.  This body stores
its argument in a module variable, so the string is held beyond the
call.  That is a retention, and par 4.1 requires it to be declared.
Without `KEPT` on `s` the checker refuses the assignment, naming the
parameter and the word that would fix it; the X13Keep refusal below
shows it.  With it, a caller reads off the heading that what it
passes will be kept.  That changes what it may pass: a slice into a
pool that dies at the next line would be a mistake, and now a
visible one.  `KEPT` composes with a mode, and here the mode is
`RO`.  The program passes a string literal.  Nothing writes through
a STR, so a literal may go to any STR parameter, and `RO` says so at
the heading.  A `SLICE OF CHAR` is different: a plain `s: SLICE OF
CHAR` could be written through (`s[0] := 'X'`).  So read-only
storage (a literal, a `CONST`, what an `RO` parameter views) is lent
only to an `RO` one, and the checker refuses the call rather than
trust that the body does not.

**OWN**, not used here, moves ownership of an owned pointer INTO the
procedure; the caller's name is dead afterwards.  Chapter 4 and
report par 4.2 cover it.

## A function answers on every path

`Sign` has three branches and three `RETURN`s.  Remove the `ELSE`
and the checker refuses the procedure: one path reaches `END`
without answering.  In Pascal or C that path would deliver whatever
happened to be in the result register: a garbage number that looks
as real as any other and travels straight into your results.  M9
will not compile a function that can fall off its end.  `X13Fall`
below is exactly that edit, with the checker's answer.

You can apply the rule by eye, because the checker applies it the
same way.  An `IF` without an `ELSE` can be passed, and so can a
`WHILE` or a `FOR`.  A `LOOP` can be passed only through its own
`EXIT`, and a `CASE` or a handler when one of its arms does not
answer.  A call never counts as an ending, whatever the procedure it
calls does.  So a function whose last act is to fail writes `RAISE`,
not a helper that raises for it.

## "Maybe" is a type, not a null pointer

`Nearest` answers `OPT PTR Point`: a point, or `NONE`.  Inside, a
found pointer is wrapped with `SOME (pts[i])`.  A `PTR` never
becomes an `OPT PTR` by itself, any more than an `I32` becomes an
`I64`.  Outside, the caller cannot touch the result except through
the guard:

    IF Nearest (pts, origin, 2.0) IS SOME p THEN ... ELSE ... END

`p` exists only inside the THEN, and only when there is something
for it to be.  You cannot write the faith-based dereference that
turns an empty result into a crash.  The guard is the only door to
the value, so the case where there is nothing cannot be forgotten.

## Enumerations: a type that is a fixed set of names

A scientist records much that is categorical, not numeric: a
quality flag, an instrument state, a land-cover class.  Written as
bare integers (`0` good, `1` suspect, `2` bad), the meaning lives in
a comment somewhere.  The compiler cannot help when a `3` slips in
or two tables disagree on what `1` meant.  An **enumeration** makes
the set of values a type:

```m9 C13Kinds.m9
MODULE C13Kinds ;

(* Chapter 13.  An ENUMERATION is a type that is a fixed, named set of
   values: a quality flag, an instrument state, a category.  It is a
   case record with no payload, so the total-CASE rule from this
   chapter applies.  Every value is covered, with no ELSE.  A value
   added to the type later breaks every CASE that forgot it, at
   compile time, not in a run.  NAME turns a value into its own
   identifier text.  An ARRAY indexed by the type tallies one slot
   per value with no ordinals written down anywhere. *)

IMPORT Io ;
IMPORT Fmt ;

TYPE Quality = (Good, Suspect, Bad) ;

PROCEDURE Keep (q: Quality) : BOOL =
  (* Keep says which readings survive quality control.  It is total
     over Quality: add a fourth flag and this procedure will not
     compile until it says what to do with it. *)
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
rule from `Sign` above applies to it directly.  `Keep` covers every
`Quality` and needs no `ELSE`.  The day someone adds a fourth flag,
it stops compiling until it says what to do with the new one.  That
is the opposite of an integer `switch` that silently falls through.
`NAME` turns a value into its own identifier text, so a report reads
`Good`, `Suspect`, `Bad` without a hand-written table that can drift
from the type.  `FOR q := Quality.Good TO Quality.Bad` walks the
members in order.  `ARRAY Quality OF I64` gives one tally slot per
value, indexed by the value itself: `tally[flags[i]]`.  No ordinal
is written down, and no index can run past the end, because the
index *is* the type.

## What the checker refuses

Here is a function that can reach its `END` without answering:
`Sign` with its `ELSE` removed.

```m9 X13Fall.m9
MODULE X13Fall ;

(* EXPECT-ERROR: a function must RETURN or RAISE on every path *)
(* Chapter 13 shows a program that must NOT compile.  It is Sign
   from C13Modes with its ELSE removed.  Now one path, v = 0.0,
   reaches END with nothing to answer.  In Pascal or C that path
   delivers whatever was in the result register.  Here the checker
   names the function before it can.  The rule is structural, so a
   reader can apply it by eye.  An IF without ELSE can be passed,
   and so can a WHILE or a FOR.  A call never counts as an ending.  *)

IMPORT Io ;

PROCEDURE Sign (v: F64) : I64 =
BEGIN
  IF v > 0.0 THEN
    RETURN 1
  ELSIF v < 0.0 THEN
    RETURN -1
  END                        (* refused: v = 0.0 answers nothing *)
END Sign ;

BEGIN
  Io.WriteI64 (Sign (0.0)) ;
  Io.WriteLine ('')
END X13Fall.
```

```refusal X13Fall
X13Fall.m9:14:1: X13Fall.Sign: a function must RETURN or RAISE on every path: Sign can reach its END (par 3)
```

This is the shared-borrow rule, on a pointer passed by value:

```m9 X13Write.m9
MODULE X13Write ;

(* EXPECT-ERROR: cannot write through a value parameter *)
(* Chapter 13 shows a program that must NOT compile.  A pointer
   passed by VALUE is a shared borrow: the body may read what it
   points at and may not change it.  In C this would be a silent
   write into the caller's data through a copied pointer.  Here the
   heading's mode says "look, do not touch"; the checker holds the
   body to it.  Write `VAR p: PTR Point` if you mean to change it.  *)

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
X13Write.m9:20:3: X13Write.Zero: cannot write through a value parameter: p is a shared borrow (take VAR, par 4.1)
X13Write.m9:21:3: X13Write.Zero: cannot write through a value parameter: p is a shared borrow (take VAR, par 4.1)
```

And this is the retention rule, on a borrowed string that reaches
module state:

```m9 X13Keep.m9
MODULE X13Keep ;

(* EXPECT-ERROR: undeclared retention: borrowed s reaches module state -- declare KEPT s (par 4.1) *)
(* Chapter 13 shows a program that must NOT compile.  Remember
   stores its argument in a module variable, so the string is KEPT
   beyond the call.  The heading does not say so.  A caller reading
   `RO s: STR` is entitled to believe the call borrows s and lets it
   go.  The body breaks that promise, and the checker names the
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
X13Keep.m9:17:13: X13Keep.Remember: undeclared retention: borrowed s reaches module state -- declare KEPT s (par 4.1)
```

The X13Write and X13Keep diagnostics both name the parameter and
the mode that would make the program legal.  Neither is a style complaint.  The first is a
write into the caller's data through a copied pointer, a classic
silent C bug.  The second is a dangling reference waiting for the
caller to free what the module still points at.

[← Previous: building strings](12-building-strings.md)

[Next: a big CSV file, and a CF NetCDF file from two of its columns →](14-big-csv.md)
