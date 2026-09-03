# 4 — Memory: pools, slices and strings

Every language answers "who frees this?" somewhere.  C answers it in
the programmer's head, garbage-collected languages answer it later
and invisibly, and M9 answers it **in the signature**: storage is
carved from a named POOL, a pool is freed as one act, and a
procedure that keeps memory beyond its own frame takes the pool as a
parameter.  Measured across this repository's whole library, the
rule holds with no exceptions worth naming: *the pool parameter
appears exactly when the allocation outlives the call.*  A signature
with a pool tells you the result lives on and who owns it; a
signature without one is a promise that nothing was kept.  Strings
are the one case the compiler handles for you: a string a procedure
*answers* — `RETURN a + b`, or a `VAR` parameter it sets — is placed
in the caller's own frame with no pool in sight (`Fmt.Fixed` below
takes none), and the pool parameter is for what is kept beyond that:
a table, a record, a string a module holds on to.

```m9 C4Mem.m9
MODULE C4Mem ;

(* Chapter 4.  Memory, made visible: pools own storage, slices view
   it, VAR says who may change what, and strings are slices of CHAR.
   Every allocation in this program can be pointed at and answered
   for -- there is no garbage collector deciding later, and no free()
   to forget.

   The comments under the procedure headers are DOCSTRINGS: `m9c
   --doc` renders them -- with their `name -- description` parameter
   lines -- into the module's reference page, and the editor shows
   them on hover.  Documentation that lives anywhere else drifts.   *)

IMPORT Io ;
IMPORT Fmt ;
IMPORT DynStr ;

PROCEDURE Spread (RO xs: SLICE OF F64) : F64 =
  (* the spread (max - min) of a series, nothing kept.

       xs -- any slice: a whole array lent at the call site, or a
             SLICE (a, start, len) view of part of one.  RO means
             this procedure may read it and provably does not write
             it -- the caller lends, nothing more.

     No POOL parameter: the docs/pools.md rule is that the pool
     appears exactly when an allocation OUTLIVES the call, and
     nothing here allocates at all.                                 *)
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

PROCEDURE Describe (VAR pool: POOL ; RO label: STR ;
                    RO xs: SLICE OF F64) : STR
  RAISES ValueRange =
  (* one formatted line about a series, built in the CALLER'S pool.
     RAISES ValueRange because Fmt.Fixed can -- the accounting is
     chapter 2's, and it is why this line is in the signature and
     not in a changelog.

       pool  -- where the answer lives.  The pool parameter IS the
                ownership contract: the result outlives this call,
                so the caller says which arena holds it and thereby
                who frees it (freeing the pool frees every string
                built here, in one act).
       label -- prefixed verbatim.
       xs    -- the series; only read.                              *)
VAR
  d : PTR DynStr.DString IN pool ;
  i : I64 ;
  sum : F64 ;
BEGIN
  sum := 0.0 ;
  FOR i := 0 TO LEN (xs) - 1 DO sum := sum + xs [i] END ;
  d := DynStr.New (pool) ;
  DynStr.Append (pool, d, label) ;
  DynStr.Append (pool, d, ': n=') ;
  DynStr.AppendI64 (pool, d, LEN (xs)) ;
  DynStr.Append (pool, d, ' mean=') ;
  DynStr.Append (pool, d, Fmt.Fixed (sum / F64 (LEN (xs)), 2)) ;
  DynStr.Append (pool, d, ' spread=') ;
  DynStr.Append (pool, d, Fmt.Fixed (Spread (xs), 2)) ;
  RETURN DynStr.View (d)
END Describe ;

PROCEDURE Shift (VAR xs: SLICE OF F64 ; offset: F64) =
  (* adds offset to every element, IN PLACE.

       xs     -- VAR: this procedure writes through the slice, and
                 the mode says so at both ends -- the caller can see
                 the mutation coming at the call site, the checker
                 refuses the write without it.
       offset -- what to add.                                       *)
VAR i : I64 ;
BEGIN
  FOR i := 0 TO LEN (xs) - 1 DO
    xs [i] := xs [i] + offset
  END
END Shift ;

VAR
  pool : POOL ;                  (* dies with the program *)
  a    : ARRAY 8 OF F64 ;
  more : SLICE OF F64 ;
  i    : I64 ;
  greeting : STR ;

BEGIN
  FOR i := 0 TO 7 DO a [i] := F64 (i) * 1.5 END ;

  (* an ARRAY lends itself as a slice at a call site; SLICE takes a
     checked sub-view -- same storage, no copy, bounds proven *)
  Io.WriteLine (Describe (pool, 'all', a)) ;
  Io.WriteLine (Describe (pool, 'mid', SLICE (a, 2, 4))) ;

  (* a slice with no array behind it: NEW carves it from a pool *)
  more := NEW (pool, F64, 5) ;
  FOR i := 0 TO 4 DO more [i] := 100.0 + F64 (i) END ;
  Shift (more, 0.25) ;
  Io.WriteLine (Describe (pool, 'shifted', more)) ;

  (* strings are SLICE OF CHAR; `+` composes into the frame's own
     arena, which the compiler creates on the first `+` and frees at
     the exit -- no pool to name, nothing to free, and a result that
     RETURNs is moved to the caller's frame on the way out *)
  greeting := 'pools: ' + 'carve, use, ' + 'free as one' ;
  Io.WriteLine (greeting) ;
  Io.WriteI64 (LEN (greeting)) ;
  Io.WriteLine (' characters, and every one accounted for')
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
30 characters, and every one accounted for
```

Walk the pieces:

- **`VAR pool : POOL`** declares an arena.  As a program-level
  variable it dies with the program; as a procedure local
  (`scratch : POOL`) it dies with the frame, which is the honest
  spelling of "temporary".  There is no per-object free — freeing
  the pool frees every string, slice and record carved from it, in
  one act that cannot miss one.
- **`SLICE OF F64`** is a view: a pointer and a length, no copy.
  An `ARRAY` lends itself as a slice at a call site, `SLICE (a, 2,
  4)` takes a checked sub-view of it, and `NEW (pool, F64, 5)`
  carves a slice with no array behind it.  Every access through any
  of them is bounds-checked against the slice's own length —
  chapter 1's founding rule, applied to views.
- **`RO` and `VAR` are the lending terms.**  `RO xs` says *read
  only, provably*; `VAR xs` says *this procedure writes through the
  slice*, visible at both ends.  `Shift (more, 0.25)` announces the
  mutation at the call site, and without `VAR` the checker refuses
  the write inside.
- **Strings are `SLICE OF CHAR`** — the predeclared name `STR` is
  exactly that.  Literals take `'` or `"` with **no escapes** (a
  string cannot contain its own delimiter, and nothing in a string
  is ever secretly something else).  `+` composes strings into the
  procedure's own FRAME: an arena the compiler creates on the first
  `+` and frees at the exit, so there is no pool to name and nothing
  to free — and a string that leaves through `RETURN` or a `VAR`
  parameter is moved into the caller's frame on the way out, which
  is why `Fmt.Fixed` takes no pool.  `s := s + x` in a loop is
  linear: the arena extends its latest allocation in place.  That
  holds while nothing else is carved between the appends; where it
  cannot be relied on, or where the string must OUTLIVE the frame,
  `DynStr` grows a buffer in a pool you name (`Describe` above), and
  a string that must outlive everything is declared where it is
  needed.  The report's rule, par 2.3: `+` composes, `DynStr`
  accumulates.
- **The docstrings are load-bearing.**  The comment under each
  procedure header, with its `name -- description` parameter lines,
  is what `m9c --doc` renders into the module's reference page and
  what the editor shows on hover (chapter 3).  These examples carry
  them from here on — documentation that lives next to the
  signature is the only kind the compiler can keep honest.

## What the checker refuses

The classic C bug in this territory is returning a pointer into a
dead stack frame.  M9's version is a pointer into a dead POOL — and
it does not compile:

```m9 X4Escape.m9
MODULE X4Escape ;

(* EXPECT-ERROR: pool-interior pointer escapes its pool *)
(* Chapter 4, a program that must NOT compile.  The pool is a LOCAL:
   everything carved from it dies when the procedure returns, so a
   pointer into it must not survive the frame.  In C this is the
   classic return-of-a-dangling-pointer; here it is refused at
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
24:3 X4Escape.Make: pool-interior pointer escapes its pool: p lives in scratch, which dies with this frame (par 4.3)
```

The type `PTR Point IN scratch` names the pool the pointer lives
in, so "does this outlive its arena?" is a question the checker can
answer — and does, at compile time, with the frame and the pool in
the message.  The fix is the signature saying where the storage
should live instead: give `Make` a `VAR pool : POOL` parameter,
declare `p : PTR Point IN pool`, and the same program compiles and
runs — the caller now owns the point, exactly as in `Describe`
above.  (Try it in the cell: it is a three-line edit.  Note the
allocation spelling while you are there: `NEW (pool, Point)`, pool
first, like every NEW.)

Ownership goes further than this chapter needs — `SHARED` counted
handles and `OWN` moves appear with the zarr store in chapter 8 —
but the rule of thumb carries the whole way: **the signature says
who owns what, and the checker holds everyone to it.**

[← Previous: definition and implementation](03-definition-implementation.md) · [Next: reading and writing data →](05-reading-data.md)
