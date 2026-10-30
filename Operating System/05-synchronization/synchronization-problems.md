# Synchronization Problems

> [!NOTE]
> **Module:** Module V – Synchronization
>
> **Difficulty:** ⭐⭐⭐⭐☆
>
> **Prerequisites:**
> - Critical Section
> - Threads
>
> **Scope:** Linux and C only. No Windows or Java.

---

# Learning Objective

After reading this note, you should be able to:

- State the three classic synchronisation problems and what each one actually
  solves.
- Explain the Producer–Consumer (Bounded Buffer) problem precisely.
- Explain the Readers–Writers problem and all three of its variants.
- Explain the Dining Philosophers problem and why it is still a real problem.
- Write a correct solution to each, in C, using pthreads.
- Explain how the three problems generalise into the same underlying problem:
  managing multiple instances of a limited resource.
- Explain why these problems are not artificial.

---

# Why Do We Need This Note?

## The Problem

The previous notes gave you the **primitives**: mutual exclusion
([test-and-set.md](test-and-set.md)), condition waiting
([semaphores.md](semaphores.md)), and exclusion with ownership
([mutex.md](mutex.md)).

This note gives you the **problems those primitives exist to solve**. Each is
a small, self-contained specification of what correct concurrent behaviour
looks like, and each is small enough to hold entirely in your head.

```text
  THE ONE IDEA UNDERNEATH ALL THREE

  Every one of these problems is the same problem:

      there is a limited, shared resource
          │
          ├── with a CAPACITY            (N buffer slots, N philosophers)
          ├── accessed in DIFFERENT WAYS  (read-only vs read-write)
          ├── by ACTORS with DIFFERENT NEEDS (producers vs consumers)
          └── that must not be OVERUSED   (2 forks per philosopher)

  Synchronisation is the management of that capacity, under
  concurrency, without deadlock or starvation.
```

---

# Real World Analogy

A single-bathroom apartment shared by several housemates, with a second
shared resource in the kitchen.

- **Producer–Consumer:** the fridge. Housemates put groceries in (producers)
  and take them out (consumers). Nobody can take a grocery that is not there,
  and nobody can put a fourth grocery in a three-shelf fridge.
- **Readers–Writers:** the bathroom. One person showering (a writer) excludes
  everyone. Three people just brushing teeth (readers) can do it at once.
- **Dining Philosophers:** five people, one bowl of spaghetti each, but only
  five forks, and the forks are shared between neighbouring diners.

---

# Intuition

```text
  THE SHARED STATE IN EACH PROBLEM

  Producer-Consumer     READER     WRITER
  ┌──────────────┐     ┌──────────────┐
  │ empty slots  │     │   reader     │  ← many at once
  │ full slots   │     │   count      │
  │ item buffer  │     │   isWriting  │  ← at most one
  └──────────────┘     └──────────────┘

  Dining Philosophers  no shared DATA at all
  ┌──────────────┐     Only 5 forks, each usable by exactly 2 people,
  │ 5 forks      │     and each person needs 2 at once.
  │ 5 bowls      │     The problem is pure resource acquisition ordering.
  └──────────────┘
```

---

# Formal Definition

> **A synchronisation problem** specifies a set of concurrent processes that
> share resources, together with a correctness condition they must maintain
> and an ordering or timing requirement they must satisfy. The task is to find
> an algorithm that satisfies both.

| Element | Meaning |
|---|---|
| Shared state | the data or resources more than one process touches |
| Correctness condition | an invariant that must hold at every point (e.g. "no two writers") |
| Progress | something must eventually run, not deadlock |
| Bounded waiting | a waiting process must not wait forever, not starve |
| Initial state | the state before any process starts |

> [!IMPORTANT]
> A correct solution must satisfy **mutual exclusion**, **progress**, and
> **bounded waiting** *simultaneously*. Most textbook solutions get the first
> one and fail on the third.

---

# Problem 1 — Producer–Consumer (Bounded Buffer)

## The specification

```text
  A buffer of N slots sits between producers and consumers.

  PRODUCER                          CONSUMER
  produce(item):                    consume():
    wait until                        wait until
      empty > 0                       full > 0
    put item in buffer                take item from buffer
    signal full                       signal empty
    signal empty                     
```

## The constraints

| Constraint | Statement |
|---|---|
| **Mutual exclusion** | only one producer writes the buffer, and only one consumer reads it, at a time |
| **Capacity** | a producer may not write when the buffer is full; a consumer may not read when it is empty |
| **Ordering** | items come out in the order they went in (if it is a FIFO) — a queue, not a bag |
| **No deadlock** | if the buffer is full, producers wait; if empty, consumers wait. Both must eventually be woken. |

## Why the naive version breaks

```c
/* BROKEN. Three separate bugs, all of them in real code. */
int buffer[N];
int in = 0, out = 0, count = 0;

void produce(int item) {
    while (count == N) ;            /* BUG 1: busy-wait, burns a core */
    buffer[in] = item;
    in = (in + 1) % N;
    count++;                        /* BUG 2: not atomic; lost update */
}

void consume(void) {
    while (count == 0) ;            /* BUG 1 again */
    int item = buffer[out];
    out = (out + 1) % N;
    count--;                        /* BUG 2 again */
}
```

| Bug | What goes wrong |
|---|---|
| busy-wait loop | burns 100% of a core spinning; on a 1-core machine, the consumer may never get scheduled, so the loop never ends |
| `count++` / `count--` are not atomic | read-modify-write. Two producers can both read `count = 3` and both write 4. One item is lost. |
| check and increment are separate | this is the **check-then-act** race: `count == N` passes, then another producer fills the last slot, then this one writes anyway |

## The correct solution: two counting semaphores and one mutex

```c
/*
 * Bounded buffer, solved with semaphores.
 *
 *   empty   counts FREE slots      → producers wait on it
 *   full    counts OCCUPIED slots  → consumers wait on it
 *   mutex   protects the ring index and the buffer itself
 */
#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>
#include <semaphore.h>
#include <unistd.h>

#define NBUFS   4
#define NITEMS  20

static int            buffer[NBUFS];
static int            head, tail, count;
static sem_t          empty, full;
static pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;

static int produced, consumed;

static void *producer(void *arg)
{
    long id = (long)arg;
    for (int i = 0; i < NITEMS; i++) {
        int item = (int)(id * 1000 + i);

        sem_wait(&empty);              /* wait for a free slot */
        pthread_mutex_lock(&mutex);
        buffer[tail] = item;           /* the ring write */
        tail = (tail + 1) % NBUFS;
        count++;
        produced++;
        pthread_mutex_unlock(&mutex);
        sem_post(&full);               /* a slot is now occupied */
    }
    return NULL;
}

static void *consumer(void *arg)
{
    (void)arg;
    for (int i = 0; i < NITEMS * 2; i++) {
        sem_wait(&full);               /* wait for something to take */
        pthread_mutex_lock(&mutex);
        int item = buffer[head];       /* the ring read */
        head = (head + 1) % NBUFS;
        count--;
        consumed++;
        pthread_mutex_unlock(&mutex);
        sem_post(&empty);              /* a slot is now free */
        printf("consumer took %d\n", item);
    }
    return NULL;
}

int main(void)
{
    pthread_t p[2], c[2];

    sem_init(&empty, 0, NBUFS);        /* NBUFS free slots */
    sem_init(&full,  0, 0);            /* 0 occupied slots */

    for (long i = 0; i < 2; i++) pthread_create(&p[i], NULL, producer, (void *)i);
    for (int i = 0; i < 2; i++) pthread_create(&c[i], NULL, consumer, NULL);

    for (int i = 0; i < 2; i++) pthread_join(p[i], NULL);
    for (int i = 0; i < 2; i++) pthread_join(c[i], NULL);

    printf("produced=%d consumed=%d final count=%d\n",
           produced, consumed, count);
    sem_destroy(&empty);
    sem_destroy(&full);
    return 0;
}
```

```bash
gcc -Wall -Wextra -O2 -pthread -o prodcon prodcon.c && ./prodcon
```

```text
consumer took 0
consumer took 1000
consumer took 1
consumer took 1001
...
produced=40 consumed=40 final count=0
```

## Why the counts are semaphores, not just a mutex

```text
  A mutex only says "one at a time". It cannot say "wait until there is
  something here".

  sem_wait(&full) is a CONDITIONAL wait: it blocks only while the
  condition is false, and it atomically decrements the count when it
  proceeds. That atomic decrement is exactly what a mutex does NOT give you.

  This is the crucial difference:

      mutex:            "wait for your turn, then go"  → mutual exclusion
      semaphore:        "wait until a resource exists" → condition synchronisation

  Two semaphores + one mutex is the canonical Producer-Consumer solution.
  See semaphores.md for the full mechanism.
```

## The generalised version

```text
  N producer threads, M consumer threads, buffer of size B.

  The solution does not change. Only the constants do:

      sem_init(&empty, 0, B);
      sem_init(&full,  0, 0);

  Design rule: the number of consumer slots is what provides BACKPRESSURE.
  If producers are faster than consumers, `empty` runs out and producers
  block. That is not a bug; that is the buffer telling the producers to
  slow down. An unbounded queue would instead exhaust memory.
```

---

# Problem 2 — Readers–Writers

## The specification

```text
  A shared database or file. Many readers, few writers.

  READER                              WRITER
  read_record():                      write_record():
    wait until no writer is active       wait until no writer
    register as a reader                AND no reader is active
    ... perform many reads ...          write the record
    unregister as a reader              release
```

| Constraint | Statement |
|---|---|
| **Multiple readers** | any number of readers may be active simultaneously |
| **Exclusive writer** | a writer has exclusive access; no reader, no other writer |
| **No reader starved by writers** | variant-dependent |
| **No writer starved by readers** | variant-dependent |

## The three variants

```text
  Reader-priority   readers get in as soon as they can, even if a
                    writer is waiting. A continuous stream of readers
                    STARVES a writer.               ← reader-preference

  Writer-priority   a waiting writer blocks all NEW readers. Existing
                    readers drain and the writer proceeds.          ← writer-preference

  Fair / no-priority  whichever is already active gets to finish; a
                    queue forms either way.                       ← no starvation
```

## Solution 1: a read-write lock (the practical answer)

```c
/*
 * Readers-Writers with a pthread rwlock.
 *
 * This is what you should use in real C code. It is implemented in
 * glibc, it is correct, and it handles the fairness policy for you.
 */
#include <stdio.h>
#include <pthread.h>
#include <unistd.h>

static pthread_rwlock_t db = PTHREAD_RWLOCK_INITIALIZER;
static int             shared_record = 0;
static long            reads, writes;

static void *reader(void *arg)
{
    (void)arg;
    for (int i = 0; i < 200; i++) {
        pthread_rwlock_rdlock(&db);
        long snapshot = shared_record;      /* must be consistent */
        reads++;
        pthread_rwlock_unlock(&db);
        (void)snapshot;
    }
    return NULL;
}

static void *writer(void *arg)
{
    (void)arg;
    for (int i = 1; i <= 200; i++) {
        pthread_rwlock_wrlock(&db);
        shared_record = i;                  /* exclusive */
        writes++;
        pthread_rwlock_unlock(&db);
    }
    return NULL;
}

int main(void)
{
    pthread_t r[4], w[2];

    for (int i = 0; i < 4; i++) pthread_create(&r[i], NULL, reader, NULL);
    for (int i = 0; i < 2; i++) pthread_create(&w[i], NULL, writer, NULL);

    for (int i = 0; i < 4; i++) pthread_join(r[i], NULL);
    for (int i = 0; i < 2; i++) pthread_join(w[i], NULL);

    printf("%ld reads, %ld writes, final value=%d\n", reads, writes, shared_record);
    return 0;
}
```

```bash
gcc -Wall -Wextra -O2 -pthread -o rwlock rwlock.c && ./rwlock
```

> [!CAUTION]
> A rwlock helps only if there are **many** concurrent readers. With one or two
> readers, the bookkeeping is pure overhead and a plain mutex is faster. Also
> note: a `rdlock` held for a long time blocks every writer, so do not hold one
> across I/O.

## Solution 2: the reader-writer solution from first principles

```c
/*
 * Readers-Writers built from a plain mutex and two condition variables.
 * This is what a rwlock actually does internally, and it shows the
 * writer-preference policy explicitly.
 */
#include <stdio.h>
#include <pthread.h>

#define NREADERS 5

static pthread_mutex_t mtx   = PTHREAD_MUTEX_INITIALIZER;
static pthread_cond_t  can_read = PTHREAD_COND_INITIALIZER;
static pthread_cond_t  can_write = PTHREAD_COND_INITIALIZER;

static int readers_active = 0;
static int writer_active = 0;
static int writers_waiting = 0;

static void read_lock(void)
{
    pthread_mutex_lock(&mtx);
    /* WRITER-PRIORITY: do not enter if a writer is queued */
    while (writer_active || writers_waiting > 0)
        pthread_cond_wait(&can_read, &mtx);
    readers_active++;
    pthread_mutex_unlock(&mtx);
}

static void read_unlock(void)
{
    pthread_mutex_lock(&mtx);
    readers_active--;
    if (readers_active == 0)
        pthread_cond_broadcast(&can_write);
    pthread_mutex_unlock(&mtx);
}

static void write_lock(void)
{
    pthread_mutex_lock(&mtx);
    writers_waiting++;
    while (writer_active || readers_active > 0)
        pthread_cond_wait(&can_write, &mtx);
    writers_waiting--;
    writer_active = 1;
    pthread_mutex_unlock(&mtx);
}

static void write_unlock(void)
{
    pthread_mutex_lock(&mtx);
    writer_active = 0;
    if (writers_waiting > 0)
        pthread_cond_broadcast(&can_write);   /* writers first */
    else
        pthread_cond_broadcast(&can_read);
    pthread_mutex_unlock(&mtx);
}

static int shared = 0;

static void *reader(void *arg)
{
    (void)arg;
    for (int i = 0; i < 1000; i++) {
        read_lock();
        int v = shared;
        (void)v;
        read_unlock();
    }
    return NULL;
}

static void *writer(void *arg)
{
    (void)arg;
    for (int i = 0; i < 1000; i++) {
        write_lock();
        shared = i;
        write_unlock();
    }
    return NULL;
}

int main(void)
{
    pthread_t r[NREADERS], w[1];
    for (int i = 0; i < NREADERS; i++) pthread_create(&r[i], NULL, reader, NULL);
    pthread_create(&w[0], NULL, writer, NULL);
    for (int i = 0; i < NREADERS; i++) pthread_join(r[i], NULL);
    pthread_join(w[0], NULL);
    printf("done, shared=%d\n", shared);
    return 0;
}
```

```bash
gcc -Wall -Wextra -O2 -pthread -o rw rw.c && ./rw
```

> [!IMPORTANT]
> Both `read_lock` and `write_lock` use a `while` loop around
> `pthread_cond_wait`, not an `if`. This is not stylistic. `cond_wait` can
> return spuriously, and it also returns when *some other* thread was signalled,
> not necessarily the one whose condition became true. The loop re-checks the
> predicate. An `if` is the single most common bug in hand-written
> condition-variable code.

## The bug in most hand-written attempts

```c
/* WRONG: signal wakes ONE arbitrary waiter, not necessarily the right class. */
static void write_unlock(void) {
    pthread_mutex_lock(&mtx);
    writer_active = 0;
    pthread_cond_signal(&can_read);   /* may wake another READER,
                                         while a writer waits forever */
    pthread_mutex_unlock(&mtx);
}
```

```text
  can_read and can_write are two different condition variables, so
  signal() on the wrong one wakes the wrong class. If any writer is
  waiting, you must broadcast on can_write.

  If you do not want to reason about classes, use ONE condition variable
  and broadcast. Broadcasting to an empty condition variable is nearly free.
```

---

# Problem 3 — Dining Philosophers

## The specification

Five philosophers sit at a round table. Between each pair is a fork. Each
philosopher alternates between thinking and eating. To eat, a philosopher must
hold **both** adjacent forks.

```text
              P0
          f4      f1
      P4              P1
          f3      f2
              P2
        (and P3 below P2)

  P_i needs f_i (left) and f_(i+1)%5 (right) to eat.
```

## Why it is famous

```text
  It looks trivial. It is not.

  Forks must be picked up one at a time. A philosopher who holds one fork
  and waits for the second is in a state that is impossible to reach in
  the real world, and it deadlocks:

      P0 holds f0, waits for f1
      P1 holds f1, waits for f2
      P2 holds f2, waits for f3
      P3 holds f3, waits for f4
      P4 holds f4, waits for f0      ← the cycle closes

  This is the textbook illustration of a CIRCULAR WAIT, and therefore
  of deadlock.
```

## Solution 1: asymmetric acquisition — break the wait-for cycle

```c
/*
 * Solution 1: asymmetric acquisition.
 *
 * P0 locks fork 0 then 1.  P1 locks 1 then 2.  P2 locks 2 then 3.
 * P3 locks 3 then 4.  P4 locks 0 then 4.
 *
 * The wait-for edges are therefore 0->1, 1->2, 2->3, 3->4, 0->4 --
 * a DAG, with no cycle back to 0. A cycle is the ONLY way a
 * mutex deadlock happens, so no cycle means no deadlock.
 *
 * Note this does NOT leave a fork unused: P4 really does take fork 0
 * and fork 4. What it does is impose a total order on the forks, and
 * always acquire in that order. That is the general technique -- see
 * Solution 3.
 */
#include <stdio.h>
#include <pthread.h>
#include <unistd.h>

#define NPHIL 5

static pthread_mutex_t stick[NPHIL];
static pthread_mutex_t meal_count_lock = PTHREAD_MUTEX_INITIALIZER;
static int meals = 0;

static void eat(int id)
{
    int first  = (id == NPHIL - 1) ? (id + 1) % NPHIL : id;   /* right if P4 */
    int second = (id == NPHIL - 1) ? id : (id + 1) % NPHIL;    /* left otherwise */

    pthread_mutex_lock(&stick[first]);
    pthread_mutex_lock(&stick[second]);

    printf("P%d eating (forks %d, %d)\n", id, first, second);

    pthread_mutex_lock(&meal_count_lock);
    meals++;
    pthread_mutex_unlock(&meal_count_lock);

    pthread_mutex_unlock(&stick[second]);
    pthread_mutex_unlock(&stick[first]);
}

static void *philosopher(void *arg)
{
    int id = (int)(long)arg;
    for (int i = 0; i < 20; i++) {
        printf("P%d thinking\n", id);
        usleep(1000);
        eat(id);
    }
    return NULL;
}

int main(void)
{
    pthread_t t[NPHIL];
    for (int i = 0; i < NPHIL; i++) pthread_mutex_init(&stick[i], NULL);
    for (int i = 0; i < NPHIL; i++) pthread_create(&t[i], NULL, philosopher, (void *)(long)i);
    for (int i = 0; i < NPHIL; i++) pthread_join(t[i], NULL);
    printf("total meals = %d\n", meals);
    return 0;
}
```

```bash
gcc -Wall -Wextra -O2 -pthread -o phil1 phil1.c && timeout 5 ./phil1 | tail -6
```

```text
P0 eating (forks 0, 1)
P3 eating (forks 3, 4)
P1 eating (forks 1, 2)
total meals = 100
```

## Solution 2: a waiter-and-resource-handoff pattern

```c
/*
 * Solution 2: one atomic "serve the next in line" routine.
 * This eliminates the hold-and-wait condition entirely: a philosopher
 * never holds a fork while waiting.
 */
#include <stdio.h>
#include <pthread.h>
#include <unistd.h>

#define NPHIL 5

static pthread_mutex_t state = PTHREAD_MUTEX_INITIALIZER;
static pthread_cond_t  can_eat[NPHIL];
static int eating[NPHIL];        /* which philosopher currently holds the pair */
static int meals = 0;

static void *philosopher(void *arg)
{
    int id = (int)(long)arg;
    for (int i = 0; i < 20; i++) {
        pthread_mutex_lock(&state);
        /* I need forks id and id+1. Any philosopher sharing either one
           blocks me: that is (id - 1), me, and (id + 1). Checking only
           the two that the common buggy version checks lets me collide
           with the philosopher on my other side. */
        while (eating[(id + NPHIL - 1) % NPHIL] ||
               eating[id] ||
               eating[(id + 1) % NPHIL])
            pthread_cond_wait(&can_eat[id], &state);
        eating[id] = 1;
        meals++;
        pthread_mutex_unlock(&state);

        usleep(500);
        printf("P%d eating\n", id);

        pthread_mutex_lock(&state);
        eating[id] = 0;
        /* I only ever block my two neighbours, so waking exactly those
           two is sufficient — and cheaper than a broadcast. */
        pthread_cond_signal(&can_eat[(id + NPHIL - 1) % NPHIL]);
        pthread_cond_signal(&can_eat[(id + 1) % NPHIL]);
        pthread_mutex_unlock(&state);
    }
    return NULL;
}

int main(void)
{
    pthread_t t[NPHIL];
    for (int i = 0; i < NPHIL; i++) pthread_cond_init(&can_eat[i], NULL);
    for (int i = 0; i < NPHIL; i++) pthread_create(&t[i], NULL, philosopher, (void *)(long)i);
    for (int i = 0; i < NPHIL; i++) pthread_join(t[i], NULL);
    printf("total meals = %d\n", meals);
    return 0;
}
```

```bash
gcc -Wall -Wextra -O2 -pthread -o phil2 phil2.c && timeout 5 ./phil2 | tail -4
```

## Solution 3: the resource hierarchy (the general lesson)

```text
  Number the resources 1..N. Always acquire in ASCENDING order.
  Release in any order.

  A philosopher may take f_i and f_(i+1) only by taking the lower-numbered
  one first. There is no longer any way to form a cycle, because a
  thread holding a high-numbered fork never waits for a low-numbered one.

  This is the SAME fix as the global lock ordering from
  threading-issues.md, and it is the fix you should reach for by default.
```

```c
/* The general form, for any number of resources. */
static void acquire_pair(int lo, int hi)
{
    pthread_mutex_lock(&res[lo]);      /* always the lower index first */
    pthread_mutex_lock(&res[hi]);
}

static void release_pair(int lo, int hi)
{
    pthread_mutex_unlock(&res[hi]);    /* any release order is safe */
    pthread_mutex_unlock(&res[lo]);
}
```

## Solution 4: waiter and arbiter — the one-way ticket

```c
/*
 * Solution 4: a single arbiter grants permission to eat.
 * The arbiter holds NO forks — it only grants the right to take them,
 * and it grants that right to exactly one philosopher at a time.
 *
 * Why this cannot deadlock: a philosopher only ever waits while it holds
 * nothing. Its forks are taken only after the arbiter has released it,
 * and the arbiter is released by the same philosopher when it finishes.
 * There is no cycle to form.
 */
#include <stdio.h>
#include <pthread.h>
#include <unistd.h>

#define NPHIL 5

static pthread_mutex_t arbiter = PTHREAD_MUTEX_INITIALIZER;
static pthread_cond_t  called    = PTHREAD_COND_INITIALIZER;  /* "I want in"    */
static pthread_cond_t  granted   = PTHREAD_COND_INITIALIZER;  /* "you may eat"  */
static int waiting[NPHIL];      /* how many want permission (for fairness)  */
static int grant_to   = -1;     /* the ONE philosopher allowed to eat, or -1  */
static int meals = 0;

static void *philosopher(void *arg)
{
    int id = (int)(long)arg;

    for (int i = 0; i < 20; i++) {
        /* --- 1. Ask the arbiter for permission, holding nothing. --- */
        pthread_mutex_lock(&arbiter);
        waiting[id] = 1;
        pthread_cond_signal(&called);
        while (grant_to != id)
            pthread_cond_wait(&granted, &arbiter);
        waiting[id] = 0;
        pthread_mutex_unlock(&arbiter);

        /* --- 2. Now, and only now, we are the sole eater. --- */
        usleep(300);
        printf("P%d eating\n", id);

        /* --- 3. Return the token. Hand it to a waiter if there is one. --- */
        pthread_mutex_lock(&arbiter);
        meals++;
        grant_to = -1;
        if (waiting[0] + waiting[1] + waiting[2] + waiting[3] + waiting[4] > 0)
            pthread_cond_signal(&called);   /* wake the arbiter below */
        pthread_mutex_unlock(&arbiter);
    }
    return NULL;
}

/* The arbiter owns the token. It hands it to exactly one waiter at a time. */
static void *arbiter_thread(void *arg)
{
    (void)arg;
    for (int i = 0; i < NPHIL * 20; i++) {
        pthread_mutex_lock(&arbiter);
        /* Wait until the token is free AND somebody actually wants it.
           Checking only `grant_to == -1` would spin through every
           iteration before the first philosopher ever asks, and the
           arbiter would exit with all of them still blocked. */
        while (grant_to != -1 ||
               !(waiting[0] || waiting[1] || waiting[2] || waiting[3] || waiting[4]))
            pthread_cond_wait(&called, &arbiter);

        for (int id = 0; id < NPHIL; id++)
            if (waiting[id]) { grant_to = id; break; }

        pthread_cond_broadcast(&granted);
        pthread_mutex_unlock(&arbiter);
    }
    return NULL;
}

int main(void)
{
    pthread_t t[NPHIL], arb;
    pthread_create(&arb, NULL, arbiter_thread, NULL);
    for (int i = 0; i < NPHIL; i++)
        pthread_create(&t[i], NULL, philosopher, (void *)(long)i);
    for (int i = 0; i < NPHIL; i++) pthread_join(t[i], NULL);
    pthread_join(arb, NULL);
    printf("total meals = %d\n", meals);
    return 0;
}
```

```bash
gcc -Wall -Wextra -O2 -pthread -o phil4 phil4.c && timeout 10 ./phil4 | tail -4
```

```text
  Note the trade-off: this is CORRECT but slow. Every philosopher that wants
  to eat must pass through the arbiter, so they take turns strictly, even
  when two of them could safely have eaten at the same time. Correctness
  first; the parallel version is Solution 1.
```

---

# Why These Problems Are Not Artificial

```text
  The textbook diagrams are abstractions. The situations are not.
```

| Text | Real equivalent |
|---|---|
| Dining philosophers: 5 forks, 2 needed | a thread pool with 8 workers, each needing 2 of 8 connection slots; a compiler needing a file handle and a buffer; a transaction needing 2 of 4 row locks |
| Producer–Consumer buffer | an OS pipe; a network socket's receive queue; a database WAL; a kernel's page cache; an audio callback pulling from a ring buffer |
| Readers–Writers | a shared config read by 100 request threads and rewritten on reload; an inode's block map; a shared character device |
| circular wait | two locks acquired in opposite orders in two code paths, one added later by a different developer |
| starving writer | a metrics flush that never gets a turn behind continuous reads |

```c
/*
 * The most common real-world form: two locks, acquired in different
 * orders by two functions. This is Dining Philosophers with N=2.
 */
static pthread_mutex_t cache_lock = PTHREAD_MUTEX_INITIALIZER;
static pthread_mutex_t stats_lock = PTHREAD_MUTEX_INITIALIZER;

static void record_and_cache(void) {
    pthread_mutex_lock(&cache_lock);
    pthread_mutex_lock(&stats_lock);     /* cache → stats */
    /* ... */
    pthread_mutex_unlock(&stats_lock);
    pthread_mutex_unlock(&cache_lock);
}

static void flush_and_cache(void) {
    pthread_mutex_lock(&stats_lock);     /* stats → cache  ← OPPOSITE ORDER */
    pthread_mutex_lock(&cache_lock);
    /* ... */
    pthread_mutex_unlock(&cache_lock);
    pthread_mutex_unlock(&stats_lock);
}
```

```bash
gcc -Wall -Wextra -pthread -o real_deadlock real_deadlock.c
./real_deadlock & sleep 2
ps -T -p $! -o tid,stat,wchan:20,comm
# both threads S in futex_wait — deadlock
```

The fix is Solution 3: decide that `cache_lock` always precedes `stats_lock`,
and document it at the top of the file.

---

# The Requirements a Solution Must Meet

| Requirement | Meaning | How to check |
|---|---|---|
| **Mutual exclusion** | no two threads in the critical section at once | by construction of the primitive |
| **No deadlock** | the wait-for graph has no cycle | order the acquisitions, or use try-and-back-off |
| **No starvation** | bounded waiting; every thread eventually proceeds | FIFO queue (a semaphore's), or aging |
| **Progress** | if the system can make progress, it eventually does | never have every thread waiting on a condition only they can satisfy |

```c
/* Progress test: are all threads blocked, with nobody able to unblock? */
static int progress_deadlock(void)
{
    int blocked = 0, total = 0;
    for (int i = 0; i < nthreads; i++) {
        total++;
        if (thread_state[i] == BLOCKED_ON_FORK) blocked++;
    }
    return (blocked == total);
}
```

---

# Common Misconceptions

### ❌ "The Dining Philosophers problem is unrealistic, so I do not need to worry about it."

The two-lock version with opposite acquisition orders is one of the most
common causes of production deadlocks. The abstraction is artificial; the
circular wait is real.

---

### ❌ "Using semaphores means I do not need a mutex."

The canonical bounded-buffer solution uses **both**: two semaphores for the
conditions, and a mutex for the data structure itself. A semaphore can protect
data, but then every access needs a `sem_wait`/`sem_post` pair, which is
strictly more error-prone than a scoped mutex.

---

### ❌ "A read-write lock makes my program faster."

Only if there are many concurrent readers. With 2 threads it is slower than a
plain mutex. And it is a starvation risk unless you configure the fairness
policy.

---

### ❌ "These three problems need three different solutions."

They need the same three techniques: **a global order** (no cycles), **a
bounded capacity with blocking** (backpressure and no busy-wait), and **a fair
waiter queue** (no starvation). The three problems are three illustrations.

---

# Interview Questions

### Basic

- State the Producer–Consumer problem and its constraints.
- State the Readers–Writers problem and its three variants.
- State the Dining Philosophers problem.
- What is the difference between a mutex and a semaphore in these solutions?

### Intermediate

- Solve the bounded-buffer problem using semaphores. Why are two semaphores and a mutex needed?
- Why does the naive busy-wait solution of Producer–Consumer fail?
- Explain the three variants of Readers–Writers and which one starves writers.
- Give two solutions to Dining Philosophers, and state which Coffman condition each breaks.

### Advanced

- A bounded buffer with one producer and one consumer, no mutex, only two
  atomics. Is it correct? Under what memory ordering?
- Your rwlock version of Readers–Writers has 1 reader and 1 writer. It is slower
  than a plain mutex. Why, and when is that acceptable?
- Extend the Dining Philosophers solution to 1,000 philosophers. What changes?
- Design the synchronisation for a log-structured storage engine with 64
  background flushers and 1,000 request threads.

---

# University Exam Notes

### Definitions

- **Producer–Consumer (bounded buffer) problem:** N producer threads place
  items into a buffer of size B; M consumer threads remove them. Producers
  block when the buffer is full, consumers when it is empty.
- **Readers–Writers problem:** Any number of readers may access the shared
  data concurrently; a writer has exclusive access. Three variants: reader
  preference, writer preference, and fair.
- **Dining Philosophers problem:** *n* philosophers, *n* forks arranged in a
  circle; each philosopher needs the two adjacent forks to eat. The naive
  solution deadlocks.
- **Circular wait:** A condition in which a cycle exists in the wait-for graph,
  and therefore deadlock is possible.
- **Waiter and arbiter:** A solution pattern in which a separate arbiter grants
  permission to use shared resources, holding nothing itself.

### Frequently Asked Questions

- State and explain the Producer–Consumer problem.
- What are the three variants of the Readers–Writers problem?
- Explain the Dining Philosophers problem and give two solutions.
- What is a circular wait, and how do you prevent it?
- Differentiate reader-preference, writer-preference and fair solutions.

---

# Key Takeaways

- All three problems are one problem: **managing limited shared resources
  under concurrency**.
- **Producer–Consumer**: two counting semaphores (`empty`, `full`) for the
  conditions, plus one mutex for the buffer. The semaphore provides blocking
  and backpressure; the mutex provides exclusion.
- **Readers–Writers**: use a `pthread_rwlock_t` in practice; understand the
  first-principles version. The three variants differ in which class they
  starve.
- **Dining Philosophers**: the naive solution deadlocks. Break the cycle with an
  asymmetric order, a resource hierarchy, or a waiter-and-arbiter.
- The general fix that works for all of them: **a global acquisition order**,
  which makes a cycle impossible by construction.
- Always use `while` around `pthread_cond_wait`, and `broadcast` to the
  condition variable matching the class you want to wake.
- The real-world version of Dining Philosophers is two locks acquired in
  opposite orders in two functions. Document the order.

---

# References

- Operating System Concepts — Silberschatz, §7.5 (Producer–Consumer), §7.6
  (Readers–Writers), §7.7 (Dining Philosophers)
- Modern Operating Systems — Tanenbaum, §2.7
- The Little Book of Semaphores — Allen Downey, chapters 1–3
- POSIX / `pthreads`: `man 3 pthread_rwlock_rdlock`, `man 3 pthread_cond_wait`
- `man 3 sem_init`
- Coffman, Denning, Denning, Horowitz, "System Deadlocks", ACM Computing Surveys 1971
