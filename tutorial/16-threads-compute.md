# 16. Threads for computation, and what the cores actually give you

Chapter 11 used threads to overlap *waiting*: eight downloads at
once. This chapter uses them to overlap *arithmetic* — a matrix
product, a matrix inversion, and a hash over a whole file — on a
laptop with eight cores, and measures what that buys. The answer is
"it depends on the work", with numbers, which is more useful than
"faster".

The discipline is the one chapter 11 taught, and it does not change
for arithmetic:

- cut the work into bands that touch **disjoint** parts of the result
  — rows of the product, columns of the inverse, chunks of the file;
- every worker claims its band through a **monitor**, computes it in
  its **own pool**, and writes only its own part of the result;
- the main thread **waits on the monitor**, then reads.

Nothing is both shared and written, so nothing is locked during the
arithmetic, and the answer cannot depend on which band finished first.

## Two matrices, banded

```m9 C16Mat.m9
MODULE C16Mat ;

(* Chapter 16: computation goes faster with threads when there are
   cores to run them on -- a matrix product and a matrix inversion,
   each done once on one thread and once on eight, and the two
   answers compared BIT FOR BIT.

   The shape of every threaded computation here is the same:
     * the work is cut into bands that touch DISJOINT parts of the
       result -- rows of the product, columns of the inverse;
     * each worker claims a band through a monitor, computes it in
       its own pool, and copies it into its own part of the result;
     * the main thread waits on the monitor and then reads.
   Nothing is shared and written by two threads, so there is nothing
   to lock during the arithmetic, and the answer cannot depend on
   which band finished first.  Every band does the SAME arithmetic
   in the SAME order as the serial version, which is why the check
   at the end can demand identical bytes rather than a tolerance.  *)

IMPORT Io ;
IMPORT Fmt ;
IMPORT Mat ;
IMPORT Time ;

CONST
  N = 800 ;                    (* 800 x 800: half a billion multiply-adds *)
  Bands = 8 ;

TYPE
  (* the finish line and the band counter -- everything the workers
     share AND change lives in the monitor, and nowhere else *)
  Work = MONITOR RECORD
    n : I64 ;                  (* workers finished *)
    next : I64 ;               (* the next band to claim *)
  END ;

  MulJob = RECORD
    w : Work ;
    a, b : PTR Mat.Matrix ;    (* read by every worker, written by none *)
    prod : PTR Mat.Matrix ;    (* rows [r0, r1) of it belong to band k *)
  END ;

  InvJob = RECORD
    w : Work ;
    l : PTR Mat.Matrix ;       (* the Cholesky factor, read by all *)
    inv : PTR Mat.Matrix ;     (* columns [c0, c1) of it belong to band k *)
  END ;

(* --- the three bound procedures, and they are all tiny --- *)
PROCEDURE Claim (VAR w: Work) : I64 =
VAR k : I64 ;
BEGIN
  k := w.next ;
  w.next := w.next + 1 ;
  RETURN k
END Claim ;

PROCEDURE Finish (VAR w: Work) =
BEGIN
  w.n := w.n + 1 ;
  SIGNAL (w)
END Finish ;

PROCEDURE AwaitAll (VAR w: Work ; want: I64) =
BEGIN
  WHILE w.n < want DO WAIT (w) END
END AwaitAll ;

(* band k of N rows is [k*N/Bands, (k+1)*N/Bands) *)
PROCEDURE Lo (k: I64) : I64 =
BEGIN
  RETURN k * N DIV Bands
END Lo ;

PROCEDURE Hi (k: I64) : I64 =
BEGIN
  RETURN (k + 1) * N DIV Bands
END Hi ;

(* --- the product, one row band per worker --- *)
PROCEDURE MulWorker (VAR j: MulJob) =
VAR
  wpool : POOL ;               (* a pool per worker: the allocator is not locked *)
  band, p : PTR Mat.Matrix IN wpool ;
  k, r0, r1, r, c : I64 ;
BEGIN
  k := Claim (j.w) ;
  IF k < Bands THEN
    r0 := Lo (k) ; r1 := Hi (k) ;
    BEGIN
      (* my rows of A, then the whole of B: the same MulM the serial
         version calls, on a slice of the problem *)
      band := Mat.New (wpool, r1 - r0, N) ;
      FOR r := 0 TO r1 - r0 - 1 DO
        FOR c := 0 TO N - 1 DO
          Mat.Set (band, r, c, Mat.Get (j.a, r0 + r, c))
        END
      END ;
      p := Mat.MulM (wpool, band, j.b) ;
      (* into MY rows of the shared product, and no one else's *)
      FOR r := 0 TO r1 - r0 - 1 DO
        FOR c := 0 TO N - 1 DO
          Mat.Set (j.prod, r0 + r, c, Mat.Get (p, r, c))
        END
      END
    EXCEPT
    | Mat.SizeError : Io.ErrLine ('a band had the wrong shape') ; Io.Halt (2)
    END
  END ;
  Finish (j.w)
END MulWorker ;

(* --- the inverse: factor once, then solve one column band per
   worker.  Mat.SpdInverse is exactly Cholesky followed by CholSolve
   on the identity, and CholSolve treats each column on its own, so a
   band of identity columns solved here is the same arithmetic. --- *)
PROCEDURE InvWorker (VAR j: InvJob) =
VAR
  wpool : POOL ;
  id, x : PTR Mat.Matrix IN wpool ;
  k, c0, c1, r, c : I64 ;
BEGIN
  k := Claim (j.w) ;
  IF k < Bands THEN
    c0 := Lo (k) ; c1 := Hi (k) ;
    BEGIN
      id := Mat.New (wpool, N, c1 - c0) ;       (* pool storage is zero *)
      FOR c := 0 TO c1 - c0 - 1 DO Mat.Set (id, c0 + c, c, 1.0) END ;
      x := Mat.CholSolve (wpool, j.l, id) ;
      FOR r := 0 TO N - 1 DO
        FOR c := 0 TO c1 - c0 - 1 DO
          Mat.Set (j.inv, r, c0 + c, Mat.Get (x, r, c))
        END
      END
    EXCEPT
    | Mat.SizeError : Io.ErrLine ('a band had the wrong shape') ; Io.Halt (2)
    END
  END ;
  Finish (j.w)
END InvWorker ;

PROCEDURE Identical (a, b: PTR Mat.Matrix) : BOOL =
VAR r, c : I64 ;
BEGIN
  IF (Mat.Rows (a) # Mat.Rows (b)) OR (Mat.Cols (a) # Mat.Cols (b)) THEN
    RETURN FALSE
  END ;
  FOR r := 0 TO Mat.Rows (a) - 1 DO
    FOR c := 0 TO Mat.Cols (a) - 1 DO
      IF Mat.Get (a, r, c) # Mat.Get (b, r, c) THEN RETURN FALSE END
    END
  END ;
  RETURN TRUE
END Identical ;

(* how far a product is from the identity, as a single number *)
PROCEDURE FarFromIdentity (m: PTR Mat.Matrix) : F64 =
VAR
  r, c : I64 ;
  want, d, worst : F64 ;
BEGIN
  worst := 0.0 ;
  FOR r := 0 TO Mat.Rows (m) - 1 DO
    FOR c := 0 TO Mat.Cols (m) - 1 DO
      IF r = c THEN want := 1.0 ELSE want := 0.0 END ;
      d := Mat.Get (m, r, c) - want ;
      IF d < 0.0 THEN d := -d END ;
      IF d > worst THEN worst := d END
    END
  END ;
  RETURN worst
END FarFromIdentity ;

VAR
  pool : POOL ;
  a, b, at, s, prodS, invS, chk : PTR Mat.Matrix IN pool ;
  mj : PTR MulJob IN pool ;
  ij : PTR InvJob IN pool ;
  r, c, i : I64 ;
  t0, t1 : Time.Instant ;
  serial, par : F64 ;

BEGIN
  BEGIN
    (* two matrices with a pattern rather than random numbers, so the
       whole run is reproducible with no seed to carry *)
    a := Mat.New (pool, N, N) ;
    b := Mat.New (pool, N, N) ;
    FOR r := 0 TO N - 1 DO
      FOR c := 0 TO N - 1 DO
        Mat.Set (a, r, c, F64 ((r * 7 + c * 3) MOD 11) / 10.0) ;
        Mat.Set (b, r, c, F64 ((r * 5 + c * 2) MOD 13) / 10.0)
      END
    END ;

    (* ---- 1. the product, serial then banded ---- *)
    t0 := Time.Now () ;
    prodS := Mat.MulM (pool, a, b) ;
    t1 := Time.Now () ;
    serial := Time.Elapsed (t0, t1) ;

    mj := NEW (pool, MulJob) ;
    mj.a := a ; mj.b := b ;
    mj.prod := Mat.New (pool, N, N) ;
    t0 := Time.Now () ;
    FOR i := 1 TO Bands DO THREAD (MulWorker, mj) END ;
    AwaitAll (mj.w, Bands) ;
    t1 := Time.Now () ;
    par := Time.Elapsed (t0, t1) ;
    (* timings only when asked (any argument): a number that changes
       from run to run has no place in gated output *)
    IF Io.ArgCount () > 1 THEN
      Io.ErrLine ('product: serial ' + Fmt.Fixed (serial, 3) + ' s, '
                  + Fmt.I64Str (Bands) + ' bands ' + Fmt.Fixed (par, 3)
                  + ' s, ' + Fmt.Fixed (serial / par, 1) + 'x')
    END ;
    IF Identical (prodS, mj.prod) THEN
      Io.WriteLine ('product of two ' + Fmt.I64Str (N) + 'x' + Fmt.I64Str (N)
                    + ' matrices in ' + Fmt.I64Str (Bands)
                    + ' bands: identical to the serial product, bit for bit')
    ELSE
      Io.WriteLine ('product: the bands DIFFER from the serial product')
    END ;

    (* ---- 2. the inverse of an SPD matrix, serial then banded ----
       S = A^T A + N I is symmetric positive definite by construction *)
    at := Mat.Transpose (pool, a) ;
    s := Mat.MulM (pool, at, a) ;
    FOR r := 0 TO N - 1 DO
      Mat.Set (s, r, r, Mat.Get (s, r, r) + F64 (N))
    END ;

    t0 := Time.Now () ;
    invS := Mat.SpdInverse (pool, s) ;
    t1 := Time.Now () ;
    serial := Time.Elapsed (t0, t1) ;

    ij := NEW (pool, InvJob) ;
    t0 := Time.Now () ;
    ij.l := Mat.Cholesky (pool, s) ;            (* once, on this thread *)
    ij.inv := Mat.New (pool, N, N) ;
    FOR i := 1 TO Bands DO THREAD (InvWorker, ij) END ;
    AwaitAll (ij.w, Bands) ;
    t1 := Time.Now () ;
    par := Time.Elapsed (t0, t1) ;
    IF Io.ArgCount () > 1 THEN
      Io.ErrLine ('inverse: serial ' + Fmt.Fixed (serial, 3) + ' s, '
                  + Fmt.I64Str (Bands) + ' bands ' + Fmt.Fixed (par, 3)
                  + ' s, ' + Fmt.Fixed (serial / par, 1) + 'x')
    END ;
    IF Identical (invS, ij.inv) THEN
      Io.WriteLine ('inverse of a ' + Fmt.I64Str (N) + 'x' + Fmt.I64Str (N)
                    + ' SPD matrix in ' + Fmt.I64Str (Bands)
                    + ' column bands: identical to SpdInverse, bit for bit')
    ELSE
      Io.WriteLine ('inverse: the bands DIFFER from SpdInverse')
    END ;

    (* and it IS an inverse: S times it is the identity to rounding *)
    chk := Mat.MulM (pool, s, ij.inv) ;
    IF FarFromIdentity (chk) < 1.0e-9 THEN
      Io.WriteLine ('S times the inverse is the identity to within 1e-9: yes')
    ELSE
      Io.WriteLine ('S times the inverse is the identity to within 1e-9: NO')
    END
  EXCEPT
  | Mat.SizeError : Io.ErrLine ('a matrix had the wrong shape') ; Io.Halt (1)
  | Mat.NotSPD : Io.ErrLine ('S is not positive definite') ; Io.Halt (1)
  | ValueRange : Io.ErrLine ('a value out of range') ; Io.Halt (1)
  END
END C16Mat.
```

The product is banded by **rows**: worker *k* copies its rows of A
into a matrix of its own and calls the same `Mat.MulM` the serial
version calls, on that band and the whole of B. The inverse is banded
by **columns**, and it works because of how `Mat.SpdInverse` is built:
a Cholesky factorisation once, then `Mat.CholSolve` on the identity
matrix — and `CholSolve` treats each column of its right-hand side on
its own. So the main thread factors once, and each worker solves its
own band of identity columns with the same procedure.

That is why the program can demand **identical bytes** rather than a
tolerance, and gets them: every element is computed by the same
arithmetic in the same order as the serial version. The check that
S times the inverse is the identity to 10⁻⁹ is the sanity check that
the inverse is an inverse at all.

```output C16Mat
product of two 800x800 matrices in 8 bands: identical to the serial product, bit for bit
inverse of a 800x800 SPD matrix in 8 column bands: identical to SpdInverse, bit for bit
S times the inverse is the identity to within 1e-9: yes
```

## A file, hashed in chunks

```m9 C16Hash.m9
MODULE C16Hash ;

(* Chapter 16, second half: a hash sum over a whole file, computed in
   parallel chunks -- and the one rule that makes a parallel sum
   reproducible: the pieces are combined in a FIXED order.

   The hash is a polynomial over the bytes, h := h * P + byte, in
   64-bit arithmetic that wraps -- M9 spells that `*%` and `+%`, the
   operators that say out loud they are modular, where ordinary `*`
   would raise Overflow.  Each worker hashes its own chunk into its
   own slot of the result; the main thread then folds the slots in
   chunk order.  Fold them in another order and the number changes,
   which the program shows on purpose: the order is part of what the
   hash IS, not an implementation detail.

   The shipped week is half a megabyte and hashes in a millisecond
   either way; the chapter's timings come from the full 186 MB file,
   given as an argument.                                             *)

IMPORT Io ;
IMPORT Fmt ;
IMPORT Time ;

CONST
  Default = 'data/fluxnet_week.csv' ;
  Chunks = 8 ;
  P = 1000003 ;                  (* a prime, and the multiplier *)

TYPE
  Work = MONITOR RECORD
    n : I64 ;
    next : I64 ;
  END ;

  Job = RECORD
    w : Work ;
    bytes : SLICE OF BYTE ;      (* the file: read by all, written by none *)
    part : SLICE OF I64 ;        (* one slot per chunk, written by index *)
  END ;

PROCEDURE Claim (VAR w: Work) : I64 =
VAR k : I64 ;
BEGIN
  k := w.next ;
  w.next := w.next + 1 ;
  RETURN k
END Claim ;

PROCEDURE Finish (VAR w: Work) =
BEGIN
  w.n := w.n + 1 ;
  SIGNAL (w)
END Finish ;

PROCEDURE AwaitAll (VAR w: Work ; want: I64) =
BEGIN
  WHILE w.n < want DO WAIT (w) END
END AwaitAll ;

(* the hash of one run of bytes.  `*%` and `+%` wrap mod 2^64 by
   definition; the result is a bit pattern held in an I64. *)
PROCEDURE HashOf (RO b: SLICE OF BYTE) : I64 =
VAR
  h, i : I64 ;
BEGIN
  h := 0 ;
  FOR i := 0 TO LEN (b) - 1 DO
    h := h *% P +% I64 (b[i])
  END ;
  RETURN h
END HashOf ;

(* chunk k of n bytes is [k*n/Chunks, (k+1)*n/Chunks) *)
PROCEDURE ChunkOf (RO b: SLICE OF BYTE ; k: I64) : SLICE OF BYTE =
VAR lo, hi : I64 ;
BEGIN
  lo := k * LEN (b) DIV Chunks ;
  hi := (k + 1) * LEN (b) DIV Chunks ;
  RETURN SLICE (b, lo, hi - lo)
END ChunkOf ;

PROCEDURE Worker (VAR j: Job) =
VAR k : I64 ;
BEGIN
  k := Claim (j.w) ;
  IF k < Chunks THEN
    j.part[k] := HashOf (ChunkOf (j.bytes, k))     (* my slot, only mine *)
  END ;
  Finish (j.w)
END Worker ;

(* the fold: chunk hashes into one number, in chunk order *)
PROCEDURE Combine (RO part: SLICE OF I64 ; reversed: BOOL) : I64 =
VAR
  total, k : I64 ;
BEGIN
  total := 0 ;
  IF reversed THEN
    FOR k := LEN (part) - 1 TO 0 BY -1 DO total := total *% P +% part[k] END
  ELSE
    FOR k := 0 TO LEN (part) - 1 DO total := total *% P +% part[k] END
  END ;
  RETURN total
END Combine ;

VAR
  pool : POOL ;
  path : STR ;
  bytes : SLICE OF BYTE ;
  serialPart : SLICE OF I64 ;
  j : PTR Job IN pool ;
  k, i : I64 ;
  t0, t1 : Time.Instant ;
  serial, par : F64 ;

BEGIN
  BEGIN
    path := Default ;
    IF Io.ArgCount () > 1 THEN path := Io.Arg (pool, 1) END ;
    bytes := Io.ReadFileBytes (pool, path) ;
    Io.WriteLine ('file: ' + Fmt.I64Str (LEN (bytes)) + ' bytes in '
                  + Fmt.I64Str (Chunks) + ' chunks') ;

    (* ---- one thread: the chunks in order, then the fold ---- *)
    serialPart := NEW (pool, I64, Chunks) ;
    t0 := Time.Now () ;
    FOR k := 0 TO Chunks - 1 DO
      serialPart[k] := HashOf (ChunkOf (bytes, k))
    END ;
    t1 := Time.Now () ;
    serial := Time.Elapsed (t0, t1) ;

    (* ---- eight threads: the same chunks, whoever gets there first,
       into slots that sit in chunk order whatever the finishing
       order was ---- *)
    j := NEW (pool, Job) ;
    j.bytes := bytes ;
    j.part := NEW (pool, I64, Chunks) ;
    t0 := Time.Now () ;
    FOR i := 1 TO Chunks DO THREAD (Worker, j) END ;
    AwaitAll (j.w, Chunks) ;
    t1 := Time.Now () ;
    par := Time.Elapsed (t0, t1) ;
    (* timings only for a file named on the command line: the shipped
       week hashes in a millisecond, and a number that changes from
       run to run has no place in gated output *)
    IF Io.ArgCount () > 1 THEN
      Io.ErrLine ('hash: serial ' + Fmt.Fixed (serial, 3) + ' s, '
                  + Fmt.I64Str (Chunks) + ' threads ' + Fmt.Fixed (par, 3)
                  + ' s, ' + Fmt.Fixed (serial / par, 1) + 'x')
    END ;

    Io.WriteLine ('hash sum, one thread:     ' + Fmt.I64Str (Combine (serialPart, FALSE))) ;
    Io.WriteLine ('hash sum, eight threads:  ' + Fmt.I64Str (Combine (j.part, FALSE))) ;
    IF Combine (serialPart, FALSE) = Combine (j.part, FALSE) THEN
      Io.WriteLine ('identical: yes')
    ELSE
      Io.WriteLine ('identical: NO')
    END ;
    Io.WriteLine ('the same slots folded in reverse order: '
                  + Fmt.I64Str (Combine (j.part, TRUE))
                  + '  -- a different number, so the order is part of the definition')
  EXCEPT
  | Io.IOError : Io.ErrLine ('cannot read ' + path) ; Io.Halt (1)
  | ValueRange : Io.ErrLine ('a value out of range') ; Io.Halt (1)
  END
END C16Hash.
```

The hash is a polynomial over the bytes, `h := h *% P +% byte`, in
arithmetic that *wraps*. M9 spells that `*%` and `+%` — operators
that say out loud they are modular — where the ordinary `*` and `+`
would raise `Overflow` the first time the value passed 2⁶³. Each
worker hashes one chunk into its own slot; the main thread folds the
slots in chunk order.

The last line of the output is the point of the second half: the
same eight slots folded in the *reverse* order give a different
number. The order in which parallel results are combined is part of
what the result *is*, and a program that lets "whichever finished
first" decide it will give a different answer on a different day.
Chapter 11's rule — every worker writes at its own index — is what
keeps the order fixed here.

```output C16Hash
file: 494494 bytes in 8 chunks
hash sum, one thread:     2587437781992675025
hash sum, eight threads:  2587437781992675025
identical: yes
the same slots folded in reverse order: -3294561009219321149  -- a different number, so the order is part of the definition
```

## What eight cores gave, measured

On this laptop (AMD Ryzen 7 8840U: 8 cores, 16 hardware threads,
under WSL2), the timings each program prints to stderr, best of two:

| work | one thread | 8 bands | speed-up |
|---|---|---|---|
| 800×800 product | 0.42 s | 0.13 s | 3.1× |
| 800×800 SPD inverse | 0.55 s | 0.20 s | 2.8× |
| hash of 186 MB | 0.17 s | 0.024 s | 7.3× |

Three things worth knowing about that table.

**The hash scales almost perfectly and the matrices do not**, on the
same cores. The hash is a simple integer loop streaming bytes; the
matrix work is floating-point with a checked array access per
element, and a laptop drops its clock under that kind of load on all
cores at once. The single-thread time was measured at boost clock;
the eight-band time at all-core clock. That ceiling belongs to the
machine, not the program.

**My first explanation was wrong, and measuring said so.** The
obvious suspect was `Mat.MulM`'s loop order: its innermost loop walks
B down a *column*, a stride of 800 doubles, so the multiply looked
memory-bound and eight threads would simply share the bottleneck.
Rewritten to walk both operands along rows, it was **twice as slow**
(0.71 s against 0.35 s) — the row-wise form pays a checked
read-modify-write of the result per inner step, where the column
form accumulates in a register. Same bits either way. The lesson is
the one this whole tutorial keeps returning to: explain after
measuring, not before.

**More bands than cores still helped.** The product went 2.6× at four
bands, 3.1× at eight, 4.5× at sixteen: the hardware threads fill the
latency the checked accesses leave. Whether that holds on your
machine is a two-line change and a stopwatch, which is why the
program prints its own timings.

## Two rules the compiler holds, and two it does not

The monitor's fields are reachable only from procedures bound to it
— `Claim`, `Finish` and `AwaitAll` — and the compiler refuses a
`j.w.next` from anywhere else; that is why nothing initialises the
monitor from the main body (pool storage is already zero). A worker
that raises with nothing to catch it stops the whole program, by
design, so each worker here catches `Mat.SizeError` itself.

What the compiler does *not* check is that the bands are disjoint:
`Mat.Set (j.prod, r0 + r, c, ...)` from eight threads is a data race
if two workers ever share a row, and nothing in the language would
say so. The row range is a contract you hold — the same one chapter
11 states as *every worker writes at its own index* — and the
bit-identical comparison at the end is how the program checks it
kept it.
