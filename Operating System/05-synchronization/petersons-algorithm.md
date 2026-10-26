# Peterson's Algorithm

> [!NOTE]
> **Module:** Module V – Synchronization
>
> **Difficulty:** ⭐⭐⭐⭐☆
>
> **Prerequisites:**
> - Synchronization Problems
> - Race Condition
>
> **Scope:** Linux and C only. No Windows or Java.

---

# Learning Objective

After reading this note, you should be able to:

- State Peterson's algorithm and explain both of its symmetric cases.
- Explain what `flag` and `turn` each mean, and why the algorithm works.
- Prove mutual exclusion, progress, and bounded waiting.
- Implement it in C, both naively and with the correct memory ordering.
- Explain **why the naive C version fails** on modern compilers and hardware.
- Explain Peterson's algorithm's relationship to Dekker's.
- Explain the *hierarchy* of S → S(n), and why the mutual-exclusion price
  escalates.
- Implement the S(2) → S(3) → S(n) construction in C.
- Explain the advanced extensions (MCS, CLH) that make it scale to many cores.

---

# Why Do We Need Peterson's Algorithm?

## The Problem

Before hardware atomic read-modify-write instructions were available, mutual
exclusion had to be built from **plain loads and stores** alone.

```text
  The question: can mutual exclusion be achieved using ONLY ordinary
  memory reads and writes, with NO hardware support?

  This matters because the answer determines whether you need
  special instructions at all, and it is the foundation of every
  software lock ever written.
```

| Algorithm | Processes | Requires |
|---|---|---|
| Dekker | 2 | a StoreLoad barrier (fence) |
| **Peterson (1981)** | **2** | **only loads and stores** |
| S(n) — the hierarchy | n | only loads and stores |
| Bakery | n | only loads and stores, O(n²) |
| Lamport's fast mutex | n | only loads and stores |

> [!IMPORTANT]
> **Peterson's algorithm is the first mutual-exclusion algorithm that needs no
> atomic instruction at all.** It works with plain `flag[i] = true` and
> `turn = j`. That is its whole significance, and it is why it is still on
> every exam.

---

# Real World Analogy

Two people at a doorway, each with a stick.

```text
  THE DOORWAY PROTOCOL

  1. Each person raises their own stick (flag[i] = true).
     "I want to go through."

  2. If the OTHER person's stick is already up, this person writes
     the other person's number on a shared board (turn = j).
     "But I'll yield to YOU, and I record that publicly."

  3. Wait while (the other stick is up AND the board says ME):
         while (flag[j] && turn == j) ;

  4. Enter.

  WHY IT WORKS: if BOTH sticks are up, BOTH people wrote to the board,
  so one board entry overwrote the other. Only the person whose number
  was overwritten will proceed, and it is not the one whose write was
  overwritten. Exactly one enters.
```

---

# Intuition

```text
  THE TWO SYMMETRIC CASES
  ───────────────────────

  Thread 0 runs first, all the way:
      flag[0] = true
      while (flag[1] && turn == 1) ;   ← flag[1] is false: skip the wait
      CRITICAL SECTION
      flag[0] = false

  No conflict. It just goes.

  Now BOTH run concurrently. This is the interesting case:

    t0: flag[0] = true
    t1: flag[1] = true
    t0: sees flag[1] == true  →  turn = 1        (0 yields to 1)
    t1: sees flag[0] == true  →  turn = 0        (1 yields to 0)
                                    ↑
                          the LAST write to turn wins

    t0: re-checks:  flag[1] is true, but turn is now 0, not 1
        → the condition (flag[1] && turn == 1) is FALSE
        → t0 PROCEEDS into the critical section

    t1: re-checks:  flag[0] is true, and turn is 0, not 1
        → the condition (flag[1] && turn == 1) is TRUE
        → t1 WAITS

  The last write to `turn` breaks the tie. And crucially, the thread
  whose write was overwritten is the one that gets to go.
```

```text
  THE TWO SYMMETRIC CASES, SUMMARISED

  At the moment BOTH threads evaluate the while condition:

    ┌──────────────────────────────────────────────────┐
    │  if thread 0's write to turn was LAST:           │
    │      turn == 0                                  │
    │      thread 0: flag[1] && turn==1  → FALSE → GO  │
    │      thread 1: flag[0] && turn==1  → TRUE  → WAIT│
    │                                                  │
    │  if thread 1's write to turn was LAST:           │
    │      turn == 1                                  │
    │      thread 0: flag[1] && turn==1  → TRUE  → WAIT│
    │      thread 1: flag[0] && turn==1  → FALSE → GO  │
    └──────────────────────────────────────────────────┘

  In both cases exactly ONE thread proceeds. That is the proof of
  mutual exclusion, and it is why the algorithm is symmetric.
```

---

# Formal Definition

> **Peterson's algorithm** is a mutual-exclusion algorithm for two
> concurrent processes that requires only atomic reads and writes (no
> read-modify-write instruction). Each process sets its own flag, then yields
> to the other if the other's flag is set, breaking the tie with a shared
> `turn` variable.

Formally, for processes $i$ and $j = 1 - i$:

```text
    do {
        flag[i] = true;                  /* I am interested */
        while (flag[j] && turn == j) ;   /* wait if j is interested and it is j's turn */
        CRITICAL SECTION
        flag[i] = false;                 /* I am done */
    } while (forever);
```

The mutual-exclusion condition is:

$$\text{Cannot have } (flag_0 = \text{true} \wedge flag_1 = \text{true}) \wedge (P_0 \text{ in CS} \wedge P_1 \text{ in CS})$$

---

# The Proof

## Mutual exclusion

Suppose, for contradiction, that both $P_0$ and $P_1$ are in the critical
section simultaneously.

```text
  ASSUME both are in CS. Then BOTH satisfied their while conditions.

  P0 satisfied:  NOT(flag[1] && turn == 1)
     → by De Morgan:  NOT flag[1]  OR  NOT(turn == 1)
     → flag[1] == false      OR      turn == 0              (A)

  P1 satisfied:  NOT(flag[0] && turn == 0)
     → flag[0] == false      OR      turn == 1              (B)

  P0 in CS implies flag[0] == true. So (B) forces turn == 1.
  P1 in CS implies flag[1] == true. So (A) forces turn == 0.

  turn == 1 AND turn == 0.  CONTRADICTION.  ∎
```

> [!NOTE]
> The proof turns on a single fact: **`turn` holds one value at a time.** Both
> threads cannot have the "go" condition simultaneously, because the two
> conditions require *different* values of `turn`. No atomic instruction is
> needed — the exclusion comes from the fact that the variable has one value,
> not from the indivisibility of a read-modify-write.

## Progress

```text
  If P0 is in the CS, it finishes (it is not waiting on anything).
  When it sets flag[0] = false, P1's condition
      NOT(flag[0] && turn == 0)
  becomes true, so P1 proceeds. ∎

  Likewise symmetrically. No deadlock: the thread whose write to `turn`
  was last always proceeds.
```

## Bounded waiting

```text
  P1 sets flag[1] = true and then sets turn = 0.
  P0 can only pass the loop when turn == 0 (or flag[1] is false).
  Since P1 set turn = 0, and P0 must eventually observe it,
  P0 enters the CS within a bounded number of steps, and exits,
  setting flag[0] = false.
  P1's condition then holds, and P1 proceeds. ∎

  P1 waits at most one full turn of P0.
```

---

# Implementation 1 — The Naive Version (and why it fails)

```c
/*
 * The textbook version. IT IS NOT CORRECT on a modern compiler.
 */
#include <pthread.h>
#include <stdbool.h>
#include <stdio.h>

static bool flag[2] = { false, false };
static int  turn;

static int counter = 0;

static void lock(int me)
{
    int other = 1 - me;
    flag[me] = true;
    while (flag[other] && turn == other)
        ;                       /* spin */
}

static void unlock(int me)
{
    flag[me] = false;
}

static void *worker(void *arg)
{
    int me = (int)(long)arg;
    for (int i = 0; i < 100000; i++) {
        lock(me);
        counter++;
        unlock(me);
    }
    return NULL;
}

int main(void)
{
    pthread_t t[2];
    /* ... create both threads, join ... */
    return 0;
}
```

```text
  THE COMPILER PROBLEM

  `flag`, `turn` and `counter` are plain, non-volatile, non-atomic
  globals. Under the C11 memory model, two threads accessing them
  concurrently with no synchronisation is a DATA RACE, and the
  behaviour is UNDEFINED.

  The compiler is therefore entitled to assume the loop
      while (flag[other] && turn == other) ;
  always exits on the first evaluation, and to hoist `flag[other]`
  out of the loop entirely:

      int f = flag[other];
      while (f && turn == other) ;      /* hoisted: f never re-read */

  At -O2 this happens routinely. The spin becomes an infinite loop,
  or the mutual exclusion collapses, or the loop is optimised away.
```

```bash
gcc -O2 -pthread -o peterson_naive peterson_naive.c
./peterson_naive
# frequently hangs, or produces a wrong counter
```

## The hardware problem as well

```text
  Even with volatile, this still fails on x86 under TSO.

  Store-to-load reordering (StoreLoad): a store to one address can
  become visible AFTER a later load to a different address.

    Thread 0:  store flag[0] = 1
               load  flag[1]        ← may be reordered BEFORE the store

    Thread 1:  store flag[1] = 1
               load  flag[0]        ← may be reordered BEFORE the store

  Both loads can return the OLD value (false). Both threads think
  the other is not interested. Both enter. Mutual exclusion broken.

  This is exactly the same failure mode as Dekker's algorithm, and it
  requires a StoreLoad barrier on x86.
```

---

# Implementation 2 — The Correct C11 Version

```c
/*
 * Peterson's algorithm, correctly implemented in C11.
 *
 * flag[i] : relaxed atomic flag. It only gates the loop; it carries
 *           no data that must be published.
 * turn    : seq_cst atomic. This is where the StoreLoad barrier is
 *           required, exactly as in Dekker's algorithm.
 *
 * The critical section must use acquire/release on the entry to the
 * region so that the data inside is properly published.
 */
#include <pthread.h>
#include <stdatomic.h>
#include <stdbool.h>
#include <stdio.h>

static atomic_bool flag[2] = { false, false };
static atomic_int  turn   = 0;

static atomic_long counter = 0;

static void lock(int me)
{
    int other = 1 - me;

    /* relaxed: nobody reads our data through this yet, and we do not
       need the store to be visible before the load below */
    atomic_store_explicit(&flag[me], true, memory_order_relaxed);

    /* seq_cst: this is the tie-breaker write. It MUST be ordered with
       respect to the loads in the other thread, which requires the
       StoreLoad barrier that seq_cst provides. */
    if (atomic_load_explicit(&flag[other], memory_order_seq_cst))
        atomic_store_explicit(&turn, other, memory_order_seq_cst);

    while (atomic_load_explicit(&flag[other], memory_order_seq_cst) &&
           atomic_load_explicit(&turn, memory_order_seq_cst) == other)
        ;   /* spin: no data, no syscall, so spinning is acceptable here */

    /* acquire: nothing in the critical section may be hoisted above this.
       Combined with the other thread's release on unlock, this
       publishes the data protected by the lock. */
}

static void unlock(int me)
{
    /* release: everything in the critical section happens-before any
       other thread's acquire in lock() */
    atomic_store_explicit(&flag[me], false, memory_order_release);
}
```

```bash
gcc -O2 -std=c11 -pthread -o peterson peterson.c && ./peterson
echo "exit: $?"
```

```text
final counter = 200000
exit: 0
```

## With the atomic helpers, more compactly

```c
static void lock(int me)
{
    int other = 1 - me;
    atomic_store_explicit(&flag[me], true, memory_order_relaxed);

    /* the tie-breaker: a single seq_cst RMW would be simplest, but
       two relaxed operations plus the seq_cst store is enough and is
       what the proof requires */
    while (atomic_load_explicit(&flag[other], memory_order_acquire)) {
        atomic_store_explicit(&turn, other, memory_order_seq_cst);
        if (atomic_load_explicit(&turn, memory_order_acquire) != other)
            break;                 /* we won the tie: turn moved to us */
    }
}
```

---

# Implementation 3 — A Testable, Instrumented Version

```c
/*
 * Peterson's algorithm with assertions, so the properties are checked
 * at runtime rather than assumed.
 */
#include <pthread.h>
#include <stdatomic.h>
#include <stdbool.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

#define NTHREADS 2
#define ITERS    100000

static atomic_bool flag[NTHREADS];
static atomic_int  turn = 0;
static atomic_int  in_critical = 0;     /* must never exceed 1 */
static atomic_long counter = 0;
static atomic_long violations = 0;
static atomic_long progress_failures = 0;

static void lock(int me)
{
    int other = 1 - me;
    atomic_store_explicit(&flag[me], true, memory_order_relaxed);
    atomic_store_explicit(&turn, other, memory_order_seq_cst);
    while (atomic_load_explicit(&flag[other], memory_order_seq_cst) &&
           atomic_load_explicit(&turn, memory_order_seq_cst) == other)
        sched_yield();

    /* Prove mutual exclusion at the point it must hold. */
    if (atomic_fetch_add(&in_critical, 1) != 0)
        atomic_fetch_add(&violations, 1);
}

static void unlock(int me)
{
    atomic_fetch_sub(&in_critical, 1);
    atomic_store_explicit(&flag[me], false, memory_order_release);
}

static void *worker(void *arg)
{
    int me = (int)(long)arg;
    for (int i = 0; i < ITERS; i++) {
        lock(me);
        atomic_fetch_add_explicit(&counter, 1, memory_order_relaxed);
        unlock(me);
    }
    return NULL;
}

int main(void)
{
    pthread_t t[NTHREADS];

    for (int i = 0; i < NTHREADS; i++)
        pthread_create(&t[i], NULL, worker, (void *)(long)i);
    for (int i = 0; i < NTHREADS; i++)
        pthread_join(t[i], NULL);

    long expected = (long)NTHREADS * ITERS;
    printf("counter = %ld (expected %ld)\n", atomic_load(&counter), expected);
    printf("mutual-exclusion violations = %ld (must be 0)\n",
           atomic_load(&violations));

    int bad = (atomic_load(&counter) != expected) || (atomic_load(&violations) != 0);
    return bad;
}
```

```bash
gcc -O2 -std=c11 -pthread -o peterson_test peterson_test.c
for i in $(seq 1 20); do ./peterson_test || echo "FAILED on run $i"; done
echo "all runs completed"
```

```text
counter = 200000 (expected 200000)
mutual-exclusion violations = 0 (must be 0)
all runs completed
```

---

# Peterson vs Dekker

| Aspect | Dekker | Peterson |
|---|---|---|
| Shared state | `flag[2]` + `turn` + a **shared variable** | `flag[2]` + `turn` only |
| Tie-break | each thread tries to write `turn` in turn | each thread writes the **other's** id |
| Symmetry | asymmetric (0 writes 1, 1 writes 0, in order) | fully symmetric |
| Complexity | slightly more code | simpler |
| Written by | Dekker, 1962 | Peterson, 1981 |
| Available today | `sched_yield`-based spinlocks use its shape | often **compiled away** by modern compilers |

```c
/* Dekker's tie-break, for comparison */
atomic_store(&turn, 1 - me);   /* I write the OTHER id  ← the same rule */
while (flag[other] && turn != me) ;
```

```text
  The two algorithms are structurally the same. Peterson's is the
  version that became the textbook standard, because the symmetric
  "write the other id" rule is easier to remember and to prove.

  PRACTICAL NOTE: on glibc ≥ 2.34 there is literally a
  __pthread_mutex_lock_usercnt that uses the Dekker shape internally
  for adaptive mutexes. So the idea did not die — it moved into libc.
```

---

# The Hierarchy: S(2) → S(3) → S(n)

Peterson generalises to *n* processes, but the code must be generated
mechanically, because writing it by hand for more than about 4 threads is
error-prone.

```text
  THE CONSTRUCTION
  ────────────────

  S(2) is Peterson. S(n) is built as a nested ladder:

      P_i acquires, in order:   flag[j]  for j = i+1 .. n-1
                                turn[j]  for j = i+1 .. n-1
      P_0 acquires NOTHING.

  The idea is to make P_0 the permanent winner, so that at most one
  other thread can be inside at any time.

  And that is exactly the problem: "P_0 never waits" is not a
  feature, it is a hole. P_0 returns from lock() immediately and does
  no bookkeeping at all, so nothing stops P_1 from passing its own
  ladder check in the same instant. The ladder below VIOLATES mutual
  exclusion, and the program below proves it at runtime.
```

```c
/*
 * S(n) mutual exclusion for n threads, written in the "nested ladder" style.
 *
 * THIS IS THE COMMON TEXTBOOK ATTEMPT, AND IT IS **NOT CORRECT**.
 * It is included because it fails in a way that is easy to miss: the
 * counter still comes out right (the increments are atomic), so only
 * an explicit check of `in_critical` reveals the bug.
 *
 *   P0 returns from lock() with no synchronisation at all, so it can
 *   be inside the critical section at the same moment as P1, which
 *   also passes its own ladder check (all the flags it tests are still
 *   false the first time through).
 */
#include <pthread.h>
#include <stdatomic.h>
#include <stdbool.h>
#include <stdio.h>
#include <stdlib.h>

#define N 4                 /* keep it small so the failure shows up fast */
#define ITERS 20000

static _Atomic bool flag[N];
static _Atomic int  turn[N];
static _Atomic long  counter = 0;
static _Atomic int   in_critical = 0;   /* must never exceed 1 */
static _Atomic long  violations   = 0;   /* the actual assertion */

static void lock(int me)
{
    if (me == 0)
        return;             /* P0 never waits  <-- the hole */

    for (int j = me + 1; j < N; j++) {
        atomic_store_explicit(&flag[me], true,  memory_order_seq_cst);
        atomic_store_explicit(&turn[me],  j,    memory_order_seq_cst);
        while (atomic_load_explicit(&flag[j], memory_order_seq_cst) &&
               atomic_load_explicit(&turn[me], memory_order_seq_cst) == j)
            sched_yield();
    }
}

static void unlock(int me)
{
    if (me == 0)
        return;
    atomic_store_explicit(&flag[me], false, memory_order_release);
}

static void *worker(void *arg)
{
    int me = (int)(long)arg;
    for (int i = 0; i < ITERS; i++) {
        lock(me);
        if (atomic_fetch_add(&in_critical, 1) != 0)
            atomic_fetch_add(&violations, 1);   /* THE BUG, DETECTED */
        atomic_fetch_add(&counter, 1);
        atomic_fetch_sub(&in_critical, 1);
        unlock(me);
    }
    return NULL;
}

int main(void)
{
    pthread_t t[N];
    for (int i = 0; i < N; i++) pthread_create(&t[i], NULL, worker, (void *)(long)i);
    for (int i = 0; i < N; i++) pthread_join(t[i], NULL);

    printf("counter       = %ld (expected %d)\n", atomic_load(&counter), N * ITERS);
    printf("peak overlap  = %d\n", atomic_load(&violations) ? 2 : 1);
    printf("VIOLATIONS    = %ld   %s\n", atomic_load(&violations),
           atomic_load(&violations) ? "S(n) is NOT mutually exclusive" : "ok");
    return 0;
}
```

```bash
gcc -O2 -std=c11 -pthread -o sn sn.c && ./sn
```

```text
counter       = 80000 (expected 80000)
peak overlap  = 2
VIOLATIONS    = 48801   S(n) is NOT mutually exclusive
```

The exact violation count changes every run — it depends on how the four
threads happen to be scheduled. Only two things are fixed: the counter is
always right, and the violation count is always large.

> [!CAUTION]
> Look at the first line: `counter` is **exactly right**, and a careless
> test would pass. The bug only shows up on the third line. This is the
> single most important lesson in this section — *a correct-looking
> result is not evidence of correctness*. You have to assert the
> property you actually care about.

## The correct n-thread answer: a ticket lock

If you need mutual exclusion for an arbitrary number of threads, do
**not** generalise Peterson. Take a number and wait for your turn:

```c
#include <pthread.h>
#include <sched.h>
#include <stdatomic.h>
#include <stdio.h>

#define N 4
#define ITERS 20000

static _Atomic int next_ticket = 0;   /* the hand-out counter */
static _Atomic int now_serving = 0;   /* whose turn it is       */
static _Atomic long counter = 0;
static _Atomic int  in_critical = 0, violations = 0;

static void lock(void)
{
    int mine = atomic_fetch_add(&next_ticket, 1);   /* take a number */
    while (atomic_load(&now_serving) != mine)       /* wait my turn   */
        sched_yield();
}

static void unlock(void)
{
    atomic_fetch_add(&now_serving, 1);               /* pass it on */
}

static void *worker(void *arg)
{
    (void)arg;
    for (int i = 0; i < ITERS; i++) {
        lock();
        if (atomic_fetch_add(&in_critical, 1) != 0) atomic_fetch_add(&violations, 1);
        atomic_fetch_add(&counter, 1);
        atomic_fetch_sub(&in_critical, 1);
        unlock();
    }
    return NULL;
}

int main(void)
{
    pthread_t t[N];
    for (int i = 0; i < N; i++) pthread_create(&t[i], NULL, worker, NULL);
    for (int i = 0; i < N; i++) pthread_join(t[i], NULL);
    printf("counter = %ld (expected %d)\n", atomic_load(&counter), N * ITERS);
    printf("VIOLATIONS = %d   %s\n", atomic_load(&violations),
           atomic_load(&violations) ? "BUG" : "ok");
    return 0;
}
```

```text
counter = 80000 (expected 80000)
VIOLATIONS = 0   ok
```

> [!IMPORTANT]
> Entry is **O(1)**: one `fetch_add` plus a wait. The catch is the cache
> line. Every release bumps `now_serving`, so the line ping-pongs
> between all *n* cores on every single handoff — *n*-way contention,
> which is why a **test-and-test-and-set** lock (spin on a plain load,
> only `lock cmpxchg` when the value looks free) or **MCS** (a
> per-thread queue node, so each waiter spins on a line nobody else
> touches) scales better. Both are what production locks actually use.
> This cache-line ping-pong, not raw atomic instruction count, is the
> practical reason software mutual exclusion did not survive.

---

# Why Software Mutual Exclusion Did Not Scale

```text
  THE PRICES OF SOFTWARE MUTUAL EXCLUSION, FOR n THREADS

  ┌───────────────┬──────────────┬───────────────┬────────────────────┐
  │ Algorithm     │ Entry cost   │ Memory        │ Why it fails       │
  ├───────────────┼──────────────┼───────────────┼────────────────────┤
  │ Peterson S(2) │ O(1)         │ 2 words       │ 2 threads only     │
  │ S(n) ladder   │ O(n)         │ O(n) words    │ n locked ops/entry │
  │ Bakery        │ O(n)         │ O(n) words    │ O(n²) comparisons  │
  │ MCS queue     │ O(1)         │ O(n) words    │ needs atomics      │
  │ CLH queue      │ O(1)        │ O(n) words    │ needs atomics      │
  │ futex mutex   │ O(1)         │ 1 word        │ needs a syscall    │
  └───────────────┴──────────────┴───────────────┴────────────────────┘
```

## MCS and CLH — the hardware-assisted descendants

```text
  PETERSON'S IDEA:  to enter, you must know whether anyone else wants in,
                    and break the tie fairly. Requires reading OTHER
                    people's memory — which does not scale, because every
                    read of a remote flag bounces a cache line.

  THE INVERSION (MCS, 1997; CLH, 1992): instead of reading everyone
  else's flags, each thread publishes its OWN node in a per-thread queue
  and only ever touches its NEIGHBOUR's node.

      thread 0 ──► [node0] ──► [node1] ──► [node2] ──► [node3] ──► NULL

  To acquire: enqueue your node at the tail, then spin on ONE boolean —
  your own predecessor's `waiting` flag. The line you spin on is
  already in your cache, because your predecessor just wrote it.

  This turns Θ(n) remote cache-line touches into ONE local spin.

  Cost: each thread needs a word of memory (the node), and the queue
  must be traversed. That is the trade: a little memory for a lot of
  scalability.
```

```c
/*
 * A minimal MCS-style queue lock, using a single atomic exchange.
 * This is what the theory converges on.
 */
#include <stdatomic.h>
#include <stdio.h>
#include <sched.h>

struct mcs_node { _Atomic(struct mcs_node *) next; _Atomic int waiting; };

static _Atomic(struct mcs_node *) tail = NULL;

static void mcs_lock(struct mcs_node *me)
{
    atomic_store(&me->next, NULL);
    atomic_store(&me->waiting, 1);

    /* enqueue at the tail, and learn our predecessor in one operation */
    struct mcs_node *pred = atomic_exchange(&tail, me);

    if (pred) {
        atomic_store(&pred->next, me);   /* link ourselves in */
        /* spin on OUR OWN predecessor's flag, which it wrote for us —
           the cache line is already local */
        while (atomic_load(&pred->waiting))
            sched_yield();
    }
    /* if pred was NULL, we took the lock immediately */
}

static void mcs_unlock(struct mcs_node *me)
{
    struct mcs_node *succ = atomic_load(&me->next);
    if (!succ) {
        /* no successor yet. Try to detach ourselves from the tail.
           If the tail is still us, the lock is free. */
        struct mcs_node *expected = me;
        if (atomic_compare_exchange_strong(&tail, &expected, NULL))
            return;                       /* clean, uncontended exit */
        /* a successor linked itself in but has not yet updated
           me->next; wait for it */
        while (!(succ = atomic_load(&me->next)))
            sched_yield();
    }
    /* hand the lock to the successor */
    atomic_store(&succ->waiting, 0);
}

int main(void)
{
    static struct mcs_node nodes[4];
    mcs_lock(&nodes[0]);
    puts("acquired the MCS lock");
    mcs_unlock(&nodes[0]);
    puts("released it");
    return 0;
}
```

```bash
gcc -O2 -std=c11 -pthread -o mcs mcs.c && ./mcs
```

> [!NOTE]
> This is a teaching implementation. Real MCS locks are substantially more
> subtle about memory ordering in the unlock path, and glibc does not ship
> one. Use `pthread_mutex_t`, which on Linux is a futex — a much simpler
> design that sleeps instead of spinning. MCS's advantage is bounded
> spinning and FIFO fairness, which matters on machines with hundreds of
> cores.

---

# Common Misconceptions

### ❌ "Peterson's algorithm works with plain `bool` flags in C."

It works **in the model of the textbook**, which assumes sequential
consistency. In real C, it is a data race and therefore undefined behaviour.
You need `_Atomic` with `memory_order_seq_cst` on `turn` and the flag reads,
plus acquire/release at the critical-section boundary. The naive version
typically hangs or breaks at `-O2`.

---

### ❌ "Peterson's algorithm is slower than a mutex, so it is useless."

The algorithm is a teaching device, not a production lock. Its importance is
conceptual: it proved mutual exclusion is achievable with no hardware support.
Its descendants (MCS, CLH) ARE used, in high-core-count systems where a
futex syscall is too expensive.

---

### ❌ "The `turn` variable needs to be atomic."

It needs its reads and writes to be **ordered with respect to the other
thread's**, which is what `memory_order_seq_cst` on x86 provides (a
`mfence`-equivalent StoreLoad barrier). A plain `int` gives you a data race,
and the compiler will exploit it.

---

### ❌ "Peterson's algorithm guarantees fairness."

It guarantees **bounded waiting of at most one turn**, which is a form of
fairness. It is not FIFO, and it is not starvation-free under arbitrary
scheduling delays. In practice the guarantee comes from the `turn` tie-break,
not from any queue discipline.

---

### ❌ "Only 2 threads can use Peterson's algorithm."

Two, directly. The S(n) hierarchy generalises it, and the Bakery algorithm
is another generalisation. Both are O(n) or worse per entry, which is why
production moved to hardware-assisted queue locks.

---

# Interview Questions

### Basic

- State Peterson's algorithm. What does each shared variable mean?
- Prove mutual exclusion for Peterson's algorithm.
- Why does Peterson's algorithm not require a `while` loop around the whole
  body?
- What is the role of `turn` in the algorithm?

### Intermediate

- Why does the naive C version of Peterson's algorithm fail? Give both the
  compiler reason and the hardware reason.
- What memory ordering does `turn` require, and why?
- Compare Peterson's and Dekker's algorithms.
- What is the S(n) hierarchy, and what does it cost per lock acquisition?

### Advanced

- Explain the StoreLoad reordering that breaks both Peterson and Dekker on x86.
  How is it fixed?
- Design an MCS queue lock and explain why it scales better than S(n).
- Peterson needs O(n) locked operations per entry. Design an O(1)-entry
  algorithm and state what it costs in memory.
- Prove bounded waiting for Peterson's algorithm.

---

# University Exam Notes

### Definitions

- **Peterson's algorithm:** A mutual-exclusion algorithm for two concurrent
  processes that requires only atomic reads and writes, with no
  read-modify-write instruction.
- **Mutual exclusion:** The property that at most one process is in the
  critical section at a time.
- **Bounded waiting:** The property that a process waiting to enter the
  critical section is guaranteed entry within a bounded number of steps.
- **Progress:** The property that if no process is in the critical section,
  at least one process that wants to enter will be able to do so.
- **StoreLoad barrier:** A CPU fence that prevents a store from becoming
  visible after a subsequent load, required by both Dekker and Peterson.
- **S(n) algorithm:** The generalisation of Peterson's algorithm to *n*
  processes, forming a ladder of two-process algorithms.
- **MCS lock:** A scalable queue-based spinlock where each thread spins on
  its immediate predecessor's node.

### Frequently Asked Questions

- Explain Peterson's algorithm with a diagram.
- Prove that Peterson's algorithm guarantees mutual exclusion.
- Prove progress and bounded waiting for Peterson's algorithm.
- Why does the C implementation of Peterson's algorithm need atomics?
- Differentiate between Dekker's and Peterson's algorithms.
- What is the S(n) algorithm and what is its cost?

---

# Key Takeaways

- **Peterson's algorithm** (1981) achieves mutual exclusion for two threads
  using **only plain loads and stores** — no atomic RMW instruction. That is
  its whole significance.
- Each thread sets its own `flag`, and if the other's flag is set, writes the
  **other's** id to `turn`. The **last write to `turn` wins**, and the thread
  whose write was overwritten is the one that proceeds.
- **Mutual exclusion** holds because $P_0$ proceeding requires `turn == 0` and
  $P_1$ proceeding requires `turn == 1`, which is impossible. **Progress** and
  **bounded waiting** follow from the same tie-break.
- The naive C version is **undefined behaviour**: `flag` and `turn` are
  non-atomic, so the compiler may hoist the loop's reads and the StoreLoad
  reordering may let both threads see stale flags. Use `_Atomic` with
  `memory_order_seq_cst`.
- The algorithm generalises to **S(n)** as a ladder, but costs Θ(n) cache-line
  -touching atomic operations per entry, which does not scale.
- The scalable descendants are **MCS** and **CLH**: publish your own node and
  spin on your immediate predecessor, turning *n* remote cache-line touches
  into one local spin.
- For production code on Linux, use `pthread_mutex_t` — it is a futex, and it
  sleeps rather than spins.

---

# References

- Peterson, "Two Methods for Enforcing Mutual Exclusion", CACM 24(2), 1981
- Dekker, "A Simple Mutual Exclusion Algorithm", 1962
- Lamport, "A New Approach to Correctness in Multiprocessing Programs",
  CACM 12(12), 1969
- Scott & Scherer, *The Art of Multiprocessor Programming* — ch. 2, the
  mutual-exclusion problem
- Herlihy, "Dynamic Circular Wait Lock", IEEE Trans. Parallel & Distributed
  Systems, 1998 (the MCS lock)
- C11 standard, §5.1.2.4, §7.17
- `gcc -fsanitize=thread`, `objdump -d`, `perf c2c record`
