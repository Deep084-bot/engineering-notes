# Race Condition

> [!NOTE]
> **Module:** Module V – Synchronization
>
> **Difficulty:** ⭐⭐⭐⭐☆
>
> **Prerequisites:**
> - Critical Section
> - Mutex
>
> **Scope:** Linux and C only. No Windows or Java.

---

# Learning Objective

After reading this note, you should be able to:

- Define a race condition precisely, and distinguish it from a data race.
- Explain why a race is a defect in the program, not in the hardware.
- Identify the two races that appear in almost every concurrent program:
  check-then-act and read-modify-write.
- Explain the difference between a data race and a logical race.
- Explain why a mutex does not automatically fix a race.
- Explain what the C11 memory model says about undefined behaviour, and what
  that means practically.
- Use `atomic_int` with the correct memory ordering to fix a race.
- Use ThreadSanitizer to find races, and read its output.
- Explain the interaction between races and compiler/CPU reordering.

---

# Why Do We Need This Note?

## The Problem

A race condition is the reason concurrency bugs are hard. Not because they are
subtle in isolation, but because they are **invisible in single-threaded
execution and non-deterministic in multi-threaded execution**.

```text
  A race means: the result of a program depends on the TIMING of
  interleaved operations. Not on the order the programmer intended.
  Not on anything written in the source.

  The same binary, the same input, produces different output
  depending on how the OS scheduled the threads that day.
```

---

# Real World Analogy

Two people updating a shared whiteboard tally of how many boxes have been
loaded onto a truck.

```text
  Truck loader A: reads the board → "17"
  Truck loader B: reads the board → "17"
  Loader A: writes "18"
  Loader B: writes "18"

  Two boxes were loaded. The board says 18. It said 17 before they
  started. One box is on the truck and nobody can account for it.

  Nobody lied. Nobody made a mistake they were aware of. Each wrote
  down what they had correctly read. The RESULT is wrong.
```

---

# Intuition

```text
  THE TWO RACE SHAPES YOU WILL MEET
  ──────────────────────────────────

  1. CHECK-THEN-ACT
     ────────────────
        if (x == expected) {      ← the check
            do_something();       ← the act
        }
     Two threads both pass the check, both act. Classic in
     check-then-set, double-checked locking, and "is the cache
     populated yet?".

  2. READ-MODIFY-WRITE
     ──────────────────
        t = counter;              ← the read
        t = t + 1;                ← the modify
        counter = t;              ← the write
     Two threads read the same value, both increment, both write the
     same result. One increment is lost.

  BOTH ARE "NON-ATOMIC OPERATIONS".
  The atomic RMW exists precisely to collapse read+modify+write into
  one indivisible step. See test-and-set.md.
```

---

# Formal Definition

> A **race condition** occurs when the behaviour of a program depends on the
> relative timing or interleaving of operations from two or more threads, and
> that dependence is not intended by the programmer.

> A **data race** (C11 §5.1.2.4) occurs when two threads access the same
> memory location, at least one of the accesses is a **write**, and the
> accesses are **not ordered by happens-before**. The behaviour is **undefined
> behaviour**, not merely "racy".

```text
  RACE CONDITION  ⊃  DATA RACE

    A race can exist with NO data race at all:
        both threads use proper atomics, but the ORDER of two
        logically-related atomic operations is wrong.
        e.g. two threads do atomic_fetch_add(&count, 1) and
        atomic_store(&flag, 1) — both individually safe, but the
        program is wrong if it assumed flag was set before
        some other thread observed count. That is a LOGICAL race.

    A data race always involves a missing happens-before edge.
```

> [!IMPORTANT]
> In C11, a data race is **undefined behaviour**. That is stronger than
> "the answer is unpredictable". The compiler is permitted to assume it does
> not happen, and to optimise accordingly. This is why some races only appear
> after you turn optimisation up.

---

# Example 1 — The Classic Lost Update

```c
#include <stdio.h>
#include <pthread.h>
#include <stdlib.h>

static int counter = 0;                    /* shared, unprotected */
static const int ITERS = 1000000;

static void *bump(void *arg)
{
    (void)arg;
    for (int i = 0; i < ITERS; i++) {
        int tmp = counter;                 /* READ   */
        tmp = tmp + 1;                     /* MODIFY */
        counter = tmp;                     /* WRITE  */
    }
    return NULL;
}

int main(void)
{
    pthread_t t[2];
    for (int i = 0; i < 2; i++) pthread_create(&t[i], NULL, bump, NULL);
    for (int i = 0; i < 2; i++) pthread_join(t[i], NULL);

    printf("counter = %d (expected %d)\n", counter, 2 * ITERS);
    return counter != 2 * ITERS;          /* usually returns nonzero */
}
```

```bash
gcc -O0 -pthread -o race1 race1.c
for i in 1 2 3 4 5; do ./race1; done
```

```text
counter = 1337214 (expected 2000000)
counter = 1998873 (expected 2000000)
counter = 1586440 (expected 2000000)
counter = 2000000 (expected 2000000)   ← one run got lucky
```

```text
  THE INTERLEAVING THAT LOSES AN UPDATE

    T0: read  counter → tmp = 500000
    T1: read  counter → tmp = 500000
    T0: write counter = 500001
    T1: write counter = 500001      ← T1's update is GONE

  Note the window is only ~1 nanosecond wide (two instructions), yet
  with a million iterations and two threads it is hit reliably.
```

## The same code with a mutex

```c
static pthread_mutex_t m = PTHREAD_MUTEX_INITIALIZER;
static int counter = 0;

static void *bump(void *arg)
{
    (void)arg;
    for (int i = 0; i < ITERS; i++) {
        pthread_mutex_lock(&m);
        counter++;
        pthread_mutex_unlock(&m);
    }
    return NULL;
}
```

```bash
gcc -O2 -pthread -o race_fixed race_fixed.c && ./race_fixed
```

```text
counter = 2000000 (expected 2000000)
```

## The same code with an atomic

```c
#include <stdatomic.h>
static atomic_int counter = 0;

static void *bump(void *arg)
{
    (void)arg;
    for (int i = 0; i < ITERS; i++)
        atomic_fetch_add_explicit(&counter, 1, memory_order_relaxed);
    return NULL;
}
```

```text
  2,000,000 relaxed atomic increments: ~5 ns each, scales with cores,
  no lock and no syscall. For a single counter this is the right answer.
```

---

# Example 2 — Check-Then-Act

```c
#include <stdio.h>
#include <pthread.h>
#include <stdbool.h>
#include <unistd.h>

static int compute_expensive_thing(void) { return 42; }

static bool cache_ready = false;
static int  cache_value  = 0;
static pthread_mutex_t init_lock = PTHREAD_MUTEX_INITIALIZER;

static void *loader(void *arg)
{
    (void)arg;
    usleep(1000);
    if (!cache_ready) {              /* CHECK: is it already loaded? */
        usleep(5000);                /* pretend this is a slow load   */
        cache_value = compute_expensive_thing();
        cache_ready = true;          /* ACT: mark it ready            */
    }
    return NULL;
}

int main(void)
{
    pthread_t t[4];
    for (int i = 0; i < 4; i++) pthread_create(&t[i], NULL, loader, NULL);
    for (int i = 0; i < 4; i++) pthread_join(t[i], NULL);
    printf("value = %d\n", cache_value);
    return 0;
}
```

```text
  All four threads see cache_ready == false, all four do the expensive
  work, and all four write. Four times the cost, and if `compute` has
  any side effect (allocating, incrementing a refcount, opening a
  connection) the result is wrong, not just slow.
```

```c
/* The correct fix: pthread_once. Exactly one thread runs the body,
   ever. The others block until it is done. */
static pthread_once_t once = PTHREAD_ONCE_INIT;
static int value;

static void compute_once(void) { value = compute_expensive_thing(); }

static void *loader2(void *arg)
{
    (void)arg;
    pthread_once(&once, compute_once);   /* atomic check-and-act */
    return NULL;
}
```

---

# Example 3 — Double-Checked Locking

```c
/*
 * The famous pattern that is wrong in C, and subtly wrong in C++ before
 * the 2011 memory model. It is a textbook example of a logical race.
 */
static Config *config = NULL;
static pthread_mutex_t config_lock = PTHREAD_MUTEX_INITIALIZER;

static Config *get_config(void)
{
    if (config == NULL) {                  /* fast path: no lock, no syscall */
        pthread_mutex_lock(&config_lock);
        if (config == NULL) {              /* CHECK AGAIN — is this a race? */
            Config *c = malloc(sizeof *c);
            c->x = 1; c->y = 2;            /* the data must be visible... */
            pthread_mutex_unlock(&config_lock);
            config = c;                    /* ...BEFORE the pointer is stored */
        } else {
            pthread_mutex_unlock(&config_lock);
        }
    }
    return config;                         /* may be visible before c->x is */
}
```

```text
  THE LOGICAL RACE

    Thread A: takes the lock, builds c, stores config = c
    Thread B: reads config (non-NULL), returns c
    Thread B: reads c->x  →  GARBAGE, because A's store to c->x has
                          not been made visible yet.

  The mutex did its job perfectly. The bug is that the STORE ORDER is
  wrong: the pointer must become visible only AFTER the object it points
  to is fully initialised. That requires a release/acquire pair on the
  pointer.
```

```c
/* The C11 fix: make the pointer atomic, with release on the store and
   acquire on the load. */
static _Atomic(Config *) config = NULL;
static pthread_mutex_t config_lock = PTHREAD_MUTEX_INITIALIZER;

static Config *get_config(void)
{
    Config *c = atomic_load_explicit(&config, memory_order_acquire);
    if (c) return c;                       /* fast path, correctly synchronised */

    pthread_mutex_lock(&config_lock);
    c = atomic_load_explicit(&config, memory_order_acquire);
    if (!c) {
        Config *nc = malloc(sizeof *nc);
        nc->x = 1;
        nc->y = 2;
        /* release: everything written to nc is visible before the
           pointer itself becomes visible to any acquiring reader */
        atomic_store_explicit(&config, nc, memory_order_release);
        c = nc;
    }
    pthread_mutex_unlock(&config_lock);
    return c;
}
```

> [!IMPORTANT]
> Double-checked locking in C requires the pointer to be an **atomic with
> release/acquire semantics**, not just a plain pointer guarded by a mutex on
> the write path. This is the single most important practical reason to know
> C11 atomics.

---

# Why Races Are Hard to Test

```text
  A race is a function of timing. Timing is not controlled by your test.

  A race with a 1-nanosecond window, exercised a million times per run,
  across 2 threads, appears roughly:
      P(at least one collision) = 1 − (1 − 1e-9)^(iterations × threads)

  With 1e6 iterations × 2 threads and a ~1e-9 window, that is about a
  1-in-500 chance per run. Your test suite of 100 runs hits it roughly
  18% of the time. So:

      • it will not reproduce on demand
      • when it does reproduce, you cannot make it do so again
      • the fix cannot be validated by the test that found it
```

```bash
# The tools that work anyway, because they do not rely on timing.
#
# 1. ThreadSanitizer: instruments every access, reports any pair of
#    accesses from different threads that are not happens-before ordered.
#    It finds the race in the FIRST execution, deterministically.
gcc -fsanitize=thread -g -O1 -pthread -o race race1.c
./race
```

```text
WARNING: ThreadSanitizer: data race
  Write of size 4 at 0x7b0400004040 by thread T1:
    #0 bump race1.c:11 (bump+0x40)
  Previous read of size 4 at 0x7b0400004040 by thread T1:
    #0 bump race1.c:10 (bump+0x30)
  Location is global 'counter' of size 4 at race1.c:6

  Atomic read of size 4 at 0x7b0400004040 by thread T0:
    #0 bump race1.c:10 (bump+0x2c)

SUMMARY: ThreadSanitizer: data race race1.c:10 in bump
```

```text
  ThreadSanitizer does not wait for a race to occur. It RECORDS every
  memory access with a happens-before label, and reports a race when it
  sees two accesses from different threads whose labels are unordered.
  A race is therefore detected on the FIRST run, every run.
```

```bash
# 2. Helgrind: no recompilation needed
valgrind --tool=helgrind ./race
```

```text
==2480== Possible data race during read of size 4
==2480==    at 0x4005A1: bump (race1.c:10)
==2480==  Address 0x5D2050 is 0 bytes inside a block of size 4 free'd
```

```bash
# 3. Compiler warnings for the easy cases
gcc -Wall -Wextra -Wthread-safety -O2 -pthread -c race1.c
# clang -Wthread-safety  (checks @lockable annotations)

# 4. Static analysis
clang --analyze race1.c
cppcheck --enable=warning,style race1.c
```

---

# Reordering: Why `-O0` Is Not Enough

```text
  THE DATA RACE IS A LANGUAGE-LEVEL BUG, NOT A TIMING BUG.

  In C11, a data race is UNDEFINED BEHAVIOUR. That gives the compiler
  a licence. Given this:

      if (flag == 0) { count++; }

  the compiler may assume that `count++` can only execute when flag is 0,
  and hoist the check, cache `count` in a register across the whole
  loop, or eliminate the read entirely.

  The consequences:
    • the bug may DISAPPEAR at -O0 and REAPPEAR at -O2
    • it may not reproduce in a debug build
    • it may only reproduce on a weakly-ordered architecture
    • changing unrelated code may move it
```

```c
/*
 * An example of the compiler exploiting undefined behaviour.
 * The loop is not "racy in practice" — it is undefined, so the
 * compiler may delete the read of `shared` entirely.
 */
#include <stdio.h>
#include <pthread.h>

static int shared = 0;
static int flag   = 0;

static void *worker(void *arg)
{
    (void)arg;
    while (!flag) { }        /* spin until flag is set */
    if (shared == 0)         /* the compiler may assume this is true
                                 forever, because nobody may write it
                                 without a synchronisation edge */
        printf("shared is still 0\n");
    return NULL;
}
```

```bash
# The same source, two optimisation levels
gcc -O0 -pthread -o r0 race_ub.c && objdump -d r0 | grep -c "cmpl.*flag"
gcc -O2 -pthread -o r2 race_ub.c && objdump -d r2 | grep -c "cmpl.*flag"
# Often the -O2 build has removed the check entirely.
```

---

# The Happens-Before Relation

```text
  The C11 memory model defines a partial order, "happens before" (HB).
  Two accesses are ORDERED if HB connects them, and UNORDERED otherwise.

  HB is established by exactly four things:
    1.  program order within a single thread
    2.  a release store synchronises-with an acquire load that reads it
    3.  a mutex unlock synchronises-with a subsequent lock of it
    4.  a thread-creation / thread-join pair

  DATA RACE  =  two conflicting accesses, in DIFFERENT threads,
               with NO happens-before path between them.

  If a release/acquire pair sits between two accesses, they are ordered,
  and there is no race — regardless of whether the platform is x86 or ARM.
```

```c
/* Each pair below establishes a happens-before edge and is therefore
   race-free. Each comment names the edge. */
#include <stdatomic.h>
#include <pthread.h>

static atomic_int   flag;      /* 1 */
static int          plain;     /* 2 */
static pthread_mutex_t m = PTHREAD_MUTEX_INITIALIZER;

void writer(void)
{
    plain = 42;                                   /* (2) */
    atomic_store_explicit(&flag, 1, memory_order_release);  /* (1) release */
}

void reader(void)
{
    while (!atomic_load_explicit(&flag, memory_order_acquire))  /* (1) acquire */
        ;
    printf("%d\n", plain);   /* SAFE: seeing flag==1 implies seeing plain==42 */
}
```

```c
/* The memory orders, cheapest to strongest. */
#include <stdatomic.h>

memory_order_relaxed   /* no ordering at all. Use for a standalone counter.
                           "atomic" but the value is not synchronised. */
memory_order_acquire   /* no reads/writes AFTER this may move BEFORE it */
memory_order_release   /* no reads/writes BEFORE this may move AFTER it */
memory_order_acq_rel   /* both, for read-modify-write */
memory_order_seq_cst   /* everything, plus a single total order across all
                           threads. The default, and the strongest. */
```

| Operation | Cheapest correct ordering | When |
|---|---|---|
| `atomic_fetch_add` on a counter | `relaxed` | the count stands alone |
| `atomic_load` / `atomic_store` of a flag | `acquire` / `release` | publishing data |
| compare-and-swap | `acq_rel` | a lock, a flag with a value |
| Dekker's algorithm | `seq_cst` | needs the StoreLoad barrier |

```c
/* Relaxed is correct for a counter, and ONLY for a counter. */
static atomic_int hits = 0;
void record(void) { atomic_fetch_add_explicit(&hits, 1, memory_order_relaxed); }
```

---

# Races and the `volatile` Keyword

```c
volatile int flag = 0;

void writer(void) { flag = 1; }        /* NOT a synchronisation primitive */
```

```text
  WRONG. `volatile` tells the compiler: "this variable may change without
  me knowing, so do not cache it in a register."

  It does NOT tell the compiler:
    • that the read-modify-write is atomic        → lost updates remain
    • that writes are visible in order            → the store can be reordered
    • that the CPU's store buffer is drained       → visibility is still delayed

  The correct declarations:

      volatile sig_atomic_t ready;   /* signal handlers ONLY */
      _Atomic int            ready;  /* threads */
```

```bash
# Proof that volatile is not enough: ThreadSanitizer still reports it
cat > vol.c <<'EOF'
#include <pthread.h>
volatile int counter = 0;
void *bump(void *a) { (void)a; for (int i=0;i<1000000;i++) counter++; return 0; }
int main(void) { pthread_t t[2];
  pthread_create(&t[0],0,bump,0); pthread_create(&t[1],0,bump,0);
  pthread_join(t[0],0); pthread_join(t[1],0);
  return counter != 2000000; }
EOF
gcc -fsanitize=thread -g -O1 -pthread -o vol vol.c && ./vol 2>&1 | head -5
```

```text
WARNING: ThreadSanitizer: data race
  Write of size 4 at 0x... by thread T1:
    #0 bump vol.c:4
  ...
SUMMARY: ThreadSanitizer: data race vol.c:4 in bump
```

---

# The Complete Diagnostic Workflow

```text
  1. REPRODUCE UNDER TSAN, not under normal execution.

       gcc -fsanitize=thread -g -O1 -pthread -o prog prog.c
       ./prog

     TSAN reports the race in the first run, deterministically, with
     both stacks and the address of the variable.

  2. IDENTIFY THE ACCESS PAIR.
     Read the two stack traces. The shared variable is named in the
     "Location is ..." line. The two accesses are the race.

  3. FIND THE MISSING EDGE.
     Ask: what SHOULD order these two accesses? The answer tells you
     which primitive is missing:
       • protecting a shared counter        → atomic_fetch_add
       • initialising once                  → pthread_once
       • protecting a data structure        → mutex
       • publishing a pointer to new data   → release/acquire on the pointer
       • many readers                      → rwlock

  4. APPLY IT WITH THE CORRECT MEMORY ORDER.
     Start at seq_cst. Measure. Relax only if you can argue why it is
     safe, and re-run TSAN after each relaxation.

  5. RE-RUN TSAN.
     If the race is gone, the fix is real. TSAN has no false negatives
     for races it understands.

  6. ADD A REGRESSION TEST.
     The test cannot reproduce the race by timing, so the test must run
     under TSAN in CI.
```

```bash
# The full loop
gcc -fsanitize=thread -g -O1 -pthread -o prog prog.c
./prog 2>&1 | tee tsan.log
# fix
gcc -fsanitize=thread -g -O1 -pthread -o prog prog.c
./prog   # clean
gcc -Wall -Wextra -fsanitize=thread,undefined -O1 -pthread -o prog prog.c
```

```bash
# CI configuration
CC=gcc
CFLAGS="-Wall -Wextra -Werror -fsanitize=thread -O1 -g -pthread"
# run the test suite under it on every push
```

```bash
# Also worth having
valgrind --tool=helgrind ./prog       # lock-order and race detection, no rebuild
perf record -g ./prog && perf report  # where the time actually goes
```

---

# Common Misconceptions

### ❌ "`volatile` fixes the race."

It does not, and this is the most widespread threading error in C. `volatile`
prevents the compiler from caching a value in a register. It says nothing
about atomicity, ordering, or visibility across cores. ThreadSanitizer reports
`volatile` races exactly as it reports ordinary ones.

---

### ❌ "A mutex on the variable is enough."

A mutex protects a variable only if **every** access to it uses the same mutex,
under the same discipline. One unlocked read in a logging function, or one
`printf("%d", counter)` outside the lock, and the mutex protects nothing. The
discipline has to be written down and enforced by review and by ThreadSanitizer.

---

### ❌ "The race only happens under high load, so it is not a real bug."

A race is not a load-dependent condition. It is undefined behaviour that
happens to manifest under load. The same code will corrupt memory in
production on a quieter day with a different access pattern.

---

### ❌ "ThreadSanitizer reports false positives."

It is extremely unlikely to report a false positive on an actual data race.
The common causes of a *misdiagnosis* are: (a) the race is real but in a
library that TSAN does not intercept; (b) the code is a **logical** race with
all-atomic operations, which TSAN does not report; (c) the fix needs a
release/acquire pair, not a mutex, and the mutex is not where TSAN looks.

---

### ❌ "If it works with 2 threads, it works with 8."

Races do not scale linearly in their likelihood. More threads mean more
possible interleavings, a wider window in which each thread can be preempted,
and more cache-line ping-pong between cores. A race that appears once in
10,000 runs with 2 threads can appear once in 10 with 8.

---

# Interview Questions

### Basic

- What is a race condition? What is a data race?
- Give an example of a check-then-act race and a read-modify-write race.
- Why is a single counter updated with `volatile` still racy?
- How do you fix a lost-update race?

### Intermediate

- What is double-checked locking, and what is wrong with the naive C version?
- Explain the C11 happens-before relation. What establishes it?
- What is the difference between `memory_order_relaxed` and `memory_order_acq_rel`?
- How does ThreadSanitizer find a race in the first run when it takes a million
  iterations to occur?

### Advanced

- A data race is undefined behaviour in C11. Explain how that can cause a bug
  that appears only at `-O2`.
- You have a logical race with no data race. How would you find it, given that
  ThreadSanitizer reports nothing?
- Design a happens-before argument proving that a particular lock-free queue
  is correct.
- `memory_order_seq_cst` is easy to reason about but costs more. Describe a
  program where you can prove `acquire`/`release` suffices, and how you would
  verify the relaxation.

---

# University Exam Notes

### Definitions

- **Race condition:** A condition in which the result of a program depends on
  the unpredictable interleaving of operations from concurrent threads.
- **Data race:** Two conflicting accesses to the same memory location, at
  least one a write, in different threads, with no happens-before ordering
  between them. Undefined behaviour in C11.
- **Check-then-act race:** A race in which a condition is tested and an action
  is performed as separate steps, so two threads can both pass the test.
- **Read-modify-write race:** A race in which a value is read, modified, and
  written back as separate steps, so an update can be lost.
- **Happens-before:** The C11 partial order on memory accesses, established by
  program order, release/acquire pairs, mutex operations, and thread
  creation/join.
- **Lost update:** The specific failure where a read-modify-write sequence
  discards a concurrent update.
- **Double-checked locking:** A lazy-initialisation pattern that checks a
  pointer before and after taking a lock; correct only with a release/acquire
  pair on the pointer.

### Frequently Asked Questions

- What is a race condition? Explain with an example.
- Differentiate between a race condition and a data race.
- What is the lost-update problem and how is it fixed?
- Explain double-checked locking and its requirements in C11.
- What is the happens-before relation? What establishes it?
- Why is `volatile` not a synchronisation primitive?
- How does ThreadSanitizer detect races?

---

# Key Takeaways

- A race means the result depends on **timing**, not on the source code. It is
  a defect in the program.
- The two shapes to recognise: **check-then-act** and **read-modify-write**.
  Both are non-atomic operations.
- A **data race** is undefined behaviour in C11. A **logical race** has no data
  race but is still wrong — all accesses atomic, the order wrong.
- A mutex only protects data if **every** access uses it under the same
  discipline. One unlocked read voids the whole scheme.
- Double-checked locking needs the pointer to be **atomic with
  release/acquire**, not a plain pointer guarded by a mutex.
- **`volatile` is not synchronisation.** Use `_Atomic`. `volatile
  sig_atomic_t` is only for signal handlers.
- Races get *worse* with more threads, and can appear only at `-O2` because
  undefined behaviour lets the compiler assume the race cannot happen.
- **ThreadSanitizer finds races in the first run**, because it tracks
  happens-before labels rather than waiting for a collision. Use it in CI.
- Start with `memory_order_seq_cst`, then relax only with an argument, and
  re-run TSAN after each relaxation.

---

# References

- Operating System Concepts — Silberschatz, §6.1 ( races), §7.1
- Modern Operating Systems — Tanenbaum, §2.7
- C11 standard, §5.1.2.4 (multithreaded execution and data races),
  §7.17 (atomic operations)
- Herlihy & Shavit, *The Art of Multiprocessor Programming*, ch. 5
- Boehm, "Threads Cannot Be Implemented as a Library", PLDI 2005
- `gcc -fsanitize=thread`, `valgrind --tool=helgrind`, `clang -Wthread-safety`
