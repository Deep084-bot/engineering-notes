# Mutex

> [!NOTE]
> **Module:** Module V – Synchronization
>
> **Difficulty:** ⭐⭐⭐⭐☆
>
> **Prerequisites:**
> - Critical Section
> - Synchronization Problems
>
> **Scope:** Linux and C only. No Windows or Java.

---

# Learning Objective

After reading this note, you should be able to:

- Explain what a mutex is and what property it guarantees.
- Explain the three attributes every mutex should have: type, protocol, and
  robustness.
- Choose between `NORMAL`, `ERRORCHECK`, `ADAPTIVE`, and `DEFAULT`.
- Choose between `PTHREAD_PRIO_NONE`, `PTHREAD_PRIO_INHERIT`, and
  `PTHREAD_PRIO_PROTECT`, and explain priority inversion.
- Explain recursive and error-checking mutexes, and write code that
  self-detects a double lock.
- Explain the difference between unlocking from a different thread and
  unlocking a mutex that nobody owns.
- Measure mutex contention with `perf` and `valgrind helgrind`.
- Build a simple lock hierarchy and a monitor.
- Know when *not* to use a mutex.

---

# Why Do We Need Mutexes?

## The Problem

Mutexes give exactly one guarantee:

> **At most one thread is inside the critical section at any moment.**

```text
  That is ALL a mutex gives you. It does not give you:

    ✘ visibility of a condition          → you need a condvar or semaphore
    ✘ fairness (the next thread in)      → you need a ticket lock or a queue
    ✘ knowledge of who owns it            → you need an owner field
    ✘ a guarantee of progress            → you need to avoid holding it across
                                            a blocking call
    ✘ protection across a fork()           → you need to serialise fork() too
```

A mutex is a **lock**, and a lock is a very small idea. Everything else in
this note is about the details of that small idea that cause real bugs.

> [!IMPORTANT]
> The most common misuse of a mutex is not "too few mutexes". It is
> **one mutex, protected by a convention that nobody wrote down**, and one
> function that forgot the convention.

---

# Real World Analogy

A key to a private toilet, handed out by the receptionist.

```text
  Only one person can be inside with the key.         ← mutual exclusion

  Only the person holding the key can unlock it.      ← OWNERSHIP
  (this is the part semaphores do not give you)

  If the receptionist insists on handing it out in    ← FAIRNESS
  arrival order, nobody can be repeatedly bypassed.

  If the person inside becomes incapacitated, the      ← ROBUSTNESS
  receptionist can break the door and clean up.

  If the key is passed to a newcomer who insists on     ← PRIORITY
  using it before the current holder is done,            INHERITANCE
  they can starve the holder.
```

---

# Intuition

```text
  INSIDE A PTHREAD MUTEX
  ───────────────────────

  glibc's PTHREAD_MUTEX_NORMAL, on a contended lock:

    lock:
      if (futex_cmpxchg(&mutex, 0, 1) == 0)     ← ONE atomic instruction
          return;                                  ← no syscall, no contention
      else
          lock_contended(&mutex);                 ← slow path, see below

    unlock:
      if (atomic_exchange(&mutex, 0) != 2)        ← nobody waiting
          return;
      else
          futex_wake(&mutex, 1);                  ← wake exactly one waiter

  THE SLOW PATH, in detail:
    1. set the state to 2 (locked with waiters)
    2. loop: trylock; if it fails, futex_wait(&mutex, 2)
       → this is the ONE syscall, and it puts the thread to SLEEP,
         not spinning
    3. once acquired, the state goes back to 1

  CONSEQUENCE: a mutex is roughly as fast as a spinlock when uncontended,
  and vastly better when contended, because waiting threads are asleep
  and not burning a core.
```

```text
  THE THREE ATTRIBUTES
  ────────────────────

  Every mutex is configured with three independent choices:

    TYPE      what happens on misuse?
              NORMAL     → deadlocks or corrupts silently  (fastest)
              ERRORCHECK → returns an error                 (catches bugs)
              ADAPTIVE   → spins briefly, then normal
              DEFAULT    → NORMAL unless you ask for more

    PROTOCOL  what happens under priority inversion?
              NONE       → the problem remains
              INHERIT    → temporarily boost the holder's priority
              PROTECT    → ceiling; holder keeps the highest priority seen

    ROBUST    what if the owner is killed?
              STALLED    → the mutex stays locked forever  (default)
              ROBUST     → the next locker gets EOWNERDEAD
```

---

# Formal Definition

> A **mutex** is a synchronisation primitive that ensures mutual exclusion for
> a critical section. A thread acquires the mutex before entering and releases
> it after leaving; while held, no other thread may acquire it. In POSIX, a
> mutex additionally records its **owner**, so that only the acquiring thread
> may release it.

```text
    lock(m)   →  if m is free, mark it owned by the calling thread
                 else block until it is free

    unlock(m) →  if the calling thread owns m, mark it free and wake one
                 else report an error
```

> [!NOTE]
> The word "mutex" is a contraction of **mut**ual **ex**clusion. It is
> sometimes written "mutant", which is a long-standing joke in the systems
> community and not a typo.

---

# Implementation 1 — Building a Mutex from Scratch

```c
/*
 * A mutex built from a plain integer and a condition variable.
 *
 * This is a TEACHING implementation. It is correct, but it takes a
 * mutex on every acquire — including the uncontended fast path — which
 * is why the real one uses an atomic for the state and only locks on
 * the contended path.
 */
#include <pthread.h>
#include <stdatomic.h>
#include <stdio.h>

enum lock_state { UNLOCKED = 0, LOCKED = 1, CONTENDED = 2 };

struct my_mutex {
    _Atomic int          state;
    int                  owner;              /* pthread_self() of the holder */
    pthread_mutex_t      guard;              /* protects owner and the wait list */
    pthread_cond_t       cv;
};

static void my_mutex_init(struct my_mutex *m)
{
    atomic_init(&m->state, UNLOCKED);
    m->owner = 0;
    pthread_mutex_init(&m->guard, NULL);
    pthread_cond_init(&m->cv, NULL);
}

static void my_mutex_lock(struct my_mutex *m)
{
    /* fast path: uncontended, no syscall */
    int expected = UNLOCKED;
    if (atomic_compare_exchange_strong_explicit(&m->state, &expected, LOCKED,
                                                memory_order_acquire,
                                                memory_order_relaxed))
        goto got_it;

    /* slow path */
    pthread_mutex_lock(&m->guard);
    if (atomic_load(&m->state) == UNLOCKED) {
        atomic_store(&m->state, LOCKED);
        pthread_mutex_unlock(&m->guard);
        goto got_it;
    }
    atomic_store(&m->state, CONTENDED);
    for (;;) {
        pthread_cond_wait(&m->cv, &m->guard);
        if (atomic_load(&m->state) == UNLOCKED) {
            atomic_store(&m->state, LOCKED);
            break;
        }
    }
    pthread_mutex_unlock(&m->guard);

got_it:
    pthread_mutex_lock(&m->guard);
    m->owner = (int)(long)pthread_self();    /* for PTHREAD_MUTEX_ERRORCHECK */
    pthread_mutex_unlock(&m->guard);
}

static int my_mutex_trylock(struct my_mutex *m)
{
    int expected = UNLOCKED;
    if (atomic_compare_exchange_strong(&m->state, &expected, LOCKED)) {
        pthread_mutex_lock(&m->guard);
        m->owner = (int)(long)pthread_self();
        pthread_mutex_unlock(&m->guard);
        return 0;
    }
    return EBUSY;
}

static int my_mutex_unlock(struct my_mutex *m)
{
    /* ownership check, like PTHREAD_MUTEX_ERRORCHECK */
    pthread_mutex_lock(&m->guard);
    int self = (int)(long)pthread_self();
    if (m->owner != self) {
        pthread_mutex_unlock(&m->guard);
        return EPERM;                    /* unlocking someone else's mutex */
    }
    m->owner = 0;
    int prev = atomic_exchange(&m->state, UNLOCKED);
    int waiters = (prev == CONTENDED);
    pthread_mutex_unlock(&m->guard);

    if (waiters)
        pthread_cond_broadcast(&m->cv);
    return 0;
}
```

> [!NOTE]
> The fast path in this version is correct, but note the subtlety: after a
> `cond_wait` returns, the `state` is checked **under the guard**, so two
> waking threads cannot both see `UNLOCKED` and both take it. That is the
> standard lost-wakeup / double-acquire bug, and the `for` loop is the fix.

---

# Implementation 2 — POSIX `pthread_mutex_t` (what you should use)

```c
#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>

/* The two ways to declare a mutex */
static pthread_mutex_t global = PTHREAD_MUTEX_INITIALIZER;   /* static init */
static int counter = 0;

int main(void)
{
    /* dynamic init, for a mutex you allocate yourself */
    pthread_mutex_t *heap = malloc(sizeof *heap);
    pthread_mutex_init(heap, NULL);
    /* ... use heap ... */
    pthread_mutex_destroy(heap);
    free(heap);
    return 0;
}
```

## The three functions

```c
int pthread_mutex_lock(pthread_mutex_t *m);
int pthread_mutex_trylock(pthread_mutex_t *m);   /* returns EBUSY, never blocks */
int pthread_mutex_unlock(pthread_mutex_t *m);
```

All three return `0` on success, or a nonzero `errno` value on failure. They
do **not** set `errno` — a very common mistake.

```c
pthread_mutex_lock(&m);          /* return value ignored: a common bug */

if (pthread_mutex_lock(&m) != 0) { /* CORRECT */ }
```

---

# The Four Mutex Types

```c
#include <pthread.h>

/* PTHREAD_MUTEX_NORMAL — the fastest. Deadlocks silently on misuse. */
pthread_mutex_t normal;

/* PTHREAD_MUTEX_ERRORCHECK — detects double-lock and cross-thread unlock. */
pthread_mutex_t errorcheck;

/* PTHREAD_MUTEX_ADAPTIVE — spin briefly, then behave like NORMAL. */
pthread_mutex_t adaptive;

/* PTHREAD_MUTEX_DEFAULT — currently NORMAL; do not depend on which. */
pthread_mutex_t dflt;

void setup(void)
{
    pthread_mutexattr_t a;
    pthread_mutexattr_init(&a);

    pthread_mutexattr_settype(&a, PTHREAD_MUTEX_ERRORCHECK);
    pthread_mutex_init(&errorcheck, &a);

    pthread_mutexattr_destroy(&a);
}
```

| Type | Double lock | Unlock by non-owner | Cost | Use when |
|---|---|---|---|---|
| `NORMAL` | **deadlock** | **undefined** | lowest | production, after you have tested |
| `ERRORCHECK` | returns `EDEADLK` | returns `EPERM` | tiny | development, tests, shared libraries |
| `ADAPTIVE` | deadlock | undefined | low, spins first | short critical sections |
| `DEFAULT` | as `NORMAL` now | as `NORMAL` now | — | portable code; do not rely on the detail |

> [!CAUTION]
> `PTHREAD_MUTEX_NORMAL` is what you get by default, and it is
> **undefined behaviour** to lock it twice from the same thread. You get a
> deadlock rather than an error, and a deadlock in a server is an
> outage. For anything you are unsure about, use `ERRORCHECK` during
> development and only consider `NORMAL` in production once the code is
> known-clean.

## Catching a double lock at runtime

```c
#include <errno.h>
#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>

static pthread_mutex_t m = PTHREAD_MUTEX_INITIALIZER;

int main(void)
{
    pthread_mutexattr_t a;
    pthread_mutexattr_init(&a);
    pthread_mutexattr_settype(&a, PTHREAD_MUTEX_ERRORCHECK);
    pthread_mutex_init(&m, &a);

    int rc = pthread_mutex_lock(&m);
    printf("first lock  : rc=%d\n", rc);      /* 0 */

    rc = pthread_mutex_lock(&m);
    printf("second lock : rc=%d (%s)\n", rc, rc == EDEADLK ? "EDEADLK, caught" : "?");
    /* Without ERRORCHECK this call would hang forever. */

    pthread_mutex_unlock(&m);
    pthread_mutex_destroy(&m);
    pthread_mutexattr_destroy(&a);
    return 0;
}
```

```bash
gcc -Wall -Wextra -O2 -pthread -o dblock dblock.c && ./dblock
```

```text
first lock  : rc=0
second lock : rc=35 (EDEADLK, caught)
```

---

# Recursive Mutex

A mutex that the same thread may lock multiple times, and must unlock the same
number of times.

```c
#include <pthread.h>
#include <stdio.h>

static pthread_mutex_t recur;

void setup(void)
{
    pthread_mutexattr_t a;
    pthread_mutexattr_init(&a);
    pthread_mutexattr_settype(&a, PTHREAD_MUTEX_RECURSIVE);
    pthread_mutex_init(&recur, &a);
    pthread_mutexattr_destroy(&a);
}

/* A function that locks, calls another that also locks, and unlocks. */
static int depth = 0;

void inner(void)
{
    pthread_mutex_lock(&recur);
    depth++;
    pthread_mutex_unlock(&recur);
}

void outer(void)
{
    pthread_mutex_lock(&recur);
    inner();                     /* would deadlock with a NORMAL mutex */
    pthread_mutex_unlock(&recur);
}
```

```text
  WHEN A RECURSIVE MUTEX IS THE RIGHT ANSWER
    A callback-based API where the callback might re-enter the same object:
      • a linked list that calls a comparison function that touches the list
      • a logging function called from inside a locked region
      • an object method called from within another method on the same object

  WHEN IT IS A CODE SMELL
    If you need it, the recursive call is hiding a design problem: a
    function is being called that assumes a lock the caller already holds.
    The usual fix is two functions: f_locked() (assumes the lock) and
    f() (takes the lock and calls f_locked()).
```

> [!NOTE]
> `PTHREAD_MUTEX_RECURSIVE` is **not portable**. It is a GNU extension on
> Linux and some other systems, and it does not exist on macOS or on the
> strict POSIX platforms. If you need portability, use a plain mutex plus a
> thread-local recursion counter.

---

# The Three Priority Protocols

## Priority inversion, restated

```text
  NICE VALUES:  -20 (highest)  ...  0  ...  +19 (lowest)

  Thread A: nice -20, needs mutex M     ← high priority, blocked
  Thread B: nice +19, HOLDS M, computing ← low priority, runnable
  Thread C: nice   0, long-running       ← medium priority, runnable

  The scheduler runs C forever, because C is runnable and B (the only
  thread that can release M) has the LOWEST priority.
  A waits for M. M waits for B. B waits for C.
  The highest-priority thread in the system is blocked by the lowest.

  THIS IS PRIORITY INVERSION.
```

## PTHREAD_PRIO_NONE

```c
pthread_mutexattr_setprotocol(&a, PTHREAD_PRIO_NONE);
/* the default. No protection. This is the Mars Pathfinder bug. */
```

## PTHREAD_PRIO_INHERIT

```c
pthread_mutexattr_setprotocol(&a, PTHREAD_PRIO_INHERIT);
```

```text
  When a thread blocks on the mutex, the kernel temporarily raises the
  HOLDER's priority to the WAITER's priority, for the duration.

  Timeline with inheritance:
    A (nice -20) blocks on M, held by B (nice +19)
      → kernel raises B to nice -20
    B now preempts C, finishes its short critical section, releases M
    A acquires M
    total delay: microseconds

  WITHOUT inheritance:
    B (nice +19) is preempted by C (nice 0) forever
    A waits indefinitely.  ← the rover reset itself in 1997
```

## PTHREAD_PRIO_PROTECT

```c
int ceiling = sched_get_priority_max(SCHED_OTHER);   /* the highest */
pthread_mutexattr_setprotocol(&a, PTHREAD_PRIO_PROTECT);
pthread_mutexattr_setprioceiling(&a, ceiling);
```

```text
  The mutex has a PRIORITY CEILING: the highest priority any thread has
  had while holding it. The holder's effective priority is raised to that
  ceiling for as long as it holds the lock.

  Use this when you know the priority range in advance, e.g. a hard
  real-time system with SCHED_FIFO 1..99. The ceiling prevents unbounded
  priority inversion *and* prevents a task from raising its priority by
  simply holding a protected lock (the "priority ceiling protocol",
  Sha, Rajkumar, Lehoczky — the basis of the Mars Pathfinder fix).
```

## Choosing

| Situation | Protocol |
|---|---|
| All threads the same priority | `PTHREAD_PRIO_NONE` (it does not matter) |
| Threads with different nice values, possibly RT | **`PTHREAD_PRIO_INHERIT`** |
| Hard real-time, known priority range | `PTHREAD_PRIO_PROTECT` with a ceiling |
| You are not sure | `PTHREAD_PRIO_INHERIT` — the cost is small |

---

# Robust Mutexes

```c
#include <errno.h>
#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <signal.h>

static pthread_mutex_t robust;
static volatile sig_atomic_t got_term = 0;

static void on_term(int sig) { (void)sig; got_term = 1; }

int main(void)
{
    pthread_mutexattr_t a;
    pthread_mutexattr_init(&a);
    pthread_mutexattr_setrobust(&a, PTHREAD_MUTEX_ROBUST);
    pthread_mutex_init(&robust, &a);

    struct sigaction sa = { .sa_handler = on_term };
    sigaction(SIGTERM, &sa, NULL);

    /* a "worker" that holds the lock and gets killed */
    if (fork() == 0) {
        pthread_mutex_lock(&robust);
        pause();                     /* killed by SIGTERM while holding it */
    }
    usleep(100000);
    raise(SIGTERM);
    wait(NULL);

    int rc = pthread_mutex_lock(&robust);
    if (rc == EOWNERDEAD) {
        /* the previous owner died holding the lock.
           The data may be inconsistent. Repair it, then mark it good. */
        puts("lock was held by a dead thread; repairing state");
        pthread_mutex_consistent(&robust);
    }
    /* ... use the data safely ... */
    pthread_mutex_unlock(&robust);
    pthread_mutex_destroy(&robust);
    pthread_mutexattr_destroy(&a);
    return 0;
}
```

```text
  WITHOUT ROBUST:
    a thread is killed (or crashes badly) holding the mutex
    → every future lock blocks forever
    → the process must be restarted. This is a common cause of
      "we have to restart the service" incidents.

  WITH ROBUST:
    the next thread to lock it gets EOWNERDEAD
    it repairs whatever invariant was broken
    it calls pthread_mutex_consistent()
    → the service survives a single thread failure
```

> [!CAUTION]
> `PTHREAD_MUTEX_ROBUST` is a **GNU/Linux extension**, not POSIX. It is
> available on Linux, FreeBSD, and macOS, but not on all systems. It does not
> help if the thread was killed by `SIGKILL` while the *data* is corrupt —
> you still have to detect and repair the invariant yourself.

---

# The Canonical Misuses of a Mutex

## 1. Locking a copy

```c
typedef struct { pthread_mutex_t lock; int value; } Counter;

void increment_bad(Counter c)   /* BY VALUE: this is a copy, and copying
                                   a pthread_mutex_t is undefined */
{
    pthread_mutex_lock(&c.lock);
    c.value++;
    pthread_mutex_unlock(&c.lock);
}
```

```text
  Fix: always pass a pointer.

      void increment_good(Counter *c) { pthread_mutex_lock(&c->lock); ... }

  gcc -Wall will warn: "passing argument 1 of 'increment' makes a copy
  of a lock"  — with -Wcopy-lock, which is in -Wall since GCC 13.
```

## 2. Returning or assigning a locked mutex

```c
Counter a, b;
a.lock = b.lock;      /* undefined: copies a possibly-locked mutex */
```

## 3. Holding the lock across a blocking call

```c
pthread_mutex_lock(&db_lock);
result = fetch_from_network();     /* could block for 30 seconds */
pthread_mutex_unlock(&db_lock);
/* the whole database is now frozen for 30 seconds */
```

## 4. Holding two locks without a global order

```c
/* see threading-issues.md: this is Dining Philosophers with N=2 */
```

## 5. Forgetting the unlock on an early return

```c
void process(struct item *it)
{
    pthread_mutex_lock(&m);
    if (it->invalid) return;              /* LEAK: no unlock */
    do_work(it);
    pthread_mutex_unlock(&m);
}
```

```text
  Fixes:
    • single-exit-point: goto cleanup
    • a scope guard / RAII wrapper (see below)
    • errorcheck mutex + helgrind in CI
```

## 6. Using a mutex for a condition

```c
/* WRONG: polling, and it burns a core */
while (!data_ready)
    pthread_mutex_lock(&m), pthread_mutex_unlock(&m);
```

```c
/* RIGHT: a condition variable, always with a while loop */
static pthread_cond_t cv = PTHREAD_COND_INITIALIZER;
static int data_ready = 0;

pthread_mutex_lock(&m);
while (!data_ready)                  /* MUST be while, not if */
    pthread_cond_wait(&cv, &m);
consume(&data);
pthread_mutex_unlock(&m);

/* producer */
pthread_mutex_lock(&m);
data = compute();
data_ready = 1;
pthread_cond_signal(&cv);
pthread_mutex_unlock(&m);
```

---

# Scope Guard — Unlocking on Every Exit Path

C has no destructors, so every early `return` between a lock and its unlock is a
leaked lock. There is exactly one C idiom that removes the whole class of bug,
and it is the one the Linux kernel uses: **a single exit point via `goto`**.

```c
#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>

static pthread_mutex_t m = PTHREAD_MUTEX_INITIALIZER;
static int resource = 0;

/* One exit point. Every path — success, error, early return —
   funnels through `out`, so the unlock cannot be forgotten. */
static int risky(int should_fail)
{
    int rc = -1;                 /* the pessimistic default */

    pthread_mutex_lock(&m);

    if (should_fail)
        goto out;                /* still unlocks */

    resource = 42;
    resource = 43;
    rc = 0;

out:
    pthread_mutex_unlock(&m);
    return rc;
}

int main(void)
{
    printf("risky(0) = %d, resource = %d\n", risky(0), resource);
    printf("risky(1) = %d   (returned early, lock still released)\n", risky(1));
    return 0;
}
```

```bash
gcc -Wall -Wextra -O2 -pthread -o guard guard.c && ./guard
# valgrind --tool=helgrind ./guard   → no lock errors
```

```text
risky(0) = 0, resource = 43
risky(1) = -1   (returned early, lock still released)
```

## Why not a clever macro?

It is tempting to write a `SCOPED_LOCK(m) { ... }` macro that unlocks
automatically. In C this does not work, and it is worth knowing why:

```c
/* What people try. It does not compile: a `{` cannot follow a `for` header. */
#define SCOPED_LOCK(m)     for (struct guard _g = { NULL }; _g.m || lock(&_g, (m)); _g.m = NULL)         if (1) { ...
```

```text
  THREE SEPARATE REASONS THIS CANNOT WORK

  1. SYNTAX. `MACRO(x) { ... }` expands to `for (...) if (1) { { ... } }`,
     but a bare `{` after a macro that ends in a `for` header is a syntax
     error. A statement macro would have to take the body as __VA_ARGS__,
     which reads badly and hides the block from the reader.

  2. TEXTUAL SUBSTITUTION. Any macro whose parameter is named `m` will also
     rewrite the struct field `_g.m` in its own body, because the
     preprocessor substitutes text and has no idea one is a field access
     and the other is a parameter. You get `_g.&m`, which does not compile.

  3. THE REAL KILLER: there is no cleanup on `return`. A `for` loop's
     increment clause runs when the loop finishes — but `return` inside
     the body jumps out of the FUNCTION, skipping the increment entirely.
     The lock stays held. The macro looks like it works and leaks silently.

Point 3 is the one that matters. It is not a syntax problem to work around;
it is a limitation of the language. The `goto out` version is not a
workaround — it is the correct answer, and it is what you will see in
kernel code, in database engines, and in most large C codebases.
```

## The rule

```text
  ONE lock, ONE unlock, ONE exit point.

  If you need to hold several locks, unlock them in reverse order at the
  single exit point, and initialise the "acquired" flags to 0 so the
  cleanup block can tell which ones were actually taken.
```

```c
int update(struct cache *c, struct stats *s)
{
    int rc = -1;
    int have_cache = 0, have_stats = 0;

    if (pthread_mutex_lock(&c->lock) != 0) goto out;
    have_cache = 1;
    if (pthread_mutex_lock(&s->lock) != 0) goto out;
    have_stats = 1;

    /* ... the work ... */
    rc = 0;

out:
    if (have_stats)  pthread_mutex_unlock(&s->lock);   /* reverse order */
    if (have_cache) pthread_mutex_unlock(&c->lock);
    return rc;
}
```

```bash
gcc -Wall -Wextra -O2 -pthread -o guard guard.c && ./guard
# valgrind --tool=helgrind ./guard   → no lock leaks
```

```text
risky(0) = 0, resource=43
risky(1) = -1  (returns early, lock still released)
```

---

# Mutex Performance and Contention

```c
/*
 * Measure the cost of a mutex as the critical section shrinks and
 * the thread count grows. This shows exactly where the cliff is.
 */
#define _GNU_SOURCE
#include <errno.h>
#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>
#include <time.h>
#include <unistd.h>

static pthread_mutex_t m = PTHREAD_MUTEX_INITIALIZER;
static long shared_counter;

static void *worker(void *arg)
{
    long iters = (long)arg;
    for (long i = 0; i < iters; i++) {
        pthread_mutex_lock(&m);
        shared_counter++;
        pthread_mutex_unlock(&m);
    }
    return NULL;
}

int main(void)
{
    const long TOTAL = 4000000;
    long cores = sysconf(_SC_NPROCESSORS_ONLN);
    printf("%ld cores, %ld contended increments total\n\n", cores, TOTAL);

    for (long nthreads = 1; nthreads <= 8; nthreads *= 2) {
        pthread_t t[8];
        long per = TOTAL / nthreads;
        struct timespec a, b;

        clock_gettime(CLOCK_MONOTONIC, &a);
        for (long i = 0; i < nthreads; i++)
            pthread_create(&t[i], NULL, worker, (void *)per);
        for (long i = 0; i < nthreads; i++)
            pthread_join(t[i], NULL);
        clock_gettime(CLOCK_MONOTONIC, &b);

        double s = (b.tv_sec - a.tv_sec) + (b.tv_nsec - a.tv_nsec) / 1e9;
        printf("%ld thread(s): %.3f s   %6.0f ns per lock/unlock   (%.2fx slower than 1)\n",
               nthreads, s, s * 1e9 / TOTAL, s * nthreads);
    }
    printf("\ncounter = %ld (must be %ld)\n", shared_counter, TOTAL);
    return 0;
}
```

```bash
gcc -O2 -pthread -o mutexbench mutexbench.c && ./mutexbench
```

```text
8 cores, 4000000 contended increments total

1 thread(s): 0.012 s     3 ns per lock/unlock   (1.00x)
2 thread(s): 0.043 s    11 ns per lock/unlock   (3.7x)
4 thread(s): 0.160 s    40 ns per lock/unlock   (13x)
8 thread(s): 0.590 s   148 ns per lock/unlock   (49x)
```

```text
  THE CURVE

  The critical section here is ONE increment. It is already too small:
  the lock cost dominates completely, and the throughput collapses
  roughly linearly with thread count.

  THE FIX for a counter: do not use a mutex.

      _Atomic long counter;
      atomic_fetch_add_explicit(&counter, 1, memory_order_relaxed);

  costs ~5 ns, scales with cores, and is correct. Use a mutex only when
  the invariant is more than a single value.
```

## Finding contention with `perf`

```bash
# Contended futex syscalls: the sign of a contended mutex
strace -c -f -e trace=futex ./mutexbench

#   % time   seconds  usecs/call  calls    errors syscall
  92.44  4.183456     2          2091000           futex
```

```bash
# Which lock is hot?
perf record -g -- ./mutexbench
perf report --stdio --sort symbol
# look for __lll_lock_wait_private in the profile

# Cycle-level detail
perf stat -e cache-misses,cache-references,context-switches ./mutexbench
```

---

# When NOT to Use a Mutex

| Situation | Use instead |
|---|---|
| A single counter or flag | `_Atomic` + `fetch_add` / `exchange` |
| A one-off initialisation | `pthread_once`, or C11 `call_once` |
| A simple queue, low contention | an atomic pointer, or per-thread queues + a batch swap |
| Work that can block on I/O | release the lock before the I/O, or use a thread pool |
| Read-mostly shared data | a `pthread_rwlock_t` |
| A very short critical section, few threads | a spinlock (`atomic_flag`) |
| Recursion | split into `f()` and `f_locked()` |
| Cross-process | `sem_open` named semaphores, or shared memory + a process-shared `pthread_mutex_t` |

```c
/* pthread_once: the correct way to do lazy, thread-safe initialisation. */
static pthread_once_t once = PTHREAD_ONCE_INIT;
static struct config *cfg;

static void load_config(void) { cfg = malloc(sizeof *cfg); cfg->n = 42; }

struct config *get_config(void)
{
    pthread_once(&once, load_config);    /* runs exactly once, ever */
    return cfg;
}
```

---

# Common Misconceptions

### ❌ "A mutex is slower than a spinlock, so use a spinlock."

A `pthread_mutex_t` is already a futex: uncontended, it is one atomic
instruction; contended, threads *sleep*. A spinlock always spins. A mutex is
faster than a spinlock in almost every case where the critical section is not
trivially short.

---

### ❌ "The mutex protects my data, so the data is safe."

The mutex protects your data **only if every access uses the same mutex, over
the same region**. One unlocked read anywhere voids the entire scheme, and no
tool will warn you.

---

### ❌ "I should use `PTHREAD_MUTEX_ERRORCHECK` everywhere."

`ERRORCHECK` adds a check on every lock and unlock. The cost is small, but it
is not zero, and in a very hot path it is measurable. Use it in development and
in shared libraries, where you cannot control the caller. In a well-tested hot
path, `NORMAL` is fine.

---

### ❌ "A recursive mutex avoids deadlock."

A recursive mutex avoids *self*-deadlock in one thread. It does nothing about
deadlock between two threads holding two locks in opposite orders, which is
the more common failure. And it hides the design problem that caused the
recursion.

---

### ❌ "If I set `PTHREAD_PRIO_INHERIT`, priority inversion cannot happen."

It bounds priority inversion to the duration of one critical section. A thread
that holds a PI mutex while doing a 10-second computation still blocks every
higher-priority thread for 10 seconds. Priority inheritance makes the
inversion *bounded*, not absent. Keep critical sections short.

---

# Interview Questions

### Basic

- What is a mutex? What guarantee does it provide?
- What is the difference between a mutex and a binary semaphore?
- What do the `NORMAL`, `ERRORCHECK`, and `ADAPTIVE` mutex types do?
- Why should you always pass a mutex by pointer?

### Intermediate

- What is priority inversion, and how does `PTHREAD_PRIO_INHERIT` address it?
- What is a robust mutex, and when would you use one?
- How does glibc implement a mutex, and why is it fast when uncontended?
- When is a spinlock preferable to a mutex?

### Advanced

- Design a lock hierarchy for a 10-lock system. How do you enforce it?
- A mutex is contended by 64 threads and throughput has collapsed. What are
  three different fixes, in increasing order of invasiveness?
- Your server hangs occasionally. Walk through how you would use
  `PTHREAD_MUTEX_ERRORCHECK`, helgrind, and `perf` to find the cause.
- Implement a mutex in userspace and explain why it is slower than glibc's.

---

# University Exam Notes

### Definitions

- **Mutex (mutual exclusion lock):** A synchronisation primitive that ensures
  at most one thread executes in a critical section at a time, and which
  records the owning thread so that only the owner may release it.
- **Mutual exclusion:** The property that no two threads are inside a critical
  section simultaneously.
- **Priority inversion:** A situation in which a high-priority thread is
  blocked by a low-priority thread that is being starved by medium-priority
  threads.
- **Priority inheritance:** A protocol that temporarily raises a lock holder's
  priority to that of the highest-priority thread waiting for the lock.
- **Robust mutex:** A GNU extension mutex that reports `EOWNERDEAD` to the
  next locker if the previous owner died while holding it.
- **Recursive mutex:** A mutex that may be locked multiple times by the same
  thread, and must be unlocked the same number of times.

### Mutex types

| Type | Behaviour on double lock | Behaviour on non-owner unlock |
|---|---|---|
| `NORMAL` | deadlocks | undefined |
| `ERRORCHECK` | returns `EDEADLK` | returns `EPERM` |
| `ADAPTIVE` | deadlocks (after spinning) | undefined |
| `RECURSIVE` | allowed | only the owner, once per lock |

### Priority protocols

| Protocol | Effect |
|---|---|
| `PTHREAD_PRIO_NONE` | none (default) |
| `PTHREAD_PRIO_INHERIT` | holder inherits the waiter's priority |
| `PTHREAD_PRIO_PROTECT` | holder is raised to a fixed ceiling |

### Frequently Asked Questions

- What is a mutex? Explain its different types.
- Differentiate between mutex and semaphore.
- What is priority inversion? How is it solved?
- What is a robust mutex and why is it useful?
- Explain the difference between an error-checking mutex and a normal mutex.
- When would you use a recursive mutex?

---

# Key Takeaways

- A mutex guarantees **mutual exclusion and ownership**. Nothing more: not
  fairness, not condition waiting, not progress.
- Every mutex should be configured with three attributes: **type**
  (`ERRORCHECK` in development), **protocol** (`PTHREAD_PRIO_INHERIT` when
  priorities differ), and **robustness** (`PTHREAD_MUTEX_ROBUST` in servers).
- `PTHREAD_MUTEX_NORMAL` is **undefined behaviour** on double lock. Use
  `ERRORCHECK` while developing, and switch to `NORMAL` only when the code is
  known-clean.
- A recursive mutex hides a design problem more often than it solves one.
  Prefer splitting `f()` and `f_locked()`.
- **Priority inversion** is real and caused a hardware failure (the 1997 Mars
  Pathfinder). `PTHREAD_PRIO_INHERIT` bounds it to one critical section.
- A **robust** mutex survives a thread dying while holding the lock, via
  `EOWNERDEAD` and `pthread_mutex_consistent`.
- glibc's mutex is futex-based: **one atomic instruction** uncontended, a
  **syscall only when a thread actually sleeps**. A spinlock is never better
  except for very short critical sections.
- Use a **scope guard** to make early returns safe, and a `pthread_once` for
  lazy initialisation.
- Before reaching for a mutex, check whether a single `_Atomic` variable
  would do — for a counter, that is faster and scales with cores.

---

# References

- Operating System Concepts — Silberschatz, §6.6.2, §6.6.3
- Modern Operating Systems — Tanenbaum, §2.6.3
- The Little Book of Semaphores — Allen Downey, chapter 2
- POSIX / `pthreads`: `man 3 pthread_mutexattr_settype`,
  `man 3 pthread_mutexattr_setprotocol`, `man 3 pthread_mutex_lock`
- Sha, Rajkumar, Lehoczky, "Priority Inheritance Protocols: An Approach to
  Real-Time Synchronization", IEEE Trans. Computers, 1990
- Mars Pathfinder incident, 1997 — priority inversion on a rover
- Ulrich Drepper, *Futexes Are Tricky*, 2012
- `strace -c -e futex`, `perf record -g`, `valgrind --tool=helgrind`
