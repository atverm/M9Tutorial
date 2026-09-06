# 17 — Living in the real world

Every program in this tutorial so far has been a closed world: it
computed something and printed it, and the only thing that crossed
its boundary was the answer. Real programs are not like that. They
are handed **parameters** by whoever ran them, they **drive other
programs** that already do a job well, and they **read back** what
those programs said. A data pipeline is exactly this shape — one
tool's output is the next tool's input — and most working software
is a data pipeline wearing a coat.

This last chapter builds a small one. It counts how often each word
appears in a passage, the way the classic shell line does —
`sort | uniq -c | sort` — but from M9, and **without a shell**: each
stage is its own program, run through `System.Exec`, fed the previous
stage's output on its standard input. It reads an optional `--top=N`
option, checks that every stage *worked*, and parses the columns that
come back.

```m9 C17Pipe.m9
MODULE C17Pipe ;

(* Chapter 17: living in the real world.

   A program that only ever talks to itself is a demo.  A useful one
   takes PARAMETERS from whoever ran it and DRIVES OTHER PROGRAMS,
   then reads back what they said.  This one counts word frequencies
   the way a shell line would -- sort, then uniq -c, then sort again
   -- but from M9, and without a shell: each stage is its OWN program,
   run through System.Exec, fed the previous stage's output on its
   stdin and INSISTED to have succeeded.

   Three things this chapter is really about:

   * A stage's output comes back as a VALUE, and so does its status.
     System.Exec answers a Result -- the exit status, everything the
     program wrote to stdout, and everything it wrote to stderr -- so
     the pipe between two tools is an M9 string, not a temp file, and
     a stage that FAILED is a status you look at rather than a thing
     you hope did not happen.  A shell one-liner hides the middle of
     the pipe; here every stage is checked.

   * The environment is a boundary, and boundaries lie unless they
     are pinned.  Sort order depends on the locale, so the SAME
     pipeline gives a different answer on a French machine -- unless
     LC_ALL=C nails it to bytes.  Exec sets it as an environment
     OVERRIDE, merged over the inherited environment, so PATH still
     finds the program.  It is the process-level echo of why Fmt owns
     its float printing instead of borrowing a locale-dependent
     printf.

   * A parameter is a DECISION, not an accident.  The count of rows
     to show is a NAMED option, --top=N, the convention System.Value
     defines -- never a bare number that could be mistaken for a
     filename -- and a value that is not a count is reported before
     the default is used.

   With no arguments it shows the top DefaultTop; run it as
   `./c17pipe --top=3` to ask for three.                             *)

IMPORT System ;
IMPORT DynStr ;
IMPORT Io ;
IMPORT Fmt ;

CONST
  DefaultTop = 5 ;

(* the text to count, fixed so the whole run is reproducible.  A
   CONST cannot be a `+` expression, so the passage is a procedure
   that composes its literals into HEAP and answers the whole. *)
PROCEDURE Passage () : STR =
BEGIN
  RETURN
    'the program reads the data and the program checks the data '
    + 'a program that checks its data and checks its result is a '
    + 'program you can trust the data does not lie and neither '
    + 'does the result the program reads the data the program '
    + 'writes the result'
END Passage ;

PROCEDURE IsLetter (c: CHAR) : BOOL =
BEGIN
  RETURN ((c >= 'a') AND (c <= 'z')) OR ((c >= 'A') AND (c <= 'Z'))
END IsLetter ;

PROCEDURE Lower (c: CHAR) : CHAR RAISES ValueRange =
BEGIN
  IF (c >= 'A') AND (c <= 'Z') THEN
    RETURN CHR (ORD (c) + 32)
  END ;
  RETURN c
END Lower ;

(* the passage as one lowercase word per line -- the input a
   sort | uniq pipeline wants.  Anything that is not a letter is a
   word break. *)
PROCEDURE WordsOf (VAR pool: POOL ; RO text: STR) : STR
  RAISES ValueRange, IndexError =
VAR
  d : PTR DynStr.DString IN pool ;
  i : I64 ;
  inWord : BOOL ;
BEGIN
  d := DynStr.New (pool) ;
  inWord := FALSE ;
  FOR i := 0 TO LEN (text) - 1 DO
    IF IsLetter (text[i]) THEN
      DynStr.AppendChar (pool, d, Lower (text[i])) ;
      inWord := TRUE
    ELSIF inWord THEN
      DynStr.AppendChar (pool, d, 0AC) ;   (* end this word *)
      inWord := FALSE
    END
  END ;
  IF inWord THEN DynStr.AppendChar (pool, d, 0AC) END ;
  RETURN DynStr.View (d)
END WordsOf ;

(* a non-negative integer that is the WHOLE of s, or -1 if it is
   not one *)
PROCEDURE ParseInt (RO s: STR) : I64 RAISES IndexError =
VAR
  i, v : I64 ;
BEGIN
  IF LEN (s) = 0 THEN RETURN 0 - 1 END ;
  v := 0 ;
  FOR i := 0 TO LEN (s) - 1 DO
    IF (s[i] < '0') OR (s[i] > '9') THEN RETURN 0 - 1 END ;
    v := v * 10 + (ORD (s[i]) - ORD ('0'))
  END ;
  RETURN v
END ParseInt ;

(* how many rows to show: --top=N, or DefaultTop.  System.Value reads
   the text after --top= in the options, or answers the default we
   name here.  A number the parser rejects is reported before the
   default is used, so a mistyped option is never mistaken for a
   deliberate omission. *)
PROCEDURE TopWanted (VAR pool: POOL) : I64 RAISES ValueRange, IndexError =
VAR
  s : STR ;
  n : I64 ;
BEGIN
  s := System.Value (pool, '--top', '') ;
  IF LEN (s) = 0 THEN RETURN DefaultTop END ;
  n := ParseInt (s) ;
  IF n >= 0 THEN RETURN n END ;
  Io.ErrLine ('--top is not a count; showing ' + Fmt.I64Str (DefaultTop)) ;
  RETURN DefaultTop
END TopWanted ;

(* run one pipeline stage: prog with args, fed `input` on its stdin,
   pinned to LC_ALL=C so the byte ordering does not depend on the
   machine's locale, and INSISTED to have succeeded.  A stage that
   exits nonzero stops the whole run, naming its status and whatever
   it wrote to stderr -- because a pipeline that RAN is not a pipeline
   that WORKED. *)
PROCEDURE Stage (VAR pool: POOL ; RO prog: STR ; RO args: SLICE OF STR ;
                 RO input: STR) : STR
  RAISES ValueRange, Io.IOError =
VAR
  env : SLICE OF STR ;
  r : System.Result ;
BEGIN
  env := NEW (pool, STR, 1) ;
  env[0] := 'LC_ALL=C' ;
  r := System.Exec (pool, prog, args, input, env) ;
  IF r.status # 0 THEN
    Io.ErrLine (prog + ' exited with status ' + Fmt.I64Str (r.status)) ;
    IF LEN (r.err) > 0 THEN Io.ErrLine (r.err) END ;
    Io.Halt (1)
  END ;
  RETURN r.out
END Stage ;

(* uniq -c prints "   COUNT word", the count right-justified in a
   field of blanks.  Parsing a tool's REAL output -- padded columns
   and all -- is most of what living in the real world is.  Show the
   first `top` rows as a ranked table. *)
PROCEDURE Report (VAR pool: POOL ; RO counts: STR ; top: I64)
  RAISES ValueRange, IndexError =
VAR
  i, j, shown, count : I64 ;
  word : STR ;
BEGIN
  Io.WriteLine ('the ' + Fmt.I64Str (top)
                + ' most frequent words in the passage:') ;
  i := 0 ;
  shown := 0 ;
  WHILE (i < LEN (counts)) AND (shown < top) DO
    WHILE (i < LEN (counts)) AND (counts[i] = ' ') DO i := i + 1 END ;
    count := 0 ;
    WHILE (i < LEN (counts)) AND (counts[i] >= '0')
          AND (counts[i] <= '9') DO
      count := count * 10 + (ORD (counts[i]) - ORD ('0')) ;
      i := i + 1
    END ;
    IF (i < LEN (counts)) AND (counts[i] = ' ') THEN i := i + 1 END ;
    j := i ;
    WHILE (j < LEN (counts)) AND (counts[j] # 0AC) DO j := j + 1 END ;
    word := SLICE (counts, i, j - i) ;
    IF LEN (word) > 0 THEN
      shown := shown + 1 ;
      Io.WriteLine ('  ' + Fmt.I64Str (shown) + '. ' + word
                    + '  (' + Fmt.I64Str (count) + ')')
    END ;
    i := j ;
    IF (i < LEN (counts)) AND (counts[i] = 0AC) THEN i := i + 1 END
  END
END Report ;

VAR
  pool : POOL ;
  top : I64 ;
  none, uniqArgs, sortArgs : SLICE OF STR ;
  words, sorted, counted, ranked : STR ;

BEGIN
  BEGIN
    top := TopWanted (pool) ;

    (* the passage as one lowercase word per line, built in memory --
       the input the first stage reads on its stdin *)
    words := WordsOf (pool, Passage ()) ;

    none := NEW (pool, STR, 0) ;
    uniqArgs := NEW (pool, STR, 1) ;
    uniqArgs[0] := '-c' ;
    sortArgs := NEW (pool, STR, 2) ;
    sortArgs[0] := '-k1,1nr' ;
    sortArgs[1] := '-k2,2' ;

    (* three stages, each its own program, each checked.  The output
       of one is the stdin of the next -- an M9 string, not a temp
       file.  The second sort ranks by count (numeric, descending)
       then by the word, so ties have a defined order and every run
       is identical down to the byte. *)
    sorted := Stage (pool, 'sort', none, words) ;
    counted := Stage (pool, 'uniq', uniqArgs, sorted) ;
    ranked := Stage (pool, 'sort', sortArgs, counted) ;

    Report (pool, ranked, top)
  EXCEPT
  | Io.IOError (p) :
      Io.ErrLine ('could not start a pipeline stage:') ;
      Io.ErrLine (p) ;
      Io.Halt (1)
  | ValueRange :
      Io.ErrLine ('a value was out of range') ; Io.Halt (1)
  | IndexError :
      Io.ErrLine ('an index was out of range') ; Io.Halt (1)
  END
END C17Pipe.
```

With no arguments it shows the five most frequent words; the counts
and their order are fixed, because the passage is fixed and the
pipeline is pinned (more on that below).

```output C17Pipe
the 5 most frequent words in the passage:
  1. the  (10)
  2. program  (6)
  3. data  (5)
  4. and  (3)
  5. checks  (3)
```

Run it with a number — `./c17pipe --top=3` — and it shows three. Run
it with a `--top=` that is not a number and it says so and falls back
to the default, rather than guessing.

## The output is a value, and so is the status

`System.Exec` runs a program directly — no shell — and answers a
`Result`: the exit `status`, everything the program wrote to `out`,
and everything it wrote to `err`. So a stage's output is an ordinary
M9 string, and the pipe between two tools is that string handed to the
next stage as its input:

    sorted  := Stage (pool, 'sort', none, words) ;
    counted := Stage (pool, 'uniq', uniqArgs, sorted) ;
    ranked  := Stage (pool, 'sort', sortArgs, counted) ;

There is no `counts.txt` on disk; `sorted` and `counted` are values,
and you can see the handoff in the program instead of inferring it
from a shell line.

And every stage's status is **checked**, inside `Stage`:

    r := System.Exec (pool, prog, args, input, env) ;
    IF r.status # 0 THEN
      Io.ErrLine (prog + ' exited with status ' + Fmt.I64Str (r.status)) ;
      IF LEN (r.err) > 0 THEN Io.ErrLine (r.err) END ;
      Io.Halt (1)
    END ;

A pipeline that *ran* is not a pipeline that *worked*. `sort` fails
if the count of key fields is wrong; `uniq` fails on a bad option; a
typo in a program name fails to start at all. Reading a failed
stage's output would parse whatever it managed to write before it
died, and report a confident wrong answer. Checking the status is the
same discipline this language applies to a slice index or a
conversion, now at the process boundary — and because each stage is
run *as itself*, its status and its stderr are separate values you can
look at.

That is the difference from running the whole line through the shell.
M9 could hand `sort | uniq -c | sort` to `Io.Run` as one string, and
it would work — but `Io.Run` answers **one** status, the shell's,
which is the *last* stage's. A `sort` that failed in the middle of the
pipe is invisible if the final `sort` succeeds, because the shell
throws away the statuses in between. Running each stage as its own
program is what makes every failure a value rather than a thing you
hope did not happen.

## The environment is a boundary, and boundaries lie unless you pin them

Look at how `Stage` builds its environment:

    env := NEW (pool, STR, 1) ;
    env[0] := 'LC_ALL=C' ;
    r := System.Exec (pool, prog, args, input, env) ;

Sorting is locale-dependent: on a machine set to a French or Swedish
locale, `sort` orders letters by that locale's rules, so the *same
pipeline over the same data* produces a different order. A program
whose output depends on an environment variable nobody set on purpose
is a program that will disagree with itself between two machines and
waste an afternoon.

`LC_ALL=C` nails the ordering to raw byte order, everywhere. The `env`
argument is a list of `NAME=VALUE` **overrides**, merged over the
environment the program already has — so this one pins `LC_ALL` and
leaves `PATH` alone, which is what lets `System.Exec` find `sort` by
its bare name in the first place. Replacing the whole environment
would have thrown `PATH` away with it.

This is the same reason [chapter 2](02-strong-typing.md)'s `Fmt` owns
its float printing instead of borrowing C's `printf`: `printf` is
locale-dependent too, and would write `3,14` where you meant `3.14` in
half the world. The rule is one rule — *a boundary that can lie must
be pinned* — and it applies to the process environment exactly as it
applies to the numeric one.

The final `sort` is pinned in another way: `-k1,1nr` sorts by the
first field (the count) numerically and in reverse, and `-k2,2` adds
the word as a tiebreaker. Without that second key, two words with the
same count could come back in either order, and the output would be
stable only by luck. With it, every run is identical down to the
byte — which is what lets this chapter's expected output be an
equality, not an approximation.

## A parameter is a decision

The count of rows to show comes in as a **named option**, and
`System.Value` reads it:

    s := System.Value (pool, '--top', '') ;
    IF LEN (s) = 0 THEN RETURN DefaultTop END ;
    n := ParseInt (s) ;
    IF n >= 0 THEN RETURN n END ;
    Io.ErrLine ('--top is not a count; showing ' + Fmt.I64Str (DefaultTop)) ;
    RETURN DefaultTop

`System.Value (pool, '--top', '')` reads the text after `--top=` in the
options, or answers the default you name — here the empty string, which
`TopWanted` then reads as "no option given" and turns into `DefaultTop`.
The option is `--top=3`, attached with `=`, never a bare `3` sitting on
the command line where nothing could say whether it was a count or a
filename. That is `System`'s whole argument convention: an option
begins with `-`, a value is attached with `=`, and a bare `--` ends the
options. It is small enough to state in full, which is the point.

And a `--top=` whose value is not a count is reported *before* the
default is used, so a mistyped option is never silently mistaken for a
deliberate omission. "No option" is a default the program *decided*,
not one it drifted into.

## Parsing what a tool actually prints

`uniq -c` does not print clean two-column data; it prints the count
**right-justified in a field of blanks**, then a space, then the
word — a format that survives the final sort, so the first lines of
the ranked output look like:

         10 the
          6 program
          5 data

So `Report` skips the leading blanks, reads the digits as the count,
steps over the one separating space, and takes the rest of the line
as the word. Parsing a real tool's real output — padding, separators
and all — instead of wishing it were tidier is most of what living in
the real world is. The bytes are what the tool sends; the program
adapts to them, not the other way around.

## The pipeline is a program you can read

`words`, `sorted`, `counted` and `ranked` are M9 strings, and the three
`Stage` calls are the pipe. What a shell hides inside a single line —
which program runs, what it is fed, whether it succeeded, what it
complained about — is here four named values and three checked calls,
in the open where a reader can see them.

That visibility is the whole idea, and it is where this tutorial
started: M9 is built so that the person reading the program — and the
data crossing its edges — can *see* what happened, and so that the
compiler refuses the ways a boundary might quietly lie. A real program
lives among other programs, arguments and locales, all of which will
hand it something unexpected eventually. The work of the last
seventeen chapters is that when they do, the program says so, at the
line where it happened, instead of computing a confident wrong answer
and printing it as if it were true.
