# 12. Building strings

A string in M9 is a value, `+` joins two of them, and the result goes
into the frame's arena or into `HEAP` (report par 2.3).  That is the
whole mechanism, and it is enough to write a wrong program that
compiles, checks and runs.  This chapter is about that program.  It
shows three ways to pad a string on the left: the first is wrong,
the second is right and slow, and the third is right.

```m9 C12Pad.m9
MODULE C12Pad ;

IMPORT Text ;

(* Chapter 12 builds strings.  It shows three ways to pad a string
   on the left: the first is wrong, the second is right and slow, and
   the third is right.  The compiler accepts all three.  *)

IMPORT Io ;
IMPORT Fmt ;
IMPORT Time ;

(* 1. THE HORROR.  The first draft named it PadLeft.  It compiles,
   checks and runs, and it pads on the RIGHT, because `d + ' '`
   appends.  No checker knows which side a name promises. *)
PROCEDURE PadWrong (RO s: STR ; width: I64) : STR =
VAR
  d : STR ;
  i : I64 ;
BEGIN
  d := '' + s ;                (* a copy of our own, in this frame *)
  FOR i := Text.BLen (s) + 1 TO width DO d := d + ' ' END ;
  RETURN d
END PadWrong ;

(* 2. CORRECT, AND QUADRATIC.  `' ' + d` prepends, so the space lands
   on the left.  But `+` extends its LEFT operand in place only when
   that operand is the newest thing in the frame's arena.  With a
   literal on the left it must copy all of d every time round, so n
   spaces cost n^2 character copies. *)
PROCEDURE PadSlow (RO s: STR ; width: I64) : STR =
VAR
  d : STR ;
  i : I64 ;
BEGIN
  d := '' + s ;
  FOR i := Text.BLen (s) + 1 TO width DO d := ' ' + d END ;
  RETURN d
END PadSlow ;

(* 3. CORRECT, AND LINEAR.  Build the padding by APPENDING, then
   prepend it to the body ONCE.  The loop's left operand is always
   the newest thing in the arena, so each `+` extends it in place.
   This is the same operator, used the way it is cheap. *)
PROCEDURE PadLeft (RO s: STR ; width: I64) : STR =
VAR
  pad : STR ;
  i : I64 ;
BEGIN
  pad := '' ;
  FOR i := Text.BLen (s) + 1 TO width DO pad := pad + ' ' END ;
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

    (* A CHAR joins on either side: one code point, and no format to
       choose.  That is why a NUMBER does not (see X12Int). *)
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
      Io.ErrLine ($'padding to ${Big}: PadSlow ${slow:.3}'
                  + $' s, PadLeft ${fast:.4} s') ;
      Io.ErrLine ($'LEN of the result: ${Text.BLen (s)}')
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
`d := d + ' '`, which appends a space, so it pads on the right.
Nothing in the toolchain can object.  The types are right, the
borrows are right, every path returns.  The fault is that the name
promises one side and the code delivers the other, and no checker
knows which side a name promises.

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
it extends that allocation in place instead of copying.  That makes
`d := d + x` in a loop linear.  With a literal on the left,
`' ' + d` can never extend anything, because the literal is not in
the arena.  So it copies all of `d` every time round, and `n` spaces
cost about `n²/2` character copies.  Run the program with any
argument and it measures both on 20,000 characters:

    padding to 20000: PadSlow 0.680 s, PadLeft 0.0001 s

(That line goes to standard error, and only when asked, because a
timing changes from run to run and the gated output above must
not.)

## Correct, and linear

`PadLeft` uses the same operator the cheap way.  The loop builds the
padding by APPENDING: `pad := pad + ' '`.  `pad` is always the
newest thing in the arena, so each `+` extends it in place.  Then
the code prepends the whole of it to the body once:
`RETURN pad + s`.  That costs one copy of `s` and `n` extensions of
`pad`, and the spaces are on the left because the final `+` puts
them there.  It takes three lines and no library, and the reader can
see the side from the last line alone.

The lesson generalises: **append in loops, and prepend once.**  When
a loop builds a string in many pieces of its own, the library's
`DynStr.Append` is the accumulator: a buffer that grows by doubling,
independent of the frame's arena.  And `+` stays what it is best at,
joining a few strings.  A line of output with values in it has a
form of its own, two sections down.

## The library already had it

The fourth line of the output comes from `Fmt.I64Pad (42, 8, FALSE)`.
For a NUMBER, Pascal's `:8` pads left to a width, and Fmt has it,
with a flag for zero fill.  Look before writing.  The corpus rule is
that the third private copy of something becomes the shared
procedure, and the first copy should be a lookup
(`docs/modules/Fmt.md`, or `m9c --json corpus/Fmt.m9`).  The next
section writes the same field inside a string, as `${n:8}`.

## A line with values in it

Most strings a program builds are one line of output: some words,
and values between them.  Written with `+`, every number needs its
`Fmt` call and every piece its quotes, and the line a reader wants
to see is cut into six.  An **interpolation literal** is that line
written as it will print, with a hole where a value goes:

```m9 C12Line.m9
MODULE C12Line ;

(* Chapter 12 writes a line that holds values.  An interpolation
   literal is a string with holes: $'...${e}...'.  Each hole holds
   one expression, and a number may end in a format.  The checker
   holds the format against the type of the value. *)

IMPORT Io ;
IMPORT Fmt ;

CONST
  Station = 'SE-Htm' ;

VAR
  rows, gaps : I64 ;
  mean, tiny : F64 ;
  flag : CHAR ;
  done : BOOL ;

BEGIN
  BEGIN
    rows := 336 ;
    gaps := 7 ;
    mean := -2.4567 ;
    tiny := 0.00001234 ;
    flag := 'U' ;
    done := gaps = 0 ;

    (* The same line twice: by hand, and with holes. *)
    Io.WriteLine ('station ' + Station + ': ' + Fmt.I64Str (rows) + ' rows, mean '
                  + Fmt.Fixed (mean, 2)) ;
    Io.WriteLine ($'station ${Station}: ${rows} rows, mean ${mean:.2}') ;

    (* A hole takes a string, a CHAR, an integer, a BOOL or a real as
       it is.  A real without a format is its shortest exact form. *)
    Io.WriteLine ($'flag ${flag}, complete ${done}, mean ${mean}') ;

    (* A hole holds any one expression. *)
    Io.WriteLine ($'${rows - gaps} of ${rows} present, ${100 * gaps DIV rows} % missing') ;

    (* An integer takes a width, and a zero before it to fill. *)
    Io.WriteLine ($'[${rows:8}] [${rows:08}] [${gaps:2}]') ;

    (* A real takes decimals, a width before them, and e for the
       exponent form.  A lone e gives every digit. *)
    Io.WriteLine ($'[${mean:.2}] [${mean:9.2}] [${tiny:.2e}] [${tiny:e}]') ;

    (* Braces and a lone $ are text, so JSON reads as it is. *)
    Io.WriteLine ($'{"station":"${Station}","rows":${rows},"cost":"$5"}')
  EXCEPT
  | ValueRange : Io.ErrLine ('a value out of range')
  END
END C12Line.
```

```output C12Line
station SE-Htm: 336 rows, mean -2.46
station SE-Htm: 336 rows, mean -2.46
flag U, complete FALSE, mean -2.4567
329 of 336 present, 2 % missing
[     336] [00000336] [ 7]
[-2.46] [    -2.46] [1.23e-5] [1.2340000000000000e-5]
{"station":"SE-Htm","rows":336,"cost":"$5"}
```

The first two lines of the output are the same line, built both
ways.  `$'...'` (or `$"..."`) opens the literal, and `${e}` is a
hole.  The compiler turns the literal into the chain of `+` you
would have written, so everything this chapter said about `+` holds
for it, and nothing about it is decided while the program runs.

- **A hole takes one expression**: a string, a `CHAR`, an integer,
  a `BOOL` or a real.  A real without a format prints its shortest
  exact form, the text that reads back to the same number.
- **A number may end in a format.**  `${n:8}` is a width, `${n:08}`
  fills it with zeros, `${x:.2}` is two decimals, `${x:9.2}` both,
  `${x:.2e}` the exponent form, and `${x:e}` every digit a 64-bit
  real has.  These are the texts `Fmt.I64Pad`, `Fmt.Fixed`,
  `Fmt.FixedPad` and `Fmt.Sci` answer, digit for digit, and a hole
  needs no import.
- **Braces and a lone `$` are text.**  Only `${` opens a hole, so
  the last line writes JSON without a single escape.
- **A hole holds no string literal**, and one expression only.  A
  value that needs a quote gets a name first.
- **A hole does not escape.**  A string that goes into JSON goes
  through `Json.Quote`, and a value that goes into SQL goes as a
  parameter, never as text in the statement.

The format is part of the program, so the checker reads it:

```m9 X12Hole.m9
MODULE X12Hole ;

(* EXPECT-ERROR: an integer in a hole takes a width and no decimals *)
(* Chapter 12 shows a program that must NOT compile.  A format in a
   hole is held against the type of the value.  Decimals belong to a
   real.  An integer has none, so ${rows:.2} is a mistake about what
   rows is, and the checker says so before the program runs.         *)

IMPORT Io ;

VAR
  rows : I64 ;

BEGIN
  rows := 336 ;
  Io.WriteLine ($'rows ${rows:.2}')      (* refused: an integer has no decimals *)
END X12Hole.
```

```refusal X12Hole
X12Hole.m9:16:26: X12Hole body: an integer in a hole takes a width and no decimals: write ${n:8}, or ${F64 (n):8.2} for a real (par 2.3)
```

Decimals belong to a real.  An integer has none, and `${rows:.2}` is
either a slip or a wrong idea of what `rows` is.  Either way it is
caught before the program runs, with the two ways to mean it.  A
width on a string is refused the same way, and so is a `BYTE` or a
record in a hole: each has more than one text, and M9 does not
choose.

From here on the chapters write their output lines this way.  `Fmt`
stays for a number that is wanted as a string on its own.

## What the checker refuses

`+` joins strings, and a `CHAR` joins as one code point on either
side, as the last line of the output shows.  A NUMBER does not.
`'rows ' + n` would have to choose a format (digits, sign, padding),
and M9 does not choose silently.

```m9 X12Int.m9
MODULE X12Int ;

(* EXPECT-ERROR: cannot concatenate a string with I64 *)
(* Chapter 12 shows a program that must NOT compile.  `+` joins
   strings.  A CHAR is one code point and joins too.  A NUMBER is
   neither.  'rows ' + n would have to choose a format: how many
   digits, a sign, padding.  M9 does not choose silently.
   Fmt.I64Str (n) says how, and the reviewer can see it.             *)

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
X12Int.m9:18:16: X12Int body: cannot concatenate a string with I64
```

A hole says how: `$'rows ${n}'` is the integer as it is, and
`$'mean ${x:.3}'` a real with three decimals.  A reviewer reading
the line can see the format that was chosen.  `Fmt.I64Str (n)` and
`Fmt.Fixed (x, 3)` say the same as calls.

[← Previous: threads, waiting in parallel](11-threads.md)

[Next: procedures, calls, returns and the parameter modes →](13-procedures-modes.md)
