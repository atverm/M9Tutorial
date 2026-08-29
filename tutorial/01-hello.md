# 1 — Hello, and why M9 exists

M9 exists because of an afternoon in which the same small program —
a reader for a scientific data format — was written twice, in two
respected compiled languages, and both versions shipped bugs their
compilers had every reason to catch.  Each failure was catalogued;
each became a language rule; the failing programs live on in the
repository's `museum/`, where the build proves, forever, that they
no longer compile.  Nothing in M9 is there because it is elegant.
Everything is there because its absence, somewhere, cost somebody a
result.

Before the first example, the philosophy in one paragraph.  **M9 is
not optimised for the writer's convenience.**  It is verbose where
verbosity is information: every import is named, every error a
procedure can raise is in its signature, every allocation says which
pool owns it, every conversion between types is written out.  You
get exactly what is written — and when what is written is wrong, the
compiler refuses it, by name, instead of running something
plausible.  This is a deliberate trade.  Code is written once, but
it is reviewed, re-read, re-run and audited many times, increasingly
by people (and machines) who did not write it.  A language that
front-loads the effort of saying precisely what you mean repays it
every time anyone — a colleague, a reviewer, an AI agent, you in two
years — has to establish what the code actually does.

## The first program

```m9 C1Hello.m9
MODULE C1Hello ;

(* Chapter 1.  A program is a MODULE whose body becomes the entry
   point.  The EXCEPT block at the bottom is the root of the whole
   error story: any exception nothing below handled arrives here,
   and a program says what it does about failure at the one frame
   with no caller left to tell.                                     *)

IMPORT Io ;

VAR
  i : I64 ;

BEGIN
  Io.WriteLine ('hello from M9') ;
  FOR i := 1 TO 3 DO
    Io.Write ('measurement ') ;
    Io.WriteI64 (i) ;
    Io.WriteLine ('')
  END
EXCEPT
| ValueRange :
    Io.ErrLine ('a value did not fit') ;
    Io.Halt (1)
END C1Hello.
```

```output C1Hello
hello from M9
measurement 1
measurement 2
measurement 3
```

Things to notice, because they generalise:

- **`MODULE` … `END name.`** — a compilation unit.  A module with a
  `BEGIN` body is a program; the body is the entry point.
- **`IMPORT Io ;`** — nothing is ambient.  If a module writes to the
  terminal, `Io` is in its imports; a reader learns a module's whole
  outside world from its first lines.
- **Declarations before statements**: `VAR i : I64` declares an
  exact 64-bit signed integer.  There is no `int` whose width
  depends on the machine — that difference once cost a format
  reader a factor-of-two stride error (chapter 2).
- **The `EXCEPT` block at the bottom** is the program's answer to
  "and what if it fails?".  The root frame is the one place with no
  caller left to inform, so it is where a program must decide —
  print, clean up, exit nonzero.  It is not decoration: for some
  operations the checker will not let a program omit the decision.

## What "checked" means here

The founding bug: a GNU Modula-2 build, at optimisation level 2,
**executed `a[42]` on an array of ten elements** and printed the
word "unreachable".  The checks existed at `-O0`; the flag removed
them.  M9's answer is that checks are part of what a program MEANS
— there is no build in which they are absent, any more than there
is a build in which `+` means subtraction.

```m9 C1Bounds.m9
MODULE C1Bounds ;

(* Chapter 1.  The bug that founded the language: a GNU Modula-2
   build at -O2 EXECUTED a[42] on an array of ten and printed
   "unreachable".  In M9 the check is part of what the program
   MEANS, not a compiler flag: this loop runs off the end on
   purpose, and the access raises IndexError instead of reading
   whatever lies past the array.                                    *)

IMPORT Io ;

VAR
  a   : ARRAY 10 OF I64 ;
  i   : I64 ;
  sum : I64 ;

BEGIN
  FOR i := 0 TO 9 DO
    a [i] := i * i
  END ;
  sum := 0 ;
  FOR i := 0 TO 12 DO      (* three past the end, deliberately *)
    sum := sum + a [i]
  END ;
  Io.WriteLine ('never reached')
EXCEPT
| IndexError :
    Io.Write ('IndexError refused the read; the sum so far was ') ;
    Io.WriteI64 (sum) ;
    Io.WriteLine ('')
END C1Bounds.
```

```output C1Bounds
IndexError refused the read; the sum so far was 285
```

The loop deliberately runs to 12 on an array of 10.  The read of
`a[10]` raises `IndexError`; the handler reports the sum of the ten
legal elements (285 = 0² + 1² + … + 9²) and the program exits
cleanly.  No flag, no sanitizer, no "debug build" — this is the
only behaviour the program can have.

The cost of this, measured on real scientific workloads, is a few
percent.  The cost of not having it was the museum.

[← Previous: installing the compiler](00-install.md) · [Next: strong typing →](02-strong-typing.md)
