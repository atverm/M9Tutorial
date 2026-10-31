# 4 — Memory: pools, slices and strings

Every language answers "who frees this?" somewhere.  C answers it in
the programmer's head.  Garbage-collected languages answer it later
and invisibly.  M9 answers it **where the storage is asked for**:
the first argument of `NEW` says who frees it.  Most of the time it
says nothing, and that is the default.  `NEW (F64, 5)`, like `+` on
two strings, allocates in the procedure's own FRAME.  The FRAME is
an arena the compiler creates on first use and frees at the exit,
with nothing to declare and nothing to free.  The things a procedure
*answers* survive that exit.  A string, a slice or a record it
builds (`RETURN a + b`, a `VAR` parameter it sets, a buffer it grew)
lands in the caller's frame, not in the one that is about to die.
(`Ramp`, `Describe` and `Fmt.Fixed` below take no pool.)  An object
a `VAR` parameter hands in brings its own storage along, so a
procedure that grows it names none either.

A named POOL is the exception, and it is written down so that it is
noticed.  It is an arena with a name, freed as one act, for the
places where somebody has to say where storage lives.
`scratch : POOL` in a procedure is this frame saying it to a
constructor that asks.  `VAR pool : POOL` in a signature is the
procedure asking its caller.  Measured across this repository's
whole library, the rule is: *the pool parameter appears exactly when
the caller must say where storage lives*.  Examples are a
constructor that promises `PTR T IN pool`, a reader that fills a
table the caller owns, and a string a module holds on to.  A
signature with a pool tells you the result lives on and who owns it.
A signature without one promises that nothing was kept beyond what
it answers.

```m9 C4Mem.m9
MODULE C4Mem ;

IMPORT Text ;

(* Chapter 4 makes memory visible.  A frame owns storage unless a
   named pool does, slices view it, VAR says who may change what, and
   strings are UTF-8 text.  Every allocation in this program can
   be pointed at and answered for.  No garbage collector decides
   later, and there is no free() to forget.

   The comments under the procedure headers are DOCSTRINGS.  `m9c
   --doc` renders them, with their `name -- description` parameter
   lines, into the module's reference page, and the editor shows
   them on hover.  Documentation that lives anywhere else drifts.   *)

IMPORT Io ;
IMPORT Fmt ;
IMPORT DynStr ;

PROCEDURE Spread (RO xs: SLICE OF F64) : F64 =
  (* Spread answers the spread (max - min) of a series, nothing kept.

       xs -- any slice: a whole array lent at the call site, or a
             SLICE (a, start, len) view of part of one.  RO means
             this procedure may read it and provably does not write
             it: the caller lends, nothing more.

     Spread takes no POOL parameter and has no NEW.  The docs/pools.md
     rule is that the pool parameter appears exactly when the caller
     must say where storage lives, and nothing here allocates at all. *)
VAR
  i : I64 ;
  lo, hi : F64 ;
BEGIN
  lo := xs [0] ;
  hi := xs [0] ;
  FOR i := 1 TO LEN (xs) - 1 DO
    IF xs [i] < lo THEN lo := xs [i] END ;
    IF xs [i] > hi THEN hi := xs [i] END
  END ;
  RETURN hi - lo
END Spread ;

PROCEDURE Ramp (n: I64 ; start: F64) : SLICE OF F64
  RAISES ValueRange =
  (* Ramp answers n values rising by one from start, as a new slice.

       n     -- how many.
       start -- the first of them.

     NEW (F64, n) names no pool, which is the default: the storage
     comes from a FRAME.  This procedure ANSWERS the slice, so that
     frame is its caller's.  The slice is still there when this frame
     is gone, and nobody had to be asked where to put it.           *)
VAR
  xs : SLICE OF F64 ;
  i : I64 ;
BEGIN
  xs := NEW (F64, n) ;
  FOR i := 0 TO n - 1 DO xs [i] := start + F64 (i) END ;
  RETURN xs
END Ramp ;

PROCEDURE Describe (RO label: STR ;
                    RO xs: SLICE OF F64) : STR
  RAISES ValueRange =
  (* Describe answers one formatted line about a series, into the
     CALLER'S frame.  The one NAMED pool of this program is here:
     DynStr.New asks which pool its buffer lives in, so this frame
     declares a scratch pool and says so.  The pool dies with the
     frame, and a string that RETURNs is moved out on the way back,
     so nothing here needs a pool in the signature.  It RAISES
     ValueRange because Fmt.Fixed can.  The accounting is chapter
     2's, and it is why this line is in the signature and not in a
     changelog.
       label -- prefixed verbatim.
       xs    -- the series; only read.                              *)
VAR
  scratch : POOL ;
  d : PTR DynStr.DString IN scratch ;
  i : I64 ;
  sum : F64 ;
BEGIN
  sum := 0.0 ;
  FOR i := 0 TO LEN (xs) - 1 DO sum := sum + xs [i] END ;
  d := DynStr.New (scratch) ;
  DynStr.Append (d, label) ;
  DynStr.Append (d, ': n=') ;
  DynStr.AppendI64 (d, LEN (xs)) ;
  DynStr.Append (d, ' mean=') ;
  DynStr.Append (d, Fmt.Fixed (sum / F64 (LEN (xs)), 2)) ;
  DynStr.Append (d, ' spread=') ;
  DynStr.Append (d, Fmt.Fixed (Spread (xs), 2)) ;
  RETURN DynStr.View (d)
END Describe ;

PROCEDURE Shift (VAR xs: SLICE OF F64 ; offset: F64) =
  (* Shift adds offset to every element, IN PLACE.

       xs     -- VAR: this procedure writes through the slice, and
                 the mode says so at both ends.  The caller can see
                 the mutation coming at the call site; the checker
                 refuses the write without it.
       offset -- what to add.                                       *)
VAR i : I64 ;
BEGIN
  FOR i := 0 TO LEN (xs) - 1 DO
    xs [i] := xs [i] + offset
  END
END Shift ;

VAR
  a    : ARRAY 8 OF F64 ;
  more : SLICE OF F64 ;
  i    : I64 ;
  greeting : STR ;

BEGIN
  FOR i := 0 TO 7 DO a [i] := F64 (i) * 1.5 END ;

  (* An ARRAY lends itself as a slice at a call site.  SLICE takes
     a checked sub-view: same storage, no copy, bounds proven. *)
  Io.WriteLine (Describe ('all', a)) ;
  Io.WriteLine (Describe ('mid', SLICE (a, 2, 4))) ;

  (* This slice has no array behind it.  Ramp carves it with NEW and
     names no pool, so it lives in THIS frame, the program's, and is
     freed with it. *)
  more := Ramp (5, 100.0) ;
  Shift (more, 0.25) ;
  Io.WriteLine (Describe ('shifted', more)) ;

  (* Strings are UTF-8 text.  `+` composes into the frame's own
     arena, which the compiler creates on the first `+` and frees at
     the exit.  There is no pool to name and nothing to free.  A
     result that RETURNs is moved to the caller's frame on the way
     out. *)
  greeting := 'pools: ' + 'carve, use, ' + 'free as one' ;
  Io.WriteLine (greeting) ;
  Io.WriteI64 (Text.BLen (greeting)) ;
  Io.WriteLine (' bytes, and every one accounted for')
EXCEPT
| ValueRange :
    Io.ErrLine ('formatting failed') ;
    Io.Halt (1)
END C4Mem.
```

```output C4Mem
all: n=8 mean=5.25 spread=10.50
mid: n=4 mean=5.25 spread=4.50
shifted: n=5 mean=102.25 spread=4.00
pools: carve, use, free as one
30 bytes, and every one accounted for
```

Walk the pieces:

- **`NEW (F64, n)` names no pool**, and that is the default: the
  storage comes from a frame.  `Ramp` *answers* its slice, so the
  slice is built in the caller's frame and is still there when
  `Ramp`'s own is gone.  The program shifts it and reads it three
  lines later.  Nothing was copied on the way, and nobody was asked
  where to put it.
- **`scratch : POOL` is the exception, by name.**  A POOL is an
  arena you declare.  As a procedure local it dies with the frame;
  as a program-level variable it dies with the program.  There is
  no per-object free.  Freeing the pool frees every string, slice
  and record carved from it, in one act that cannot miss one.
  `Describe` declares one because `DynStr.New (scratch)` asks which
  pool its buffer lives in.  `NEW (pool, T, n)`, pool first, gives
  the same answer at an allocation of your own.  When you read a
  pool in a program, somebody had to decide a lifetime there.
- **`SLICE OF F64`** is a view: a pointer and a length, no copy.
  An `ARRAY` lends itself as a slice at a call site, `SLICE (a, 2,
  4)` takes a checked sub-view of it, and `NEW (F64, n)` carves a
  slice with no array behind it.  Every access through any of them
  is bounds-checked against the slice's own length.  That is
  chapter 1's founding rule, applied to views.
- **`RO` and `VAR` are the lending terms.**  `RO xs` says *read
  only, provably*; `VAR xs` says *this procedure writes through the
  slice*, visible at both ends.  `Shift (more, 0.25)` announces the
  mutation at the call site, and without `VAR` the checker refuses
  the write inside.
- **Strings are UTF-8 text**; the predeclared name `STR` names that
  type.  `Text.BLen` counts bytes and `Text.CpCount` characters, and
  `FOR c IN s` walks the characters.  A string has no subscript: a
  `SLICE OF CHAR` holds characters you edit in place, and a literal
  fits one too.  Literals take `'` or `"` with **no escapes**.  (A
  string cannot contain its own delimiter, and nothing in a string
  is ever secretly something else.)  `+` composes strings into the
  procedure's own FRAME: an arena the compiler creates on the first
  `+` and frees at the exit.  So there is no pool to name and
  nothing to free.  A string that leaves through `RETURN` or a `VAR`
  parameter is moved into the caller's frame on the way out; that is
  why `Fmt.Fixed` takes no pool.  A `+` that its own statement
  consumes (handed to a call, compared in a condition, assigned to a
  number) is freed when that statement ends.  So a loop that writes
  a composed line each turn uses the arena for one line, not for all
  of them.  `s := s + x` in a loop is linear: the arena extends its
  latest allocation in place.  That holds while nothing else is
  carved between the appends.  Where it cannot be relied on,
  `DynStr` grows a buffer in a pool you name.  `Describe` above
  names a `scratch` pool of its own and RETURNs the view, which is
  moved into the caller's frame on the way out like any other
  answer.  A string that must outlive everything is declared where
  it is needed.  The report's rule, par 2.3, says: `+` composes,
  `DynStr` accumulates.
- **The docstrings are load-bearing.**  `m9c --doc` renders the
  comment under each procedure header, with its
  `name -- description` parameter lines, into the module's reference
  page, and the editor shows it on hover (chapter 3).  These examples
  carry them from here on.  Documentation that lives next to the
  signature is the only kind the compiler can keep honest.

## What the checker refuses

The classic C bug in this territory is returning a pointer into a
dead stack frame.  M9's version is a pointer into a dead POOL, and
it does not compile:

```m9 X4Escape.m9
MODULE X4Escape ;

(* EXPECT-ERROR: pool-interior pointer escapes its pool *)
(* Chapter 4 shows a program that must NOT compile.  The pool is a
   LOCAL: everything carved from it dies when the procedure returns.
   So a pointer into it must not survive the frame.  In C this is
   the classic return-of-a-dangling-pointer.  Here it is refused at
   compile time, by name.                                           *)

IMPORT Io ;

TYPE
  Point = RECORD
    x, y : F64 ;
  END ;

PROCEDURE Make () : PTR Point =
VAR
  scratch : POOL ;
  p : PTR Point IN scratch ;
BEGIN
  p := NEW (scratch, Point) ;
  p.x := 1.0 ;
  RETURN p                 (* refused: scratch dies with this frame *)
END Make ;

VAR q : PTR Point ;

BEGIN
  q := Make () ;
  Io.WriteLine ('never compiled')
END X4Escape.
```

```refusal X4Escape
X4Escape.m9:24:3: X4Escape.Make: pool-interior pointer escapes its pool: p lives in scratch, which dies with this frame (par 4.3)
```

The type `PTR Point IN scratch` names the pool the pointer lives
in.  So the checker can answer "does this outlive its arena?", and
it does, at compile time, with the frame and the pool in the
message.  There are two fixes, and the choice between them is this
chapter's question again.

If `Make` simply answers a point, name no pool at all: delete the
`scratch` line, declare `p : PTR Point` and allocate with `p := NEW
(Point)`.  Do all three, because a `p` still declared `IN scratch`
cannot hold a frame allocation; the checker says that as well.  The
point is then built in the caller's frame, the way `Ramp` answers
its slice.

If the caller should decide where the point lives, let the
signature ask.  Give `Make` a `VAR pool : POOL` parameter and
declare `p : PTR Point IN pool`.  Allocate with `NEW (pool, Point)`,
pool first like every NEW that names one, and answer `PTR Point IN
pool`.  The program then declares a pool of its own, passes it
(`q := Make (pool)`), and owns the point.  `Csv.Open` in the next
chapter works the same way: it hands back a table in the pool it was
given.  (Try both in the cell.)

Ownership goes further than this chapter needs.  NEW's first
argument can say a third thing, `NEW (OWN, T)`, for storage one
binding owns.  The `SHARED` counted handles and `OWN` moves appear
with the zarr store in chapter 8.  But the rule of thumb carries the
whole way: **the program says who owns what, where it asks for the
storage and in the signature, and the checker holds everyone to
it.**

[← Previous: definition and implementation](03-definition-implementation.md) · [Next: reading and writing data →](05-reading-data.md)
