# 3 — Definition and implementation

A module has two parts, usually kept in one file: the DEFINITION,
which is everything a caller may rely on, and the IMPLEMENTATION,
which is nobody else's business.  The definition is a contract in
the enforceable sense: the checker compares every implemented
procedure against its declared signature and refuses drift, and a
promised procedure that is never implemented is an error, not a
linker surprise or a TODO.

Here is a small library — temperature handling with a physical
floor — followed by a client.  Notice that the definition carries
the EXCEPTION with its payload fields, and every procedure's
complete RAISES list:

```m9 Temps.m9
DEFINITION MODULE Temps ;

(* Chapter 3's library: temperature conversions with a floor.  The
   definition IS the contract -- what exists, what it needs, and
   exactly what can go wrong.  The RAISES list is not documentation
   that can rot: the checker proves the implementation raises
   nothing the definition did not admit to.                         *)

EXCEPTION
  TooCold (got, limit: F64) ;

PROCEDURE ToKelvin (celsius: F64) : F64 RAISES TooCold ;
  (* refuses anything below absolute zero.

       celsius -- the reading as measured; -273.15 is the floor,
                  and the exception carries both it and the value
                  that broke it.                                    *)

PROCEDURE Mean (RO readings: SLICE OF F64) : F64 RAISES TooCold ;
  (* the mean of a series, every element validated on the way in.  *)

END Temps.

IMPLEMENTATION MODULE Temps ;

CONST
  Zero = 273.15 ;

PROCEDURE ToKelvin (celsius: F64) : F64 RAISES TooCold =
BEGIN
  IF celsius < 0.0 - Zero THEN
    RAISE TooCold (celsius, 0.0 - Zero)
  END ;
  RETURN celsius + Zero
END ToKelvin ;

PROCEDURE Mean (RO readings: SLICE OF F64) : F64 RAISES TooCold =
VAR
  sum : F64 ;
  i   : I64 ;
BEGIN
  sum := 0.0 ;
  FOR i := 0 TO LEN (readings) - 1 DO
    sum := sum + ToKelvin (readings [i])
  END ;
  RETURN sum / F64 (LEN (readings))
END Mean ;

END Temps.
```

Two halves, one file, and the boundary is real: `Zero` is a
constant of the implementation; no client can see it.  The comments
under each definition are not just prose either — `m9c --doc`
renders them, per procedure and per parameter, into the module's
reference page; the library documentation you will meet in later
chapters is generated exactly that way, from files exactly like
this.

## What the contract refuses

Change the implementation's parameter type and the module no longer
compiles:

```m9 X3Sig.m9
DEFINITION MODULE X3Sig ;

(* EXPECT-ERROR: signature differs from definition *)
(* Chapter 3, a module that must NOT compile.  The definition says
   ToKelvin takes an F64; the implementation drifts to F32.  In a
   dynamic language this is a latent runtime surprise; here the two
   halves are compared as canonical signatures and the drift is a
   compile error naming the procedure.                              *)

PROCEDURE ToKelvin (celsius: F64) : F64 ;

END X3Sig.

IMPLEMENTATION MODULE X3Sig ;

PROCEDURE ToKelvin (celsius: F32) : F64 =
BEGIN
  RETURN F64 (celsius) + 273.15
END ToKelvin ;

END X3Sig.
```

```refusal X3Sig
16:1 X3Sig.ToKelvin: signature differs from definition:
    definition     (celsius: F64) : F64
    implementation (celsius: F32) : F64
```

Omit a promised procedure and it is named:

```m9 X3Missing.m9
DEFINITION MODULE X3Missing ;

(* EXPECT-ERROR: not implemented *)
(* Chapter 3, a module that must NOT compile.  A definition is a
   promise; an implementation that omits a promised procedure is
   refused, so "TODO" cannot ship silently.                         *)

PROCEDURE ToKelvin (celsius: F64) : F64 ;
PROCEDURE ToFahrenheit (celsius: F64) : F64 ;

END X3Missing.

IMPLEMENTATION MODULE X3Missing ;

PROCEDURE ToKelvin (celsius: F64) : F64 =
BEGIN
  RETURN celsius + 273.15
END ToKelvin ;

END X3Missing.
```

```refusal X3Missing
13:1 X3Missing.ToFahrenheit: declared in the definition but not implemented
```

## The client's side of the contract

```m9 C3Use.m9
MODULE C3Use ;

(* Chapter 3.  A client sees only the definition: Temps.ToKelvin can
   raise TooCold and the checker will not let this program pretend
   otherwise -- the handler below is not politeness, it is what made
   the module compile.  The payload binds by name and position, so
   the message can say WHICH value broke WHAT limit.                *)

IMPORT Io ;
IMPORT Fmt ;
IMPORT Temps ;

VAR
  pool : POOL ;
  r    : ARRAY 3 OF F64 ;

BEGIN
  r [0] := 5.5 ;  r [1] := -2.0 ;  r [2] := 11.25 ;
  Io.Write ('mean of the series: ') ;
  Io.Write (Fmt.Fixed (pool, Temps.Mean (r), 2)) ;
  Io.WriteLine (' K') ;
  r [1] := -300.0 ;                (* not a temperature *)
  Io.Write (Fmt.Fixed (pool, Temps.Mean (r), 2)) ;
  Io.WriteLine (' never printed')
EXCEPT
| Temps.TooCold (got, limit) :
    Io.Write ('TooCold: ') ;
    Io.Write (Fmt.Fixed (pool, got, 1)) ;
    Io.Write (' is below ') ;
    Io.WriteLine (Fmt.Fixed (pool, limit, 2))
| ValueRange :
    Io.ErrLine ('formatting failed') ;
    Io.Halt (1)
END C3Use.
```

```output C3Use
mean of the series: 278.07 K
TooCold: -300.0 is below -273.15
```

The handler arm `Temps.TooCold (got, limit)` binds the payload the
exception was declared with, so the message can say WHICH reading
broke WHAT limit — errors are values here, with structure, not
strings fished out of a log.  And the handler is not optional
politeness: `Mean` declares `RAISES TooCold`, this program's body
calls it, and the checker required this frame to either declare the
exception onward or answer for it.  Delete the handler and the
module joins the X-files.

The deeper point of this chapter: **the signature is the review.**
A reader deciding whether to trust `Temps.Mean` reads one
declaration and knows its inputs, its output, its failure modes,
and (chapter 5) who owns its storage.  Nothing else in the file can
contradict that declaration and still compile.

[← Previous: strong typing](02-strong-typing.md) · [Next: memory, pools and strings →](04-memory.md)
