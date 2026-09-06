# 12. Building strings

A string in M9 is a value, `+` joins two of them, and the result goes
into the frame's arena or into `HEAP` (report par 2.3).  That is the
whole mechanism, and it is enough to write a wrong program that
compiles, checks and runs.  This chapter is about that program: three
ways to pad a string on the left, of which the first is wrong, the
second is right and slow, and the third is right.

```m9 C12Pad.m9
MODULE C12Pad ;

(* Chapter 12: building strings -- three ways to pad a string on the
   left, of which the first is wrong, the second is right and slow,
   and the third is right.  The compiler accepts all three.  *)

IMPORT Io ;
IMPORT Fmt ;
IMPORT Time ;

(* 1. THE HORROR.  Named PadLeft in the first draft, and it compiles,
   checks and runs -- and pads on the RIGHT, because `d + ' '`
   appends.  No checker knows which side a name promises. *)
PROCEDURE PadWrong (RO s: STR ; width: I64) : STR =
VAR
  d : STR ;
  i : I64 ;
BEGIN
  d := '' + s ;                (* a copy of our own, in this frame *)
  FOR i := LEN (s) + 1 TO width DO d := d + ' ' END ;
  RETURN d
END PadWrong ;

(* 2. CORRECT, AND QUADRATIC.  `' ' + d` prepends, so the space lands
   on the left.  But `+` extends its LEFT operand in place only when
   that operand is the newest thing in the frame's arena; with a
   literal on the left it must copy all of d every time round, so n
   spaces cost n^2 character copies. *)
PROCEDURE PadSlow (RO s: STR ; width: I64) : STR =
VAR
  d : STR ;
  i : I64 ;
BEGIN
  d := '' + s ;
  FOR i := LEN (s) + 1 TO width DO d := ' ' + d END ;
  RETURN d
END PadSlow ;

(* 3. CORRECT, AND LINEAR.  Build the padding by APPENDING -- the
   loop's left operand is always the newest thing in the arena, so
   each `+` extends it in place -- and prepend it to the body ONCE.
   The same operator, used the way it is cheap. *)
PROCEDURE PadLeft (RO s: STR ; width: I64) : STR =
VAR
  pad : STR ;
  i : I64 ;
BEGIN
  pad := '' ;
  FOR i := LEN (s) + 1 TO width DO pad := pad + ' ' END ;
  RETURN pad + s
END PadLeft ;

CONST Big = 20000 ;            (* for the timing, when asked *)

VAR
  t0, t1 : Time.Instant ;
  slow, fast : F64 ;
  s : STR ;
  c : CHAR ;

BEGIN
  BEGIN
    Io.WriteLine ('[' + PadWrong ('42', 8) + ']  PadWrong: named left, pads right') ;
    Io.WriteLine ('[' + PadSlow ('42', 8) + ']  PadSlow: correct, and quadratic') ;
    Io.WriteLine ('[' + PadLeft ('42', 8) + ']  PadLeft: correct, and linear') ;
    Io.WriteLine ('[' + Fmt.I64Pad (42, 8, FALSE)
                  + ']  Fmt.I64Pad: for a number, the library already has it') ;

    (* a CHAR joins on either side: one code point, and no format to
       choose -- which is why a NUMBER does not (see X12Int) *)
    c := '*' ;
    Io.WriteLine ('[' + ('42' + c) + ']  a CHAR appended, [' + (c + '42')
                  + '] prepended') ;

    (* run with any argument to see the cost of prepending *)
    IF Io.ArgCount () > 1 THEN
      t0 := Time.Now () ;
      s := PadSlow ('x', Big) ;
      t1 := Time.Now () ;
      slow := Time.Elapsed (t0, t1) ;
      t0 := Time.Now () ;
      s := PadLeft ('x', Big) ;
      t1 := Time.Now () ;
      fast := Time.Elapsed (t0, t1) ;
      Io.ErrLine ('padding to ' + Fmt.I64Str (Big) + ': PadSlow '
                  + Fmt.Fixed (slow, 3) + ' s, PadLeft '
                  + Fmt.Fixed (fast, 4) + ' s') ;
      Io.ErrLine ('LEN of the result: ' + Fmt.I64Str (LEN (s)))
    END
  EXCEPT
  | ValueRange : Io.ErrLine ('a value out of range')
  END
END C12Pad.
```

```output C12Pad
[42      ]  PadWrong: named left, pads right
[      42]  PadSlow: correct, and quadratic
[      42]  PadLeft: correct, and linear
[      42]  Fmt.I64Pad: for a number, the library already has it
[42*]  a CHAR appended, [*42] prepended
```

## The horror: a checker cannot read a name

`PadWrong` was called `PadLeft` in the first draft.  Its loop says
`d := d + ' '`, which appends a space, so it pads on the right — and
nothing in the toolchain can object.  The types are right, the
borrows are right, every path returns.  What is wrong is that the
name promises one side and the code delivers the other, and no
checker knows which side a name promises.

That is the reviewer's job, and it is why this language is shaped
the way it is: the loop is four tokens long and reads as exactly
what it does.  A reviewer who reads `d + ' '` knows the space goes
on the right.  A reviewer who reads a call to a library `pad` with a
flag argument has to look the flag up.

## Correct, and quadratic

`PadSlow` fixes the side: `d := ' ' + d` prepends, so the spaces
land where the name says.  It is also thousands of times slower than
it needs to be, and the reason is worth understanding once.

`+` allocates its result in the frame's arena, and it has one
optimisation: when its LEFT operand is the newest allocation there,
it extends that allocation in place instead of copying.  That is
what makes `d := d + x` in a loop linear.  With a literal on the
left, `' ' + d` can never extend anything — the literal is not in
the arena — so it copies all of `d` every time round, and `n` spaces
cost about `n²/2` character copies.  Run the program with any
argument and it measures both on 20,000 characters:

    padding to 20000: PadSlow 0.680 s, PadLeft 0.0001 s

(That line goes to standard error, and only when asked, because a
timing changes from run to run and the gated output above must
not.)

## Correct, and linear

`PadLeft` uses the same operator the cheap way.  The loop builds the
padding by APPENDING — `pad := pad + ' '`, where `pad` is always the
newest thing in the arena, so each `+` extends it in place — and
then prepends the whole of it to the body once: `RETURN pad + s`.
One copy of `s`, `n` extensions of `pad`, and the spaces are on the
left because the final `+` puts them there.  Three lines, no
library, and the reader can see the side from the last line alone.

The lesson generalises: **append in loops, and prepend once.**  When
a loop builds a string in many pieces of its own, the library's
`DynStr.Append` is the accumulator — a buffer that grows by
doubling, independent of the frame's arena — and `+` stays what it
is best at, composing a line of output out of a few pieces:
`'rows ' + Fmt.I64Str (n)`.

## The library already had it

The fourth line of the output comes from `Fmt.I64Pad (42, 8, FALSE)`:
for a NUMBER, left-padding to a width is Pascal's `:8`, and Fmt has
it, with a flag for zero fill.  Look before writing — the corpus
rule is that the third private copy of something becomes the shared
procedure, and the first copy should be a lookup
(`docs/modules/Fmt.md`, or `m9c --json corpus/Fmt.m9`).

## What the checker refuses

`+` joins strings, and a `CHAR` joins as one code point on either
side — the last line of the output.  A NUMBER does not: `'rows ' + n`
would have to choose a format — digits, sign, padding — and M9 does
not choose silently.

```m9 X12Int.m9
MODULE X12Int ;

(* EXPECT-ERROR: cannot concatenate a string with I64 *)
(* Chapter 12, a program that must NOT compile.  `+` joins strings,
   and a CHAR is one code point and joins too; a NUMBER is neither.
   'rows ' + n would have to choose a format -- how many digits, a
   sign, padding -- and M9 does not choose silently.  Fmt.I64Str (n)
   says how, and the reviewer can see it.                            *)

IMPORT Io ;

VAR
  n : I64 ;
  s : STR ;

BEGIN
  n := 336 ;
  s := 'rows ' + n ;          (* refused: format the number first *)
  Io.WriteLine (s)
END X12Int.
```

```refusal X12Int
18:16 X12Int body: cannot concatenate a string with I64
```

`Fmt.I64Str (n)` says how, `Fmt.Fixed (x, 3)` says how for a real,
and a reviewer reading the line can see the format that was chosen.

[← Previous: threads, waiting in parallel](11-threads.md)

[Next: procedures, calls, returns and the parameter modes →](13-procedures-modes.md)
