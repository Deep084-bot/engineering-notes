# Critical Section

> [!NOTE]
> **Module:** Module I – Introduction
>
> **Difficulty:** ⭐⭐⭐☆☆☆
>
> **Prerequisites:**
> - Modes of Operation (cpu-modes.md)
> - Context Switching
>
> **Scope:** Linux and C only. No Windows or Java.

---

# Learning Objective

After reading this note, you should be able to:

- Define a critical section and a critical region.
- Explain what a race condition is and why critical sections exist.
- State the three formal requirements of a correct critical-section solution.
- Identify the critical section in a real piece of C code.
- Explain mutual exclusion, progress and bounded waiting.
- Explain the difference between a critical section and a critical region.
- Give a concrete example where failing to protect a critical section corrupts data.

---

# Why Do We Need a Critical Section?

## The Problem

Two threads both do this:

```c
counter = counter + 1;
```

It looks like one operation. It is not.

```text
  counter = counter + 1  really means:

      MOV  counter, RAX     ← 1. read
      ADD  1, RAX           ← 2. modify
      MOV  RAX, counter     ← 3. write back
```

Three steps. If another thread runs between them, the read-modify-write is
torn.

```text
  Initial:  counter = 5

  Thread A              Thread B
  ─────────              ─────────
  MOV  counter → RAX = 5
                          MOV  counter → RBX = 5
  ADD  1, RAX → 6
                          ADD  1, RBX → 6
  MOV  RAX → counter = 6
                          MOV  RBX → counter = 6

  Final: counter = 6

  Expected: 7        ← ONE update was lost
```

Two increments were performed. Only one survived.

This is a **race condition**, and the code that caused it is a **critical
section** that was not protected.

---

# Real World Analogy

A bank has 4,000 pounds in a vault.

Two tellers both take a customer to "add 100 pounds".

There is a paper ledger, and it has one line: `Vault: 4000`.

```text
  Teller A reads "4000"
  Teller B reads "4000"          ← both read before either writes
  Teller A writes "4100"
  Teller B writes "4100"
```

The bank is now 100 pounds short, and there is no record of where the money
went.

The fix is not better arithmetic. It is a rule: **only one person may touch the
ledger at a time.**

That rule plus the section of work it governs is exactly a critical section
and mutual exclusion.

---

# Intuition

```text
  Every process that shares data has this structure:

  ┌─────────────────────────────────────────────┐
  │  NON-CRITICAL SECTION                       │
  │  reading input, computing, printing          │
  │  (safe to run concurrently)                  │
  ├─────────────────────────────────────────────┤
  │  ► ENTER CRITICAL SECTION                    │
  │  acquire the lock                            │
  ├─────────────────────────────────────────────┤
  │  ► CRITICAL SECTION                          │
  │  read shared data                           │
  │  modify shared data                         │
  │  write shared data                          │
  │  ← must be atomic as a whole                │
  ├─────────────────────────────────────────────┤
  │  EXIT CRITICAL SECTION                       │
  │  release the lock                            │
  ├─────────────────────────────────────────────┤
  │  NON-CRITICAL SECTION                       │
  └─────────────────────────────────────────────┘
```

The critical section is the window where shared data is *unstable* — partway
through a change — and therefore must not be interleaved.

---

# Formal Definition

> **Critical section:** A segment of code in a process that accesses shared
> resources — variables, files, devices — and must not be executed
> concurrently by more than one process or thread.

| Term | Definition |
|---|---|
| **Critical resource** | A resource that permits only one access at a time |
| **Critical section** | The code segment accessing a critical resource |
| **Critical region** | The *time interval* during which a process is inside its critical section |
| **Mutual exclusion** | No two processes are in their critical sections at the same time |
| **Entry section** | Code acquiring the right to enter |
| **Exit section** | Code releasing that right |

```text
  Critical SECTION = a region of code        (source-level, static)
  Critical REGION = a period of time         (runtime, dynamic)

  A process has ONE critical section in its code.
  It may be INSIDE that section, or outside it, over time.
```

---

# Requirements of a Correct Solution

Any mechanism that protects a critical section must satisfy **three**
requirements. If a solution fails any one, it is broken.

## 1. Mutual Exclusion

> At most one process may be inside its critical section at any time.

```text
  P1:  ┃ CRIT ┃──────────┃
  P2:  ──────┃ CRIT ┃────────────

  ✔ never overlapping
```

This is the primary requirement. Everything else is a refinement of it.

## 2. Progress

> If no process is in its critical section, and some process wants to enter, then
> only that process's own next step is executed — it is allowed to proceed.

```text
  Both free. P1 wants in.
  ✔ P1 enters (no arbitrary delay)
```

Progress is what stops a solution from deadlocking by refusing to let anyone in.

## 3. Bounded Waiting

> If a process wants to enter its critical section, there is a limit on the
> number of times other processes may enter before it is allowed in.

```text
  P1 waits. P2 enters. P3 enters. ... P1 is in after a bounded number of
  entries — not an unbounded one.
```

Without bounded waiting, a low-priority process can be starved forever.

| Requirement | Prevents | Analogy |
|---|---|---|
| Mutual exclusion | Corruption | Only one person in the vault |
| Progress | Deadlock | Someone must be let in |
| Bounded waiting | Starvation | The queue is finite and fair |

---

# Finding the Critical Section in Real Code

## Example 1 — A global counter

```c
static long total = 0;                 /* shared, global */

void add_to_total(long n)
{
    total += n;                        /* ◄── THE critical section */
}
```

One statement, but three machine instructions, and therefore not atomic.

## Example 2 — A non-reentrant function

```c
static char buffer[64];
static int  used = 0;

int append(const char *s)
{
    int n = strlen(s);                          /* uses the shared buffer */

    if (used + n >= sizeof buffer)
        return -1;

    memcpy(buffer + used, s, n);                /* ◄── THE critical section */
    used += n;
    return n;
}
```

The `strlen` is fine on its own, but the check-then-act sequence
(`used + n >= size`, then `used += n`) is a race: two threads can both pass the
check and both write.

## Example 3 — A bank transfer

```c
void transfer(Account *from, Account *to, long amount)
{
    from->balance -= amount;      /* ◄─┐ */
    to->balance   += amount;      /*  ├─ one critical section, BOTH
                                     │  statements, or money is created */
}                                  /* ◄─┘ */
```

If a thread is preempted between the two statements, and the process is killed,
money has been destroyed.

## The pattern to look for

```text
  ⚠  read a shared value
  ⚠  test it
  ⚠  modify it
  ⚠  write it back
        ↓
     all four must be inseparable
```

The classic signature is **check-then-act**: a conditional followed by an
action that depends on the conditional's result.

---

# Protecting a Critical Section

## The Mechanism

```text
  ┌────────────────────────────────────────────────┐
  │  NON-CRITICAL SECTION                          │
  ├────────────────────────────────────────────────┤
  │  ENTRY: wait until lock is free                │
  │         acquire lock                           │
  ├════════════════════════════════════════════════┤
  │  CRITICAL SECTION  —  one at a time only      │
  ├════════════════════════════════════════════════┤
  │  EXIT:  release lock                           │
  ├────────────────────────────────────────────────┤
  │  NON-CRITICAL SECTION                          │
  └────────────────────────────────────────────────┘
```

## C with a mutex

```c
#include <pthread.h>

static pthread_mutex_t lock = PTHREAD_MUTEX_INITIALIZER;
static long total = 0;

void add_to_total(long n)
{
    pthread_mutex_lock(&lock);        /* ENTRY  */
    total += n;                       /* CRITICAL SECTION */
    pthread_mutex_unlock(&lock);      /* EXIT   */
}
```

## C with a binary semaphore

```c
#include <semaphore.h>

static sem_t lock;

void init(void) { sem_init(&lock, 0, 1); }
void destroy(void) { sem_destroy(&lock); }

void add_to_total(long n)
{
    sem_wait(&lock);                  /* P() — acquire */
    total += n;
    sem_post(&lock);                  /* V() — release */
}
```

## C11 atomics — when a lock is unnecessary

```c
#include <stdatomic.h>

static _Atomic long total = 0;

/* No lock needed: fetch_add is a single atomic RMW instruction. */
void add_to_total(long n)
{
    atomic_fetch_add_explicit(&total, n, memory_order_relaxed);
}

long read_total(void)
{
    return atomic_load_explicit(&total, memory_order_relaxed);
}
```

```text
  On x86-64 this compiles to:
      LOCK XADD [rip+total], rax
  one instruction, indivisible
```

> [!TIP]
> Not every shared variable needs a mutex. A single atomic operation on one
> word usually does not. A lock is needed when the update spans several
> operations, several variables, or an invariant across them.

---

# Critical Section vs Mode Switch

Both are sometimes loosely called "a switch into the kernel". They are
different.

| | Critical section | Mode switch |
|---|---|---|
| Purpose | Prevent interleaving | Change privilege level |
| Scope | A code region | An instruction boundary |
| Enforced by | Synchronisation primitives | CPU hardware |
| Who participates | Two or more cooperating threads | One process, two privilege levels |
| Cost | Microseconds (futex) or nanoseconds (uncontended) | A few cycles |
| Failure mode | Race condition | Illegal instruction fault |

---

# Nested Critical Sections

Real code has functions that call other functions that also need the lock.

```c
static pthread_mutex_t acct_lock = PTHREAD_MUTEX_INITIALIZER;

void log_transfer(Account *a, long amount)   /* needs the lock too */
{
    pthread_mutex_lock(&acct_lock);
    fprintf(stderr, "transfer %ld from %s\n", amount, a->name);
    pthread_mutex_unlock(&acct_lock);
}

void transfer(Account *from, Account *to, long amount)
{
    pthread_mutex_lock(&acct_lock);
    from->balance -= amount;
    to->balance   += amount;
    log_transfer(from, amount);     /* ◄── DEADLOCK: locks the same
                                       non-recursive mutex again */
    pthread_mutex_unlock(&acct_lock);
}
```

## Solutions

| Solution | How | Cost |
|---|---|---|
| **Recursive mutex** | `PTHREAD_MUTEX_RECURSIVE` | Small bookkeeping per lock |
| **Internal `__thread` guard** | a per-thread flag, skip re-locking | No atomic op in the fast path |
| **Restructure** | pass locked state explicitly, split into `_locked` variants | Best, no runtime cost |
| **A coarse global lock** | one mutex for all account code | Simple, hurts concurrency |

```c
/* The internal guard — the cleanest general fix */
static pthread_mutex_t acct_lock = PTHREAD_MUTEX_INITIALIZER;
static __thread int acct_depth = 0;

static void acct_lock_acquire(void)
{
    if (acct_depth == 0)
        pthread_mutex_lock(&acct_lock);
    acct_depth++;
}

static void acct_lock_release(void)
{
    if (--acct_depth == 0)
        pthread_mutex_unlock(&acct_lock);
}
```

---

# Critical Sections in the Kernel

The same concept, with different primitives, because the kernel has stricter
rules.

| Primitive | Sleeps while held? | Use for |
|---|---|---|
| **Spinlock** (`spinlock_t`) | never | Very short sections; cannot sleep (interrupt context) |
| **Mutex** (`struct mutex`) | yes | Long sections in process context |
| **Read-write semaphore** (`rw_semaphore`) | yes | Many readers, few writers |
| **Atomic ops** | never | Single-word operations |
| **Seqlock** (`seqcount_t`) | reader never sleeps | Read-mostly data, writers are rare |
| **RCU** | reader never sleeps | Read-mostly data structures, e.g. routing tables |

```c
/* Kernel style: pick by context and duration, not by habit. */

spinlock_t    queue_lock;      /* IRQ handlers touch it → must be a spinlock */
mutex_t       device_mutex;    /* long sleeps → mutex */
rw_semaphore  table_sem;       /* lookups dominate → rwsem */
```

## Why the choice matters

```c
/* WRONG: holding a spinlock across a sleepable operation */
spin_lock(&lock);
mutex_lock(&other);      /* may sleep → kernel BUG: sleeping while holding a spinlock */
spin_unlock(&lock);

/* WRONG: using a mutex in interrupt context */
mutex_lock(&lock);       /* the interrupt may already hold it → self-deadlock */
spin_lock(&lock);
mutex_unlock(&lock);
```

The rule is simple: **spinlocks for interrupt-reachable code, mutexes for
sleepable work.**

---

# Critical Sections and the CPU Cache

A critical section is a correctness requirement. It is also a performance
requirement, because it improves cache locality.

```text
  Unprotected, two threads updating two variables on different cache lines:

      CPU 0 cache:  [total: ...]     [count: ...]
      CPU 1 cache:  [total: ...]     [count: ...]
                   ↖ each core's copy is stale as soon as the other writes

  Protected by one mutex:

      CPU 0 cache:  [total: count: ....]  ← one shared line
      CPU 1 cache:  [same line, invalidated]
                   ↖ the lock itself is the shared line — unavoidable
```

## False sharing

```c
struct counters {
    long a;    /* frequently written */
    long b;    /* frequently written */
    long c;    /* frequently written */
};

static struct counters c;
```

```text
  Cache line = 64 bytes, so a, b and c share ONE line.

  CPU 0 writes a → the whole line is invalidated on CPU 1
  CPU 1 writes b → the whole line is invalidated on CPU 0
  CPU 0 writes c → the whole line is invalidated on CPU 1

  Three unrelated variables, three useless invalidations per round.
```

```c
/* Fix: pad each counter onto its own cache line */
struct counters {
    long a;
    char pad[64 - sizeof(long)];
    long b;
    char pad2[64 - sizeof(long)];
    long c;
} __attribute__((aligned(64)));

static struct counters c;
```

```bash
perf c2c record -g ./program
perf c2c report
# look for "Shared Data Cache Line" false-sharing hits
```

> [!IMPORTANT]
> True sharing (one mutex protecting many variables) is a correctness cost
> you must pay. False sharing (one cache line holding variables that are not
> logically related) is a pure performance cost you can design away.

---

# Common Misconceptions

### ❌ "A critical section is the same as a critical resource."

Incorrect.

A critical **resource** is the thing being protected — a file, a variable, a
device. A critical **section** is the code that touches it. Several processes
may each have a critical section over the same resource.

---

### ❌ "`x++` is a single operation, so it is atomic."

Incorrect.

`x++` is a read-modify-write: at least two instructions. The C standard does
not make it atomic, and the compiler may split it further. Use
`atomic_fetch_add()` if you need that.

---

### ❌ "Only shared data needs protection."

Incorrect.

Any shared *invariant* needs protection, even if no single variable is shared.
Two variables whose relationship must hold (`x < y`) need a lock even if each is
written by only one thread — unless you use a single atomic that encodes both.

---

### ❌ "A critical section always needs a mutex."

Incorrect.

A single-word read-modify-write can be done with a C11 atomic in one
instruction, with no lock and no futex. Locks cost microseconds; atomics cost
nanoseconds. Use the cheapest correct primitive.

---

### ❌ "Holding a lock longer is always worse."

Incorrect.

Holding a lock longer reduces concurrency but increases the number of atomic
operations, and can actually be faster when it removes cache-line ping-ponging.
The right measure is throughput under contention, not lock hold time in
isolation.

---

# Interview Questions

### Basic

- What is a critical section? Why is it needed?
- What is a race condition?
- State the three requirements of a critical-section solution.
- Differentiate between a critical section and a critical region.

### Intermediate

- Identify the critical section in a given piece of code and explain why.
- What is check-then-act, and why is it dangerous?
- What is the difference between a critical section and a mode switch?
- How do you handle nested critical sections?

### Advanced

- When would you choose an atomic over a mutex? Give concrete criteria.
- What is false sharing, and how do you detect and fix it?
- Why do spinlocks and mutexes have different rules about sleeping?
- How would you find the critical sections in a large codebase?
- A single-threaded program has a bug that only appears in a race. How do you find it?

---

# University Exam Notes

### Definitions

- **Critical section:** A segment of code that accesses shared, unprotected
  resources and must not be executed concurrently.
- **Critical region:** The time interval during which a process is inside its
  critical section.
- **Mutual exclusion:** The condition that at most one process is inside its
  critical section at a time.
- **Race condition:** A bug in which the result depends on the unpredictable
  interleaving of thread operations.
- **Check-then-act:** The pattern of testing a condition and then acting on the
  result, unsafe without mutual exclusion.

### Frequently Asked Questions

- What is a critical section? Explain with a diagram.
- What is a race condition? Give an example.
- State and explain the three requirements of critical-section problems.
- Differentiate between mode switch and critical section.
- How do you protect a critical section in C?
- What is the difference between a spinlock and a mutex?
- What is the significance of nested critical sections?

---

# Key Takeaways

- A critical section is code that touches shared data; it must not interleave.
- The read-modify-write pattern (`counter++`) is the canonical example.
- Three requirements: **mutual exclusion**, **progress**, **bounded waiting**.
- Critical **section** is code; critical **region** is time.
- Check-then-act is the shape of almost every critical-section bug.
- Use the cheapest correct primitive: an atomic for one word, a mutex for
  anything spanning several operations.
- Kernel rules differ: spinlocks must not sleep, mutexes may.
- Watch for **false sharing** — unrelated variables on one cache line destroy
  parallel scaling.

---

# References

- Operating System Concepts — Silberschatz
- Modern Operating Systems — Tanenbaum and Bos
- The Little Book of Semaphores — Allen Downey (free online)
- `man 3 pthread_mutex_lock`, `man 3 sem_wait`
- C11 standard, §7.17 — `<stdatomic.h>`
- Linux Kernel Documentation — `locking/locktypes.rst`
