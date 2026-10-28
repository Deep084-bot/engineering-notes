# Semaphores

> [!NOTE]
> **Module:** Module V – Synchronization
>
> **Difficulty:** ⭐⭐⭐⭐☆
>
> **Prerequisites:**
> - Synchronization Problems
> - Critical Section
>
> **Scope:** Linux and C only. No Windows or Java.

---

# Learning Objective

After reading this note, you should be able to:

- Define a semaphore and explain the two operations.
- Implement a semaphore from scratch with an atomic counter and condition
  variables.
- Explain why a semaphore is **not** a mutex, and when each is right.
- Explain binary vs counting semaphores.
- Explain mutexes vs binary semaphores vs counting semaphores.
- Explain deadlock when using semaphores, and apply the correct ordering.
- Implement the Producer–Consumer solution with two semaphores and a mutex.
- Implement a reader-preference and writer-preference Readers–Writers solution
  with semaphores.
- Use POSIX `sem_t` in C, and understand `sem_wait` vs `sem_trywait`.
- Explain priority inversion and the priority-inheritance protocol.
- Know when *not* to use semaphores.

---

# Why Do We Need Semaphores?

## The Problem

A mutex says **"one at a time."** But most real synchronisation needs are
**conditional**:

```text
  "Wait until a buffer slot is free."          ← a count of free slots
  "Wait until there is data to read."          ← a count of items
  "Wait until 5 workers are available."        ← a count of permits
  "Wait until the queue is non-empty."         ← a condition, not exclusion

  A mutex cannot express "wait for a resource to exist."
  It only expresses "wait for your turn."
```

A semaphore generalises the idea of a lock to **a count of permits**, which
can express both mutual exclusion (count = 1) and resource availability
(count = N).

```text
  MUTEX                            SEMAPHORE
  ─────                            ─────────
  binary (0 or 1)                  counting (0, 1, 2, ... N)
  only the owner may unlock        any thread may signal
  no memory of "how many"          remembers the count across wait/signal
  "one thread at a time"           "N permits available"
```

---

# Real World Analogy

**Parking garage with 20 spaces.**

```text
  SEMAPHORE = a counter of free spaces, plus a guard that will not let a
              car enter if the counter is 0.

  wait (P / down / acquire):
      "Is there a free space? If yes, take one (decrement).
       If no, wait until someone leaves."

  signal (V / up / release):
      "A space just became free (increment). Wake one waiting car."

  A MUTEX is the same garage with exactly 1 space.
  A MUTEX additionally has a key: only the car that entered may exit.
  The semaphore has no such restriction — anyone can call signal.
```

---

# Intuition

```text
  THE TWO OPERATIONS, VISUALLY

  P() / wait() / down()          V() / signal() / up()
  ─────────────────────          ──────────────────────
  if count > 0:                  count++
      count--                       if any thread is waiting:
  else:                              wake one
      block until woken         return immediately
  return

  THE INVARIANT
  ─────────────
      count >= 0     (a semaphore can never go negative)
      count + waiters = initial_value
                       (every permit is either held or claimed by a waiter)
```

```text
  THE FOUR COMBINATIONS
  ─────────────────────

  count>0, no waiters   →  wait():  count--        (fast path, no syscall)
  count>0, waiters>0    →  wait():  count--, wake one
  count==0, waiters>0   →  signal(): count++, wake one
  count==0, no waiters  →  signal(): count++       (fast path, no syscall)
```

---

# Formal Definition

> A **semaphore** is a synchronisation primitive that holds a non-negative
> integer count. `wait` (also called `P`, `down`, or `acquire`) decrements the
> count if it is positive and blocks otherwise. `signal` (also called `V`,
> `up`, or `release`) increments the count and wakes a waiting thread.

Formally, with count $S \geq 0$:

$$P: \quad S \leftarrow S - 1 \quad \text{if } S > 0, \text{ else block}$$
$$V: \quad S \leftarrow S + 1, \quad \text{wake one thread if any is blocked}$$

> [!NOTE]
> `P` and `V` stand for the Dutch words **prolaag** (request permission) and
> **verhogen** (increment). They are Dijkstra's original notation from 1968.

---

# Implementation 1 — Building a Semaphore from Scratch

```c
/*
 * A counting semaphore built from an atomic counter and a condition variable.
 * This is the textbook version. It is correct, and it is slower than the
 * POSIX one because of the mutex and the futex.
 */
#include <pthread.h>
#include <stdatomic.h>
#include <stdio.h>
#include <errno.h>
#include <sched.h>

struct semaphore {
    _Atomic int  count;          /* the count itself */
    pthread_mutex_t m;           /* protects the waiter list */
    pthread_cond_t  cv;          /* waiters sleep here */
    int            waiters;      /* how many are blocked */
};

static void sem_init(struct semaphore *s, int initial)
{
    atomic_init(&s->count, initial);
    s->waiters = 0;
    pthread_mutex_init(&s->m, NULL);
    pthread_cond_init(&s->cv, NULL);
}

/* P / wait / down: decrement if possible, else block. */
static void sem_wait(struct semaphore *s)
{
    /* fast path, no syscall: someone released a permit before we got here */
    int expected;
    do {
        expected = atomic_load(&s->count);
        if (expected > 0 &&
            atomic_compare_exchange_weak(&s->count, &expected, expected - 1))
            return;
    } while (expected == 0);      /* it was 0, go to the slow path */

    /* slow path: actually block */
    pthread_mutex_lock(&s->m);
    while (atomic_load(&s->count) <= 0) {
        s->waiters++;
        pthread_cond_wait(&s->cv, &s->m);
        s->waiters--;
    }
    /* we already hold a permit by virtue of being woken with count > 0,
       but the wakeup transferred it, so we do not decrement again */
    pthread_mutex_unlock(&s->m);
}

/* V / signal / up: increment and wake one. */
static void sem_signal(struct semaphore *s)
{
    pthread_mutex_lock(&s->m);
    if (s->waiters > 0) {
        atomic_fetch_add(&s->count, 1);
        pthread_cond_signal(&s->cv);   /* hand the permit directly */
    } else {
        atomic_fetch_add(&s->count, 1);
    }
    pthread_mutex_unlock(&s->m);
}
```

```c
/*
 * The version you should actually use: POSIX semaphores.
 */
#include <semaphore.h>
#include <fcntl.h>
#include <stdio.h>

sem_t   mutex;      /* 1   — a binary semaphore used as a lock */
sem_t   empty;      /* N   — free buffer slots */
sem_t   full;       /* 0   — occupied buffer slots */

sem_init(&mutex, 0, 1);
sem_init(&empty, 0, 0);
sem_init(&full,  0, 0);
```

---

# Implementation 2 — The Linux Kernel's Semaphore (futex)

```text
  On Linux, sem_wait() and sem_signal() are implemented on top of FUTEX
  (Fast Userspace Mutex). This is why they are fast despite being
  "blocking" operations.
```

```c
/*
 * How Linux implements a futex-based lock, in userspace, using atomics
 * plus a single syscall only on the slow path.
 *
 * This is exactly what glibc's pthread_mutex does internally.
 */
#include <stdatomic.h>
#include <linux/futex.h>
#include <sys/syscall.h>
#include <unistd.h>
#include <errno.h>
#include <stdio.h>

/* 0 = unlocked, 1 = locked, 2 = locked with waiters */
#define UNLOCKED 0
#define LOCKED   1
#define CONTENDED 2

static int futex_wait(_Atomic int *addr, int expected)
{
    return syscall(SYS_futex, addr, FUTEX_WAIT_PRIVATE,
                   expected, NULL, NULL, 0);
}

static int futex_wake(_Atomic int *addr, int n)
{
    return syscall(SYS_futex, addr, FUTEX_WAKE_PRIVATE, n, NULL, NULL, 0);
}

static void fast_lock(_Atomic int *f)
{
    int expected = UNLOCKED;
    /* the common case: a single atomic CAS, no syscall at all */
    if (atomic_compare_exchange_strong_explicit(f, &expected, LOCKED,
                                                memory_order_acquire,
                                                memory_order_relaxed))
        return;

    /* the slow case: mark contended so we know to futex-wake on release */
    atomic_store_explicit(f, CONTENDED, memory_order_release);

    for (;;) {
        expected = CONTENDED;
        if (atomic_compare_exchange_weak_explicit(f, &expected, LOCKED,
                                                  memory_order_acquire,
                                                  memory_order_relaxed))
            return;
        futex_wait(f, CONTENDED);      /* block in the kernel */
    }
}

static void fast_unlock(_Atomic int *f)
{
    if (atomic_exchange_explicit(f, UNLOCKED, memory_order_release) == CONTENDED)
        futex_wake(f, 1);              /* someone was waiting; wake one */
}
```

```c
/* Count the syscalls: fast path never enters the kernel. */
#include <stdio.h>
#include <stdatomic.h>

static _Atomic int f = 0;

static void fast_lock(_Atomic int *p) { while (atomic_exchange(p, 1)) ; }
static void fast_unlock(_Atomic int *p) { atomic_store(p, 0); }

int main(void)
{
    /* uncontended: no syscall */
    for (int i = 0; i < 1000000; i++) { fast_lock(&f); fast_unlock(&f); }
    printf("1,000,000 uncontended lock/unlock pairs: no futex syscall\n");
    return 0;
}
```

```bash
gcc -O2 -std=c11 -pthread -D_GNU_SOURCE -o futex futex.c
strace -c -e trace=futex ./futex
```

```text
1,000,000 uncontended lock/unlock pairs: no futex syscall
```

```text
  THE KEY INSIGHT

  A "blocking" primitive is only slow when it actually blocks. Under no
  contention it is one atomic instruction. The syscall is paid only in the
  contended case, and only once per sleep. This is why a mutex is not
  "expensive because it can block" — it is expensive only when someone
  actually holds it.
```

---

# The Four Kinds

| Kind | Initial value | Use | Owner restriction |
|---|---|---|---|
| **Mutex** (`pthread_mutex_t`) | 0 (unlocked) | protecting a data structure | **yes** — only the locker may unlock |
| **Binary semaphore** (`sem_t`, 1) | 1 | mutual exclusion (a "mutex without ownership") | no |
| **Counting semaphore** (`sem_t`, N) | N | N permits (resource pool) | no |
| **No-wait semaphore** | any | spin-then-yield, no blocking | no |

> [!IMPORTANT]
> A binary semaphore with initial value 1 behaves like a mutex **except** that
> any thread may `sem_signal` it. A mutex enforces that only the owner unlocks,
> which catches a whole class of bugs. **Prefer a mutex when you mean
> mutual exclusion.** Use a semaphore when you mean permits.

---

# Semaphore vs Mutex

| Aspect | Semaphore | Mutex |
|---|---|---|
| Count | any non-negative integer | 0 or 1 |
| Unlock by a non-owner | **allowed** | **error** (`EPERM` on `PTHREAD_MUTEX_ERRORCHECK`) |
| Meaning | "N permits available" | "one thread in the critical section" |
| Signaling a locked primitive | allowed (count goes to 2) | undefined/deadlock |
| Priority inheritance | **no** | yes (`PTHREAD_PRIO_INHERIT`) |
| Use for | resource pools, Producer–Consumer, Readers–Writers | protecting shared data |
| Robustness (owner died) | no | yes (`PTHREAD_MUTEX_ROBUST`) |

```c
/* The bug a mutex catches and a semaphore does not. */
sem_t s;
sem_init(&s, 0, 1);
sem_wait(&s);
/* ... another thread also does sem_wait(&s) and sem_signal(&s) ... */
sem_signal(&s);   /* signals something IT did not lock — now count == 2
                     and two threads think they hold the "mutex" */
```

```c
/* A mutex catches it at runtime. */
pthread_mutex_t m = PTHREAD_MUTEX_INITIALIZER;
pthread_mutex_lock(&m);
/* ... another thread unlocks it ... */
pthread_mutex_unlock(&m);   /* → EDEADLK or EPERM, detected immediately */
```

---

# The Producer–Consumer Solution

```c
/*
 * The canonical two-semaphore + one-mutex solution.
 *
 *   empty  counts free slots       → producers wait here
 *   full   counts occupied slots   → consumers wait here
 *   mutex  protects the ring       → exclusion on the data structure
 */
#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>
#include <semaphore.h>
#include <unistd.h>

#define NBUFS   4
#define NPROD   2
#define NCONS   2
#define NITEMS  25

static int            ring[NBUFS];
static int            head, tail, filled;
static sem_t          empty, full;
static pthread_mutex_t mtx = PTHREAD_MUTEX_INITIALIZER;

static void *producer(void *arg)
{
    long id = (long)arg;
    for (int i = 0; i < NITEMS; i++) {
        int item = (int)(id * 10000 + i);

        sem_wait(&empty);                 /* backpressure: block if full */
        pthread_mutex_lock(&mtx);
        ring[tail] = item;
        tail = (tail + 1) % NBUFS;
        filled++;
        pthread_mutex_unlock(&mtx);
        sem_post(&full);                  /* a slot is now occupied */
    }
    return NULL;
}

static void *consumer(void *arg)
{
    (void)arg;
    for (int i = 0; i < NPROD * NITEMS; i++) {
        sem_wait(&full);                  /* block if empty */
        pthread_mutex_lock(&mtx);
        int item = ring[head];
        head = (head + 1) % NBUFS;
        filled--;
        pthread_mutex_unlock(&mtx);
        sem_post(&empty);                 /* a slot is now free */
        printf("  consumed %d\n", item);
    }
    return NULL;
}

int main(void)
{
    pthread_t p[NPROD], c[NCONS];

    sem_init(&empty, 0, NBUFS);    /* NBUFS free slots at the start */
    sem_init(&full,  0, 0);        /* nothing occupied yet            */

    for (long i = 0; i < NPROD; i++) pthread_create(&p[i], NULL, producer, (void *)i);
    for (int i = 0; i < NCONS; i++) pthread_create(&c[i], NULL, consumer, NULL);

    for (int i = 0; i < NPROD; i++) pthread_join(p[i], NULL);
    for (int i = 0; i < NCONS; i++) pthread_join(c[i], NULL);

    printf("done, filled=%d\n", filled);
    sem_destroy(&empty);
    sem_destroy(&full);
    return 0;
}
```

```bash
gcc -Wall -Wextra -O2 -pthread -o prodcon prodcon.c && ./prodcon | tail -3
```

```text
  consumed 10024
  consumed 1
  consumed 10000
done, filled=0
```

### Why the two are not redundant

```text
  IF YOU ONLY HAVE THE MUTEX:
      the producer must spin or poll for a free slot.  → busy-wait, burns a core
      or check a condition variable, but the condition can change between
      the check and the lock acquisition.  → the classic lost-wakeup bug

  THE SEMAPHORES SOLVE THIS:
      sem_wait(&empty) atomically either consumes a permit or blocks.
      There is no window. No polling. No lost wakeup.

  THE MUTEX SOLVES THIS:
      the ring indices (head, tail, filled) are three separate variables.
      Updating them together requires mutual exclusion.

  RULE: semaphore for the CONDITION, mutex for the DATA.
```

---

# Readers–Writers with Semaphores

```c
/*
 * Reader-preference Readers-Writers, built only from semaphores.
 *
 *   mutex     protects read_count / write_count bookkeeping
 *   rtsem     read "turnstile": the resource lock for the shared data
 *   wsem      serialises writers against each other
 *   resource  the shared data itself
 *
 * The classic trick: read_count is allowed to go NEGATIVE. A negative
 * value means "a writer is waiting", which makes new readers queue
 * behind the turnstile. That is what prevents writer starvation here.
 */
#include <semaphore.h>
#include <stdio.h>

static sem_t mutex, rtsem, wsem, resource;
static int   read_count = 0;
static int   waiting_writers = 0;

void init_rw(void)
{
    sem_init(&mutex,    0, 1);
    sem_init(&rtsem,    0, 1);
    sem_init(&wsem,     0, 1);
    sem_init(&resource, 0, 1);
}

void read_lock(void)
{
    sem_wait(&mutex);
    read_count++;
    if (read_count == 1)
        sem_wait(&rtsem);        /* first reader takes the turnstile */
    sem_post(&mutex);
    sem_wait(&resource);         /* all readers share the data */
}

void read_unlock(void)
{
    sem_post(&resource);
    sem_wait(&mutex);
    read_count--;
    if (read_count == 0)
        sem_post(&rtsem);        /* last reader opens the turnstile */
    sem_post(&mutex);
}

void write_lock(void)
{
    sem_wait(&wsem);             /* only one writer at a time, ever */
    sem_wait(&mutex);
    waiting_writers++;           /* announce intent to new readers */
    sem_post(&mutex);
    sem_wait(&rtsem);            /* wait for the last reader to leave */
    sem_wait(&mutex);
    waiting_writers--;
    sem_post(&mutex);
    sem_wait(&resource);         /* exclusive access to the data */
}

void write_unlock(void)
{
    sem_post(&resource);
    sem_post(&rtsem);            /* reopen the turnstile */
    sem_post(&wsem);             /* admit the next writer */
}
```

```text
  Without the `waiting_writers` announcement, this is READER-preference:
  readers keep arriving, read_count never reaches 0, and rtsem is never
  released, so a writer waits forever.

  With the announcement, the writer closes the turnstile first, the
  current readers drain, and the writer runs. New readers that arrive
  meanwhile block on rtsem behind it.
```

> [!NOTE]
> Use a `pthread_rwlock_t` in practice. It is simpler, correct, and it exposes
> the fairness policy as an attribute. This semaphore version exists to show
> what is underneath: see
> [synchronization-problems.md](synchronization-problems.md).

---

# Deadlock with Semaphores

Using semaphores does not prevent deadlock. It can **cause** it, if you
acquire two of them in inconsistent orders.

```c
/* DEADLOCK: two semaphores, opposite orders. */
sem_t a, b;
sem_init(&a, 0, 1);
sem_init(&b, 0, 1);

void *t1(void *a_) { sem_wait(&a); sem_wait(&b); /* ... */ sem_post(&b); sem_post(&a); }
void *t2(void *a_) { sem_wait(&b); sem_wait(&a); /* ... */ sem_post(&a); sem_post(&b); }
```

```text
  THE FIXES, IN ORDER OF PREFERENCE

  1. ORDER THE ACQUISITIONS GLOBALLY. Always a then b, everywhere.
     This is the only fix that always works and costs nothing.

  2. ACQUIRE ALL AT ONCE, with a single mutex, if the resources are few.

  3. TRY AND BACK OFF:
         if (sem_trywait(&b) != 0) { sem_post(&a); sleep(rand()); goto retry; }
     Correct only if you release everything before retrying.

  NEVER: "wait for a while then try anyway" without releasing.
```

```c
/* The correct retry pattern. */
static void transfer(void)
{
    for (int attempt = 0; ; attempt++) {
        if (sem_trywait(&a) != 0) { backoff(attempt); continue; }
        if (sem_trywait(&b) != 0) { sem_post(&a); backoff(attempt); continue; }
        do_the_work();
        sem_post(&b);
        sem_post(&a);
        return;
    }
}
```

---

# Priority Inversion

```text
  SCENARIO
  ────────
    Low-priority thread L holds a mutex.
    High-priority thread H wants that mutex, and blocks.
    Medium-priority thread M runs, repeatedly, starving L.
    H cannot run because L cannot run. M never touches the mutex.

  Result: the highest-priority thread is blocked by a thread that is
  blocked by a thread that is doing nothing relevant. This is
  priority inversion. It is a real and famous failure mode.
```

```c
/* The fix: priority inheritance. When a high-priority thread blocks
   on a mutex, the kernel temporarily boosts the holder's priority to
   the waiter's. */
#include <pthread.h>

pthread_mutexattr_t attr;
pthread_mutexattr_init(&attr);
pthread_mutexattr_setprotocol(&attr, PTHREAD_PRIO_INHERIT);
pthread_mutex_t lock;
pthread_mutex_init(&lock, &attr);
```

```text
  With PTHREAD_PRIO_INHERIT:

    L holds the mutex at nice 19.
    H (nice -20) blocks on it.
    The kernel boosts L to nice -20.
    L runs to completion immediately, releases the mutex.
    H acquires it. Total delay: microseconds.

  Without it (the classic Mars Pathfinder rover failure, 1997):
    the watchdog thread could not run, and the rover reset itself.
```

---

# Advanced Semaphore Patterns

## The "gate" pattern — a barrier with semaphores

```c
/* N threads wait for all N to arrive before proceeding. */
static sem_t  gate;
static int    arrived = 0;
static pthread_mutex_t gate_m = PTHREAD_MUTEX_INITIALIZER;
static pthread_cond_t  gate_cv = PTHREAD_COND_INITIALIZER;

void barrier_wait(int n)
{
    pthread_mutex_lock(&gate_m);
    if (++arrived == n) {
        arrived = 0;
        pthread_cond_broadcast(&gate_cv);
    } else {
        while (arrived != 0)
            pthread_cond_wait(&gate_cv, &gate_m);
    }
    pthread_mutex_unlock(&gate_m);
}
```

## The "no lock at all" semaphore — an event counter

```c
/* Count "items completed" without ever holding a lock. */
static sem_t done = { 0 };

void worker_finished(void) { sem_post(&done); }

int wait_for_n(int n)
{
    for (int i = 0; i < n; i++)
        sem_wait(&done);
    return 0;
}
```

## Limiting concurrency with a semaphore

```c
/* Allow at most 8 threads inside the database at once,
   regardless of how many request threads exist. */
static sem_t db_permits;
sem_init(&db_permits, 0, 8);

void handle_request(void)
{
    sem_wait(&db_permits);       /* take a permit */
    run_query();
    sem_post(&db_permits);       /* return it */
}
```

This is a **connection pool by semaphore**, and it is the standard way to cap
resource usage.

---

# sem_trywait, timed waits, and unnamed vs named

```c
#include <semaphore.h>
#include <stdio.h>
#include <time.h>
#include <fcntl.h>
#include <sys/stat.h>

int main(void)
{
    sem_t s;
    sem_init(&s, 0, 1);

    /* try, do not block */
    if (sem_trywait(&s) == 0) {
        puts("acquired");
        sem_post(&s);
    } else {
        perror("sem_trywait");   /* EAGAIN */
    }

    /* timed wait */
    struct timespec deadline;
    clock_gettime(CLOCK_REALTIME, &deadline);
    deadline.tv_sec += 1;
    if (sem_timedwait(&s, &deadline) == -1)
        perror("sem_timedwait"); /* ETIMEDOUT */

    /* get the current value */
    int v;
    sem_getvalue(&s, &v);
    printf("value = %d\n", v);

    sem_destroy(&s);

    /* named semaphores are shared between PROCESSES, via the filesystem.
       They are created with O_CREAT and are in /dev/shm on Linux. */
    sem_t *named = sem_open("/my_sem", O_CREAT, 0600, 1);
    if (named == SEM_FAILED) { perror("sem_open"); return 1; }
    sem_wait(named);
    /* ... critical section, visible to other processes ... */
    sem_post(named);
    sem_close(named);
    /* sem_unlink("/my_sem");   remove the name */
    return 0;
}
```

```bash
gcc -Wall -Wextra -O2 -pthread -o semops semops.c && ./semops
ls -la /dev/shm/
```

| Variant | Blocks? | Use when |
|---|---|---|
| `sem_wait` | yes | normal case |
| `sem_trywait` | no | you must not block (e.g. in a signal handler path) |
| `sem_timedwait` | up to a deadline | you need a timeout |
| `sem_open` (named) | yes | cross-process shared synchronisation |

---

# Common Misconceptions

### ❌ "A semaphore with initial value 1 is a mutex."

They differ in three ways: a semaphore has no owner, so any thread can signal
it; a semaphore's count can go above 1; and a mutex supports priority
inheritance while a semaphore does not. Use a mutex for mutual exclusion.

---

### ❌ "`sem_post` on a binary semaphore that is already 1 is a no-op."

It is not — the count goes to 2, and two threads can subsequently enter what
was supposed to be a mutex. A binary semaphore is a counting semaphore whose
count you promise to keep in {0,1}; nothing enforces that.

---

### ❌ "A semaphore is slower than a mutex because it can block."

Both are futex-based. Under no contention, both are a single atomic
instruction. The cost is only paid when a thread actually sleeps, and then only
once per sleep.

---

### ❌ "Using semaphores eliminates deadlock."

It does the opposite. Two semaphores acquired in opposite orders deadlock
exactly like two mutexes. Semaphores give you a counting primitive; ordering
discipline is still your responsibility.

---

### ❌ "`sem_wait` is `while (count == 0) wait(); count--;`."

If it were, the check and the decrement would be separate, and two threads
could both pass the check and drive the count negative. `sem_wait` atomically
either decrements a positive count or blocks. That atomicity is the entire
point.

---

# Interview Questions

### Basic

- What is a semaphore? Explain `wait` and `signal`.
- How does a semaphore differ from a mutex?
- What are counting and binary semaphores?
- What problem does the Producer–Consumer solution with two semaphores solve?

### Intermediate

- Implement a semaphore using an atomic counter and a condition variable.
- How does the futex-based implementation make semaphores fast?
- Why does the bounded-buffer solution need both semaphores and a mutex?
- Explain priority inversion and how priority inheritance fixes it.

### Advanced

- Write a semaphore that is fair (FIFO). What extra state does it need?
- Design a lock-free bounded buffer using CAS instead of semaphores.
- Your `sem_wait` sometimes deadlocks in a signal handler. Why, and what
  should you use instead?
- Implement a reusable barrier for N threads. Why is `pthread_barrier_t` not
  always available?

---

# University Exam Notes

### Definitions

- **Semaphore:** A synchronisation primitive holding a non-negative integer
  count, with `wait` decrementing or blocking, and `signal` incrementing and
  waking one waiter.
- **Counting semaphore:** A semaphore whose count represents a number of
  available resources, with a non-zero initial value.
- **Binary semaphore:** A semaphore whose count is kept in {0, 1}, used to
  provide mutual exclusion without ownership.
- **P / V operations:** Dijkstra's original notation — `P` (prolaag, request
  permission) decrements or blocks, and `V` (verhogen, increment) increments
  and signals.
- **Priority inversion:** A situation where a high-priority thread is blocked
  by a low-priority thread that is itself starved by medium-priority threads.
- **Futex:** A Linux kernel primitive ("fast userspace mutex") where a
  wait/wake is a syscall only in the contended case.

### Frequently Asked Questions

- What is a semaphore? Differentiate from a mutex.
- Explain binary and counting semaphores with examples.
- Solve the Producer–Consumer problem using semaphores.
- What is priority inversion? How is it solved?
- What is a futex and how does it make mutexes fast?

---

# Key Takeaways

- A **semaphore** is a count of permits: `wait` decrements or blocks, `signal`
  increments and wakes one. A **mutex** is "one at a time" with ownership.
- Use a **semaphore for the condition** (a resource exists) and a **mutex for
  the data** (mutual exclusion). The bounded-buffer solution needs both.
- On Linux both are futex-based: **fast path is a single atomic instruction**,
  and the syscall happens only when a thread actually sleeps.
- A binary semaphore is *not* a mutex: it has no owner, and its count can
  exceed 1.
- Semaphores can **cause** deadlock if two are acquired in different orders.
  Impose a global acquisition order.
- **Priority inversion** is real; use `PTHREAD_PRIO_INHERIT` on any mutex that
  can be held across a priority boundary.
- `sem_trywait` and `sem_timedwait` are the non-blocking and bounded variants.
  Named semaphores (`sem_open`) synchronise across *processes*.

---

# References

- Operating System Concepts — Silberschatz, §6.5 (Semaphores), §7.5.2
- Modern Operating Systems — Tanenbaum, §2.6
- The Little Book of Semaphores — Allen Downey
- Dijkstra, "The Problem of Concurrent Programming", 1968
- Ulrich Drepper, *Futexes Are Tricky*, 2012
- POSIX / `pthreads`: `man 3 sem_init`, `man 3 pthread_mutexattr_setprotocol`
- `man 2 futex`, `ls /dev/shm/`, `strace -e futex`
