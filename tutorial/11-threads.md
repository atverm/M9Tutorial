# 11 — Threads: waiting in parallel

Building a dataset from the web is mostly **waiting**. A request goes
out, and for a few hundred milliseconds your program has nothing to
do but sit there; then the next one, and the next. Two hundred files
at 300 ms each is a minute of doing nothing, one wait at a time.

Threads fix exactly that. Not by computing faster — the work is not
computation — but by letting the waits overlap: eight requests in
flight, eight waits happening at once, and the whole thing finished
in roughly the time of the slowest one.

Measured, on eight pages from a server that takes 300 ms to answer:

    one at a time   2.41 s
    all at once     0.31 s

That is the entire chapter in two numbers. The rest is how to get
them **without** paying the usual price for concurrency, which is
that the answer starts depending on which thread happened to finish
first.

## The answer does not depend on who finished first

Here is the program. Eight workers, one page each, and the thing to
look at is where the results go.

```m9 C11Fetch.m9
MODULE C11Fetch ;

(* Chapter 11: fetching eight files at once, and getting the same
   answer as fetching them one at a time.

   Build (the store server is started for you when this runs in the
   tutorial; locally, serve examples/data on 18931):

     m9c --make -c Http
     m9c --make -c C11Fetch
     gcc -O2 -flto C11Fetch.o Http.o DynStr.o Io.o Fmt.o \
         m9rt.c tcpshim.c tlsshim.c -lssl -lcrypto -lm -o c11fetch

   Everything a worker writes, it writes AT ITS OWN INDEX, so the
   answer cannot depend on which download finished first -- and the
   sequential pass below proves it by agreeing with the concurrent
   one, byte for byte.                                              *)

IMPORT Http ;
IMPORT Io ;

CONST
  NFetch = 8 ;
  Port = 18931 ;
  Host = '127.0.0.1' ;

TYPE
  (* EVERYTHING THE WORKERS SHARE AND CHANGE LIVES HERE.  A MONITOR
     RECORD's fields are reached only through procedures bound to it,
     and those hold its lock -- which is why the two bound procedures
     below are SHORT and the fetching is not in them.

     Both fields earn their place.  `n` is the finish line.  `next` is
     the next page to claim, and it was a plain field of Job in the
     first version of this example -- so Claim read it, added one and
     wrote it back with no lock at all, from eight threads.  Two
     workers then take the same page and some page is never taken:
     `page 7 status 0 bytes 0`, about one run in twenty, which is the
     worst kind of wrong because nineteen runs say it is right. *)
  Work = MONITOR RECORD
    n : I64 ;                 (* how many workers have finished *)
    next : I64 ;              (* the next page to claim *)
  END ;

  Job = RECORD
    w : Work ;
    status : SLICE OF I64 ;   (* one slot per page, written by index *)
    size : SLICE OF I64 ;
  END ;

(* THE PAGES.  Real files from the chapter's own zarr store, served
   on loopback: eight chunks of the CO2 array. *)
PROCEDURE PathOf (k: I64) : STR =
BEGIN
  CASE k OF
  | 0 : RETURN '/co2.zarr/0.0'
  | 1 : RETURN '/co2.zarr/0.1'
  | 2 : RETURN '/co2.zarr/0.2'
  | 3 : RETURN '/co2.zarr/1.0'
  | 4 : RETURN '/co2.zarr/1.1'
  | 5 : RETURN '/co2.zarr/1.2'
  | 6 : RETURN '/co2.zarr/2.0'
  | 7 : RETURN '/co2.zarr/2.2'
  ELSE RETURN '/'
  END
END PathOf ;

(* one page, into slot k.  Nothing here is shared with another worker
   except the two slices, and each worker touches only its own k. *)
PROCEDURE One (VAR status: SLICE OF I64 ; VAR size: SLICE OF I64 ;
               k: I64) =
VAR
  body : SLICE OF BYTE ;
  pool : POOL ;              (* a pool per call: m9_pool_alloc has no
                                lock, so a worker allocates its own *)
  n, st : I64 ;
BEGIN
  body := NEW (pool, BYTE, 65536) ;
  n := 0 ;
  BEGIN
    st := Http.Get (Host, Port, PathOf (k), body, n)
  EXCEPT
  | Http.TransportError : st := -1
  | ValueRange : st := -2
  END ;
  status[k] := st ;
  size[k] := n
END One ;

PROCEDURE Claim (VAR w: Work) : I64 =
VAR k : I64 ;
BEGIN
  (* bound to the monitor, so the read and the write cannot be split
     by another worker: this is one indivisible claim *)
  k := w.next ;
  w.next := w.next + 1 ;
  RETURN k
END Claim ;

PROCEDURE Finish (VAR w: Work) =
BEGIN
  (* bound to the monitor, and deliberately tiny: the lock is held
     for a counter and a signal, never for a download *)
  w.n := w.n + 1 ;
  SIGNAL (w)
END Finish ;

PROCEDURE Worker (VAR j: Job) =
VAR k : I64 ;
BEGIN
  k := Claim (j.w) ;
  IF k < NFetch THEN One (j.status, j.size, k) END ;
  Finish (j.w)
END Worker ;

PROCEDURE AwaitAll (VAR w: Work ; want: I64) =
BEGIN
  WHILE w.n < want DO
    WAIT (w)
  END
END AwaitAll ;

VAR
  pool : POOL ;
  j : PTR Job ;
  seqStatus, seqSize : SLICE OF I64 ;
  i, differ : I64 ;

BEGIN
  seqStatus := NEW (pool, I64, NFetch) ;
  seqSize := NEW (pool, I64, NFetch) ;

  (* one at a time: the wait for each page happens in turn *)
  i := 0 ;
  WHILE i < NFetch DO
    One (seqStatus, seqSize, i) ;
    i := i + 1
  END ;

  (* all at once: eight threads, one page each.  The waits overlap --
     which is the whole point, because waiting is what an I/O-bound
     program does with its time. *)
  j := NEW (pool, Job) ;
  j.status := NEW (pool, I64, NFetch) ;
  j.size := NEW (pool, I64, NFetch) ;
  (* no `j.w.n := 0` here: par 6 refuses reaching into a monitor from
     outside a bound procedure, and the assignment was writing zero
     over zero anyway -- pool storage is defined-zero (par 4.3). *)
  i := 0 ;
  WHILE i < NFetch DO
    THREAD (Worker, j) ;
    i := i + 1
  END ;
  AwaitAll (j.w, NFetch) ;

  (* THE ANSWER, in page order and not in finishing order *)
  i := 0 ;
  WHILE i < NFetch DO
    Io.Write ('page ') ;
    Io.WriteI64 (i) ;
    Io.Write (' status ') ;
    Io.WriteI64 (j.status[i]) ;
    Io.Write (' bytes ') ;
    Io.WriteI64 (j.size[i]) ;
    Io.WriteLine ('') ;
    i := i + 1
  END ;

  differ := 0 ;
  i := 0 ;
  WHILE i < NFetch DO
    IF j.status[i] # seqStatus[i] THEN differ := differ + 1 END ;
    IF j.size[i] # seqSize[i] THEN differ := differ + 1 END ;
    i := i + 1
  END ;
  Io.Write ('one at a time and all at once agree: ') ;
  IF differ = 0 THEN Io.WriteLine ('yes')
  ELSE Io.WriteLine ('NO') END
END C11Fetch.
```

Three ideas carry it, and they are the whole discipline:

**Every worker writes at its own index.** `status[k]` and `size[k]`,
where `k` is the page this worker claimed. Nobody appends to a shared
list, so the order things arrive in never reaches the answer. The
program prints in page order because the *data* is in page order —
not because anything was sorted afterwards.

**Everything shared and changing is in the monitor, and the monitor
is tiny.** `Work` is a `MONITOR RECORD`: its fields can be touched
only by procedures bound to it, and such a procedure holds its lock
for the whole body. Both of its fields are shared and both change —
`n`, the count of finished workers, and `next`, the page to claim
next — so both are in there, and the two bound procedures that touch
them are two statements each. The downloading happens *outside*, in
`Worker`. Put the fetch inside a bound procedure and eight workers
queue politely behind one lock, which is a very reliable way to make
threading slower than not threading.

`next` is in the monitor because the first version of this example
left it in `Job`, where `Claim` read it, added one and wrote it back
with no lock. Two workers then claim the same page and one page is
claimed by nobody, which prints as `page 7 status 0 bytes 0`. It
survived every run on this machine and failed in CI, because the
threads are started one after another and each begins a slow download
immediately, so the increments almost never overlap. **Almost never is
not a property you can build on**, and it is exactly the failure that
a language cannot catch for you: writing `j.next := j.next + 1` from
two threads is legal M9. Putting the field where only a bound
procedure can reach it is what makes the claim indivisible.

**Each worker allocates from its own pool.** `One` declares
`pool : POOL` as a local, so the arena it carves belongs to that call.
The runtime's allocator takes no lock; a shared pool across threads
would be a race in the runtime rather than in your program.

The last two lines are the proof rather than the claim: the same
eight pages are fetched sequentially first, and the two passes are
compared element by element.

```output C11Fetch
page 0 status 200 bytes 4151
page 1 status 200 bytes 4109
page 2 status 200 bytes 2912
page 3 status 200 bytes 4162
page 4 status 200 bytes 4144
page 5 status 200 bytes 2886
page 6 status 200 bytes 4140
page 7 status 200 bytes 2906
one at a time and all at once agree: yes
```

If you run it, you get those bytes. So does the gate, on every
commit — which is the point of comparing, rather than trusting.

## What the compiler refuses

The mistake threading invites is sharing something mutable by
accident. M9 will not let you:

```m9 X11Share.m9
MODULE X11Share ;

(* EXPECT-ERROR: cannot write through a value parameter: j is a shared borrow *)

(* The mistake threading invites, refused before it can race.

   A worker is handed a pointer to the job it shares with the others.
   Written as a VALUE parameter, that pointer is a SHARED BORROW
   (report par 4.1): everyone may read it, nobody may write through
   it -- and this worker tries to keep its own tally in the shared
   record.  Two workers doing that is a data race, and a race that
   fires one time in ten is the kind of bug that survives a hundred
   green test runs.

   The fix is one word: VAR, which says out loud that this parameter
   is the mutable handle.  Chapter 11's C11Fetch does exactly that,
   and writes only at its own index.                               *)

IMPORT Io ;

TYPE
  Job = RECORD
    done : I64 ;
  END ;

PROCEDURE Worker (j: PTR Job) =
BEGIN
  j.done := j.done + 1
END Worker ;

VAR
  pool : POOL ;
  p : PTR Job IN pool ;

BEGIN
  p := NEW (pool, Job) ;
  p.done := 0 ;
  Worker (p) ;
  Io.WriteLine ('never gets here')
END X11Share.
```

```refusal X11Share
28:3 X11Share.Worker: cannot write through a value parameter: j is a shared borrow (take VAR, par 4.1)
```

A `PTR` parameter written without `VAR` is a **shared borrow**
(report §4.1): everyone may read it, nobody may write through it. The
fix is one word — `VAR`, which says out loud that this parameter is
the mutable handle. That is why `Worker` in the example takes
`VAR j: Job` while `One` touches nothing but its own slot.

This is a compile error, not a race you find in production at the
hundredth run. It is the same rule you met in chapter 4, doing a job
you could not see there.

**And notice exactly how far it goes**, because the example above paid
for the distinction. `VAR` settles *who may write*. It says nothing
about *two threads writing at once*: the racy `j.next := j.next + 1`
took `VAR j: Job` and compiled, as it should. The compiler stops you
sharing what you did not mean to share; the monitor is how you say
what happens when several threads reach the same field on purpose.
Two different jobs, and you need both.

## `[SERIAL]`, and what it costs you today

Every foreign C procedure in M9 is declared `[SERIAL]` or
`[REENTRANT]`, and the annotation is not documentation — the
generator emits a monitor around every `[SERIAL]` call, so declaring
a library serial makes it safe to call from threads and makes those
calls queue.

The socket layer this chapter uses says:

| procedure | annotation |
|---|---|
| `Connect` | `[SERIAL]` — conservative, until its thread-safety is audited |
| `Recv`, `Send`, `Close` | `[REENTRANT]` |
| `TlsConnect`, `TlsRead`, `TlsWrite`, `TlsClose` | `[REENTRANT]` |

So both plain HTTP and HTTPS overlap their waiting; only the brief
plain `connect` is serialised, and that one is marked conservatively
rather than because anything is known to be wrong with it.

The TLS row said `[SERIAL]` when this chapter was first written, and
the reason was concrete: the shim keeps a table of handles, and it
claimed a slot without a lock, so two threads could take the same one
and the second would free the first's connection. The annotation was
not a guess — it was the truth about that code. Making it
`[REENTRANT]` meant *changing the code first*: one shared TLS context
built once instead of one per connection, and a mutex held for the
table bookkeeping only, never across a handshake, a read or a write.

The point for you is what did **not** have to change: no program that
used `Http.GetTls` was edited. The contract lives in the declaration,
the compiler enforces what the declaration says, and when the thing
behind it got better every caller got better with it. That is what an
annotation is for — and if it had been a comment rather than a
checked declaration, nobody could have told you which programs were
safe in the meantime.

## One thing to know before you run this in a container

Threads and a tight memory limit interact in a way that has nothing
to do with M9 and will bite you anyway.  glibc gives each thread that
contends for `malloc` its own **arena**, and every arena RESERVES
64 MB of address space -- not memory it uses, address space it claims.
Eight workers can therefore reserve half a gigabyte while allocating
almost nothing, and under a limit like `ulimit -v` (or a container
with a small address-space cap) the program dies with `OutOfMemory`
having done nothing wrong.

The cells on this page run in a sandbox capped at 256 MB, and this
example failed about one run in three until the cap was set:

    MALLOC_ARENA_MAX=2 ./myprogram

Measured here: 20 of 20 runs failed without it, 0 of 20 with it.  The
symptom is worth recognising because it is so misleading -- a program
that allocates a few kilobytes, reporting that it is out of memory,
intermittently.

## What threads are not for

Not for making arithmetic faster in a tutorial-sized program: eight
threads adding numbers spend more time starting than working. Not for
hiding a slow algorithm. And not for anything where the answer
depends on the order results arrive in — if you find yourself
appending to a shared list from several workers, stop and give each
worker an index, as `C11Fetch` does.

Where they earn their place is exactly where you started: a program
whose time is spent waiting for something else. Downloads, a dozen
files off a slow disk, a hundred queries to a service that thinks for
a moment. Waiting is the resource being parallelised.

[← Previous: a real dataset, end to end](10-real-data.md)
