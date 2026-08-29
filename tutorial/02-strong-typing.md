# 2 — Strong typing

The width of a number is part of its meaning.  The failure behind
this chapter: a Modula-2 compiler mapped `LONGREAL` to the x87
80-bit format, which occupies sixteen bytes in memory — so a loop
reading 8-byte doubles off the wire strode through them at the
wrong width and read garbage from the second element on.  Nothing
warned, because the language's own type said "a long real", and
what that meant depended on the compiler.

In M9 every numeric type names its width — `I8`..`I64`, `U8`..`U64`,
`F32`, `F64`, `BYTE` — and **nothing converts implicitly**, not even
the "safe" direction:

```m9 X2Assign.m9
MODULE X2Assign ;

(* EXPECT-ERROR: cannot assign F64 to I64 *)
(* Chapter 2, a program that must NOT compile.  There are no
   implicit conversions, not even the "safe" widening ones: if two
   widths meet, the program says which one wins, or the checker
   refuses.  gm2's LONGREAL was x87 long double striding 16 bytes
   over 8-byte wire doubles -- silently.  This is the rule that
   makes that failure uncompilable.                                 *)

IMPORT Io ;

VAR
  i : I64 ;
  x : F64 ;

BEGIN
  x := 2.5 ;
  i := x ;                 (* refused: no implicit F64 -> I64 *)
  Io.WriteI64 (i)
END X2Assign.
```

```refusal X2Assign
19:5 X2Assign body: cannot assign F64 to I64 (no implicit conversions, par 2.1)
```

That diagnostic is real — the gate compiles this file on every run
and requires exactly that refusal.  The assignment must be written
as what it is: a conversion, `i := I64 (x)` — which brings us to
the second rule.

## Conversions are calls, and they can fail

`I64 (x)` is an ordinary-looking call with a checked meaning: if
the value does not fit the target, it raises `ValueRange`.  The
museum piece behind it: `Trunc(NaN)` crashed one runtime and, in a
well-known array library, silently answers the most negative
integer — a number that then flows onward into someone's mean.

```m9 C2Widths.m9
MODULE C2Widths ;

(* Chapter 2.  Every integer and real type names its exact width --
   I64, I32, F64, F32 -- and nothing converts by itself.  A
   conversion is a call, the call can fail, and a failing conversion
   RAISES instead of wrapping, truncating, or inventing INT64_MIN
   the way a NaN truncation silently does elsewhere.

   Each probe carries its own EXCEPT block: a handler belongs to the
   frame that knows what failure means there.                       *)

IMPORT Io ;

PROCEDURE TryI32 (v: I64) =
  (* narrows v to 32 bits, reporting instead of propagating.

       v -- any I64; whether it fits I32 is exactly what the
            conversion checks.  This comment block is a DOCSTRING:
            m9c --doc renders it, chapter 3 shows where.            *)
VAR n : I32 ;
BEGIN
  n := I32 (v) ;
  Io.Write ('I32 fits: ') ;
  Io.WriteI64 (I64 (n)) ;
  Io.WriteLine ('')
EXCEPT
| ValueRange :
    Io.Write ('I32 (') ;
    Io.WriteI64 (v) ;
    Io.WriteLine (') raised ValueRange')
END TryI32 ;

PROCEDURE TryTruncNan () =
VAR
  x : F64 ;
  k : I64 ;
BEGIN
  x := 0.0 / 0.0 ;        (* NaN travels freely in floats ... *)
  k := I64 (x) ;          (* ... and raises at the integer boundary *)
  Io.WriteI64 (k)
EXCEPT
| ValueRange :
    Io.WriteLine ('I64 (NaN) raised ValueRange')
END TryTruncNan ;

BEGIN
  TryI32 (2000000000) ;
  TryI32 (3000000000) ;
  TryTruncNan ()
END C2Widths.
```

```output C2Widths
I32 fits: 2000000000
I32 (3000000000) raised ValueRange
I64 (NaN) raised ValueRange
```

Note the shape: each probe procedure carries its own `EXCEPT`
block, because a handler belongs to the frame that knows what the
failure means there.  NaN travels freely between floats — that is
IEEE arithmetic and it is right — and is stopped at the integer
boundary, where "not a number" has no honest representation.

## The accounting is exhaustive

A procedure that performs a checked conversion has two options:
handle the failure, or declare it.  There is no third option in
which the signature stays quiet:

```m9 X2Raises.m9
MODULE X2Raises ;

(* EXPECT-ERROR: unhandled RAISES ValueRange *)
(* Chapter 2, a program that must NOT compile.  A checked conversion
   can raise ValueRange, so a procedure that converts must either
   handle it or declare it in RAISES -- the accounting is exhaustive,
   which is what makes a signature's RAISES list trustworthy.       *)

IMPORT Io ;

PROCEDURE Narrow (v: I64) : I32 =
BEGIN
  RETURN I32 (v)           (* can raise ValueRange: undeclared *)
END Narrow ;

BEGIN
  Io.WriteI64 (I64 (Narrow (70000)))
END X2Raises.
```

```refusal X2Raises
12:1 X2Raises.Narrow: unhandled RAISES ValueRange from I32 conversion
```

This is what makes a RAISES list worth reading: it is not
documentation that can rot, it is checked arithmetic over the call
graph.  When chapter 3 shows a definition promising
`RAISES TooCold`, that promise is proven, not asserted.

## When you MEAN wraparound, say so

Checked `+` raises `Overflow` — always, in every build, for the
reason chapter 1 gave.  Sometimes modular arithmetic is the point
(hashing, checksums, random-number generators).  M9 spells that as
different operators — `+%`, `-%`, `*%` — defined modulo 2^64, so
the intent is visible in the source and greppable forever:

```m9 C2Wrap.m9
MODULE C2Wrap ;

(* Chapter 2.  `+` on integers is CHECKED: overflow raises, always,
   because that is what the operator means -- there is no release
   mode where it stops meaning that.  When wrapping is what you
   want, say so: `+%` is a different operator, defined modulo 2^64,
   and a reader can grep for the percent sign.                      *)

IMPORT Io ;

PROCEDURE Checked (a, b: I64) =
VAR s : I64 ;
BEGIN
  s := a + b ;
  Io.Write ('a + b = ') ;
  Io.WriteI64 (s) ;
  Io.WriteLine ('')
EXCEPT
| Overflow :
    Io.WriteLine ('a + b raised Overflow')
END Checked ;

VAR
  top : I64 ;

BEGIN
  top := MAX (I64) ;
  Checked (top, 0) ;
  Checked (top, 1) ;             (* checked: refuses *)
  Io.Write ('top +% 1 = ') ;
  Io.WriteI64 (top +% 1) ;       (* wrapping: stated in the operator *)
  Io.WriteLine ('')
END C2Wrap.
```

```output C2Wrap
a + b = 9223372036854775807
a + b raised Overflow
top +% 1 = -9223372036854775808
```

`MAX (I64) +% 1` lands exactly on the most negative I64, printed
with all its digits.

[← Previous: hello, and why M9 exists](01-hello.md) · [Next: definition and implementation →](03-definition-implementation.md)
