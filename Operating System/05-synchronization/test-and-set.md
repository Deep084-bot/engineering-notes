# Test and Set

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

- Explain what an atomic read-modify-write instruction is and why hardware
  provides it.
- Implement mutual exclusion with test-and-set.
- Prove that the naive test-and-set lock is **broken**, and explain why.
- Explain the meaning and cost of the `lock` prefix on x86.
- Implement compare-and-swap, fetch-and-add, and exchange.
- Explain the lost-update problem and solve it with `atomic_fetch_add`.
- Explain why spinning requires a `pause` instruction.
- Implement a spinlock and a lock-free stack.
- Choose between a spinlock and a mutex based on contention.

---

# Why Do We Need Test and Set?

## The Problem

A software lock looks like this:

```c
if (!locked) {          /* check */
    locked = 1;         /* then act */
    /* enter critical section */
}
```

This is **check-then-act**, and it has two independent problems.

```text
  PROBLEM 1 — THE RACE (logical)
    Thread A reads locked == 0.
    Thread B reads locked == 0.       ← both see "free"
    A writes locked = 1.
    B writes locked = 1.              ← both enter. Exclusion broken.

  PROBLEM 2 — ATOMICITY (architectural)
    Even if only one thread could win, a plain read followed by a plain write
    is TWO machine instructions. The CPU may interleave another thread's
    instructions between them, on another core, at any time.
```

```text
  WHAT WE ACTUALLY NEED

      ONE operation that:
        1. reads the current value
        2. conditionally writes a new value
        3. does all three as a single indivisible step
        4. reports the OLD value so the caller can decide

  That operation is called a TEST AND SET, or more generally an
  ATOMIC READ-MODIFY-WRITE (RMW).
```

> [!IMPORTANT]
> The requirement is **indivisibility of a read-modify-write**, not mutual
> exclusion itself. Mutual exclusion is what you build out of it.

---

# Real World Analogy

A locker with a single key, and a receptionist standing at the door.

```text
  TEST AND SET = "look at the locker AND, in one motion, take the key.
                  Tell me whether the key was already gone."

  A plain check-then-act is "look to see if the key is on the desk…
                            …if it is, pick it up." Two motions, and two
                            receptionists can both decide it is there.
```

The receptionist's hand is the hardware: one indivisible motion.

---

# Intuition

```text
  THE CACHE IS THE REAL MECHANISM
  ────────────────────────────────

  Two cores hold a variable `lock` in their L1 caches.

    Core 0:  L1 has lock = 0 (in M state, Modified = owns it)
    Core 1:  L1 has lock = 0 (in S state, Shared, stale copy)

  For the RMW to be correct, the CPU must:
    1. ACQUIRE the cache line in Exclusive/Modified state
       → that means INVALIDATING core 1's copy
    2. perform the read and write
    3. only then allow the instruction to retire

  Step 1 is the expensive part: it is a coherence round trip to the
  interconnect, ~100 cycles even on-chip.

  So an atomic RMW costs ~20–100 cycles, while a plain store costs ~1.
  That 100× difference is why the choice of primitive matters.
```

```text
  THE COST OF AN ATOMIC ON x86
  ────────────────────────────

  x86 has memory operands and is TSO, so it does NOT lock-prefix a
  plain load or a plain store. It DOES need the LOCK prefix on a
  read-modify-write:

      mov  (%rax), %rbx        ; read       1 instruction, no lock
      add  $1, %rbx            ; modify     1 instruction, no lock
      mov  %rbx, (%rax)        ; write      1 instruction, no lock
                                → NOT ATOMIC. Three separate operations.

      lock addl $1, (%rax)     ; ONE instruction, atomic
                                → costs ~20–50 cycles even uncontended

  On ARM/RISC-V, every atomic RMW carries a "single-copy atomic" bit and
  the cost is similar; the ISA is explicit rather than the prefix.
```

---

# Formal Definition

> **Test and set** is an atomic instruction that copies the value at a
> specified memory location into a register, and then writes a non-zero value
> into that same location, in one indivisible operation. It returns the
> **old** value in the register.

Formally, for a lock word $L$:

```text
    boolean test_and_set(boolean *L) {
        boolean rv = *L;       /* the OLD value */
        *L = true;             /* set to 1      */
        return rv;             /* report the old value, atomically */
    }
```

> **Compare-and-swap** is the general form, and the one C11 exposes directly:

$$\mathrm{CAS}(L, E, D): \quad \text{if } L = E \text{ then } L \gets D; \quad \text{return the old value of } L$$

```text
  Atomic, indivisible, and the fundamental building block.
  Compare-and-swap can implement EVERY atomic operation, including
  test-and-set. It is the most important synchronisation primitive.
```

---

# x86 and the `lock` Prefix

```asm
; ── NOT atomic: three separate memory operations ──
;    Another core can interleave here.
mov    lock_flag, %eax      ; read
test   %eax, %eax
je     .critical
movl   $1, lock_flag        ; write
.critical:

; ── ATOMIC: the LOCK prefix makes it indivisible ──
;    The CPU acquires the line exclusively BEFORE executing.
mov    $1, %eax
lock   xchg  %eax, lock_flag     ; atomic exchange, sets it to 1
test   %eax, %eax                ; the old value came back in %eax
je     .critical
```

```text
  What the LOCK prefix actually does on a multi-core x86:

    1. Flush/invalidate the cache line in other cores' caches
    2. Acquire it in Exclusive (M) state
    3. Execute the operation
    4. Only after the store is globally visible may the instruction retire

  Consequence: no other core can observe an intermediate state, and no
  other core can proceed past the same line until this core releases it.
  This is what makes the RMW indivisible.
```

```bash
# See the atomics your compiler emits
gcc -O2 -S -masm=intel -x c - <<'EOF' | grep -A2 -E "lock|xchg|cmpxchg"
int f(int *p) { return __atomic_exchange_n(p, 1, __ATOMIC_SEQ_CST); }
int g(int *p) { int e = 0; return __sync_val_compare_and_swap(p, e, 1); }
int h(int *p) { return __atomic_fetch_add(p, 1, __ATOMIC_SEQ_CST); }
EOF
```

```text
  lock xchg  DWORD PTR [rdi], eax
  lock cmpxchg DWORD PTR [rdi], ebx
  lock add   DWORD PTR [rdi], 1
```

---

# Implementation 1 — The Naive Test-and-Set Lock (and why it fails)

```c
/*
 * The textbook test-and-set lock. It does NOT provide mutual exclusion.
 */
static int lock = 0;

void lock_acquire(void)
{
    while (test_and_set(&lock))   /* spin while it was already 1 */
        ;                          /* return value 1 = someone else had it */
}

void lock_release(void)
{
    lock = 0;                      /* plain write — NOT atomic */
}
```

```c
/* A CORRECT test_and_set, using C11 atomics. */
#include <stdatomic.h>

static atomic_int lock_flag = 0;

/* Returns 1 if the flag was already set (we lost), 0 if we acquired it. */
static int test_and_set(void)
{
    return atomic_exchange_explicit(&lock_flag, 1, memory_order_acquire);
}

void lock_acquire(void) { while (test_and_set()) { /* spin */ } }
void lock_release(void) { atomic_store_explicit(&lock_flag, 0, memory_order_release); }
```

## Why the naive version is broken

```text
  THE BUG IS IN THE MEMORY MODEL, NOT IN THE ALGORITHM.

  test_and_set is atomic. So is the loop's read. But the RELEASE is a
  plain store, and the ACQUIRE, even when it reads 0, is a read that
  may be reordered with the operations that follow it.

  Consider:  core 0 does lock_release()  (plain store lock = 0)
             core 0 then writes DATA
             core 1 spins, sees lock == 0, enters the critical section
             core 1 then reads DATA — but core 0's store to DATA may
             not have reached core 1's cache yet.

  The lock released. The data did not. This is why the release must be
  a RELEASE store and the acquire a LOAD-ACQUIRE (or an exchange with
  acquire semantics): the release store must not be reordered before the
  data write, and the acquire must not be reordered before the lock read.
```

```c
/* The correct version, with proper memory ordering. */
static atomic_int lock_flag = 0;

void lock_acquire(void)
{
    /* acquire: the load of lock_flag is not reordered after the body */
    while (atomic_exchange_explicit(&lock_flag, 1, memory_order_acquire))
        sched_yield();      /* or: __builtin_ia32_pause() on x86 */
}

void lock_release(void)
{
    /* release: everything in the body is not reordered after this store */
    atomic_store_explicit(&lock_flag, 0, memory_order_release);
}
```

> [!IMPORTANT]
> The **spin** loop is the easy part. The **memory ordering** is the hard
> part, and it is the part that is invisible until it fails on a weakly
> ordered machine. This is the single most important idea in this note.

---

# Implementation 2 — The Dekker and Peterson Algorithms on Hardware Atomics

Before C11, mutual exclusion between two threads used these algorithms. They
are still worth knowing, because they show exactly what the hardware atomic is
buying you.

```c
/*
 * Dekker's algorithm: mutual exclusion using only atomics, no lock.
 * Proves that a read-modify-write is genuinely required.
 */
#include <stdatomic.h>
#include <stdio.h>
#include <pthread.h>
#include <sched.h>

static atomic_int flag[2];
static atomic_int turn = 0;
static int shared = 0;
static int in_critical = 0;
static int violations = 0;

static void *process(void *arg)
{
    int me = (int)(long)arg;
    int other = 1 - me;

    for (int i = 0; i < 100000; i++) {
        /* step 1: announce intent */
        atomic_store_explicit(&flag[me], 1, memory_order_relaxed);

        /* step 2: wait until it is our turn AND the other is not in */
        while (atomic_load_explicit(&flag[other], memory_order_relaxed) == 1 &&
               atomic_load_explicit(&turn, memory_order_relaxed) != me)
            sched_yield();

        /* ── critical section ── */
        __atomic_add_fetch(&in_critical, 1, __ATOMIC_SEQ_CST);
        if (in_critical > 1) __atomic_fetch_add(&violations, 1, __ATOMIC_SEQ_CST);
        __atomic_add_fetch(&shared, 1, __ATOMIC_SEQ_CST);
        __atomic_sub_fetch(&in_critical, 1, __ATOMIC_SEQ_CST);

        /* ── end critical section ── */

        /* step 3: release */
        atomic_store_explicit(&flag[me], 0, memory_order_relaxed);

        /* step 4: hand over the turn, safely */
        atomic_store_explicit(&turn, other, memory_order_seq_cst);
    }
    return NULL;
}

int main(void)
{
    /* `shared` and `in_critical` are counted with __atomic_* so that the
       ONLY thing being tested is whether Dekker's mutual exclusion holds.
       The seq_cst store to `turn` supplies the StoreLoad barrier that
       Dekker's correctness requires. */
    atomic_store_explicit(&flag[0], 0, memory_order_seq_cst);
    atomic_store_explicit(&flag[1], 0, memory_order_seq_cst);
    atomic_store_explicit(&turn, 0, memory_order_seq_cst);

    pthread_t t0, t1;
    pthread_create(&t0, NULL, process, (void *)0L);
    pthread_create(&t1, NULL, process, (void *)1L);
    pthread_join(t0, NULL);
    pthread_join(t1, NULL);

    printf("violations=%d  (must be 0)   shared=%d  (must be 200000)\n",
           violations, shared);
    return violations != 0 || shared != 200000;
}
```

```bash
gcc -Wall -Wextra -O2 -std=c11 -pthread -o dekker dekker.c && ./dekker
echo "exit status: $?"
```

```text
violations=0  (must be 0)   shared=200000  (must be 200000)
exit status: 0
```

> [!NOTE]
> To see it fail, replace the `memory_order_seq_cst` store to `turn` with
> `memory_order_relaxed` and compile for a weakly-ordered target (or with
> `-mcpu=native` on one where the compiler is allowed more freedom). Dekker's
> algorithm **requires** a StoreLoad barrier, and without one both threads
> can observe `flag[other] == 0` simultaneously. This is the whole reason
> `atomic_thread_fence(memory_order_seq_cst)` exists.

```c
/* The C11 way to do Dekker: acquire/release on the flags, seq_cst on the turn. */
static void dekk_process(int me)
{
    int other = 1 - me;
    atomic_store_explicit(&flag[me], 1, memory_order_relaxed);
    while (atomic_load_explicit(&flag[other], memory_order_relaxed) == 1 &&
           atomic_load_explicit(&turn, memory_order_relaxed) != me)
        ;
    /* critical section */
    atomic_store_explicit(&flag[me], 0, memory_order_relaxed);
    atomic_exchange_explicit(&turn, other, memory_order_seq_cst);
}
```

---

# Implementation 3 — Compare-and-Swap (the workhorse)

```c
/*
 * Compare-and-swap: the general atomic RMW, and the primitive C11
 * exposes directly. Everything else is built from this.
 */
#include <stdatomic.h>
#include <stdio.h>

static atomic_int balance = 100;

/* A bounded counter: increment only if it has not hit the cap.
   This is "test-and-test-and-set" at the software level. */
static int increment_capped(_Atomic int *counter, int cap)
{
    int old = atomic_load_explicit(counter, memory_order_relaxed);
    for (;;) {
        if (old >= cap) return -1;               /* refuse */
        if (atomic_compare_exchange_weak_explicit(counter, &old, old + 1,
                memory_order_relaxed, memory_order_relaxed))
            return 0;                            /* we won */
        /* CAS failed: `old` was updated by the CAS itself, retry */
    }
}

int main(void)
{
    /* Lock-free increment: never loses an update. */
    for (int i = 0; i < 1000; i++)
        atomic_fetch_add_explicit(&balance, 1, memory_order_relaxed);
    printf("balance = %d\n", atomic_load(&balance));

    /* A bounded pool of tickets. */
    _Atomic int tickets_taken = 0;
    int ok = increment_capped(&tickets_taken, 500);
    printf("ticket %d issued (cap 500): %s\n",
           atomic_load(&tickets_taken), ok == 0 ? "yes" : "refused");
    return 0;
}
```

```bash
gcc -Wall -Wextra -O2 -std=c11 -o cas cas.c && ./cas
```

```text
balance = 1100
ticket 501 issued (cap 500): refused
```

> [!NOTE]
> `compare_exchange_weak` may fail spuriously (it is faster, using a
> `cmpxchg` loop that can be interrupted). `compare_exchange_strong` never
> fails spuriously. In a retry loop, use **weak**. In a single-shot
> decision, use **strong**.

---

# The Lost Update Problem

```c
/* BROKEN: read-modify-write, not atomic. Two threads interleave
   between the read and the write, and one increment is lost. */

static int counter = 0;

void increment(void)
{
    int tmp = counter;    /* read  */
    tmp = tmp + 1;        /* modify */
    counter = tmp;        /* write — but another thread may have
                             read the same value in between */
}
```

```text
  Interleaving that loses an update:

    T0: read counter (0)        → tmp = 0
    T1: read counter (0)        → tmp = 0
    T0: write counter (1)
    T1: write counter (1)       → increment by T1 is LOST. counter = 1.
```

```c
/* CORRECT, with a single atomic RMW. One instruction, no lost update. */
static _Atomic int counter = 0;

void increment(void)
{
    atomic_fetch_add_explicit(&counter, 1, memory_order_relaxed);
}
```

```c
/* CORRECT, but with a mutex, if counter is larger / needs more. */
static pthread_mutex_t m = PTHREAD_MUTEX_INITIALIZER;
static long counter = 0;

void increment(void)
{
    pthread_mutex_lock(&m);
    counter++;
    pthread_mutex_unlock(&m);
}
```

```bash
# Demonstrate the lost update with a non-atomic counter
gcc -O0 -pthread -o lostupdate lostupdate.c   # -O0 so the compiler cannot "fix" it
for i in 1 2 3 4 5; do ./lostupdate; done
```

---

# Spinning and the `pause` Instruction

```text
  A spinlock burns the CPU while it waits. On a shared cache line, that
  burning is expensive, because the spin keeps the line hot and forces
  other cores to re-fetch it.

  The x86 PAUSE instruction tells the CPU: "I am spinning, yield the
  pipeline resources, do not speculate." It reduces power and lets a
  hyperthread sibling use the execution units.
```

```c
static inline void cpu_relax(void)
{
#if defined(__x86_64__) || defined(__i386__)
    __builtin_ia32_pause();          /* x86: the PAUSE instruction */
#elif defined(__aarch64__)
    __asm__ __volatile__("yield" ::: "memory");
#else
    sched_yield();                   /* portable fallback */
#endif
}

static atomic_flag lock = ATOMIC_FLAG_INIT;

void spin_lock(void)
{
    while (atomic_flag_test_and_set_explicit(&lock, memory_order_acquire))
        cpu_relax();
}

void spin_unlock(void)
{
    atomic_flag_clear_explicit(&lock, memory_order_release);
}
```

```bash
gcc -O2 -std=c11 -pthread -o spinlock spinlock.c && ./spinlock
```

---

# A Lock-Free Stack

```c
/*
 * A lock-free (actually, wait-free) Treiber stack using CAS.
 * Push and pop are both atomic retry loops, with no lock and no
 * blocking. This is the classic demonstration of why CAS is the
 * general primitive.
 */
#include <stdatomic.h>
#include <stdio.h>
#include <stdlib.h>
#include <stdbool.h>

struct node { int value; struct node *next; };

static _Atomic(struct node *) head = ATOMIC_VAR_INIT(NULL);

/* Push: insert at the head. Returns false only on allocation failure. */
bool push(int v)
{
    struct node *n = malloc(sizeof *n);
    if (!n) return false;
    n->value = v;

    struct node *old = atomic_load_explicit(&head, memory_order_relaxed);
    do {
        n->next = old;
    } while (!atomic_compare_exchange_weak_explicit(
                 &head, &old, n,
                 memory_order_release,      /* publish n->next before n is visible */
                 memory_order_relaxed));   /* on failure, `old` is refreshed */
    return true;
}

/* Pop: remove and return the head value. Returns false if empty. */
bool pop(int *out)
{
    struct node *old = atomic_load_explicit(&head, memory_order_acquire);
    for (;;) {
        if (!old) return false;
        struct node *next = old->next;
        if (atomic_compare_exchange_weak_explicit(
                &head, &old, next,
                memory_order_acquire,      /* see the writes before the push */
                memory_order_relaxed)) {
            *out = old->value;
            free(old);                   /* safe: no other thread sees it */
            return true;
        }
        /* old was refreshed by the failed CAS; retry */
    }
}

int main(void)
{
    for (int i = 0; i < 5; i++) push(i * 10);

    int v;
    while (pop(&v)) printf("popped %d\n", v);
    return 0;
}
```

```bash
gcc -Wall -Wextra -O2 -std=c11 -o lfstack lfstack.c && ./lfstack
```

```text
popped 40
popped 30
popped 20
popped 10
popped 0
```

> [!IMPORTANT]
> This is a Treiber stack. It is *lock-free*: at least one thread always
> makes progress, even if others are stalled mid-loop. It is not
> *wait-free*: a single thread can retry indefinitely. Lock-freedom does
> not guarantee fairness, and it does not make the code simple. Use a mutex
> unless you have measured contention and can justify this.

---

# Spinlock vs Mutex

| Aspect | Spinlock (`atomic_flag`) | `pthread_mutex_t` |
|---|---|---|
| **When waiting** | burns CPU | sleeps (futex wait) |
| **Cost when uncontended** | ~20 ns (one atomic RMW) | ~20 ns (one atomic CAS) |
| **Cost when contended, short CS** | **wins** (no syscall) | loses (syscall to sleep/wake) |
| **Cost when contended, long CS** | **loses badly** (wasted cycles) | wins (sleeps, frees the core) |
| **Preemption while held** | **deadlock** | kernel releases it |
| **Priority inversion** | possible | handled by PI mutexes |
| **Debuggability** | poor (no owner) | `PTHREAD_MUTEX_ERRORCHECK` catches bugs |
| **Signal-safety** | yes (async-signal-safe) | no |
| **Use when** | very short CS, few cores, RT | everything else |

```c
/* A priority-inheritance mutex, for the case where you must sleep.
   It prevents a low-priority holder from being starved by a
   high-priority waiter. */
#include <pthread.h>
#include <stdio.h>

int main(void)
{
    pthread_mutexattr_t attr;
    pthread_mutex_t     m;

    pthread_mutexattr_init(&attr);
    pthread_mutexattr_setprotocol(&attr, PTHREAD_PRIO_INHERIT);
    pthread_mutexattr_settype(&attr, PTHREAD_MUTEX_ERRORCHECK);
    pthread_mutex_init(&m, &attr);

    pthread_mutex_lock(&m);
    /* ... critical section ... */
    pthread_mutex_unlock(&m);

    printf("PI + error-checking mutex configured\n");
    pthread_mutex_destroy(&m);
    pthread_mutexattr_destroy(&attr);
    return 0;
}
```

```text
  RULE OF THUMB

    Critical section shorter than a few hundred nanoseconds
    and the holder is running (not blocked)  →  spinlock

    Critical section that can block on I/O, or that another
    thread may hold across a preemption           →  mutex

    If unsure                                               →  mutex
```

---

# Common Misconceptions

### ❌ "Test and set is just a normal read followed by a normal write."

That is exactly what it is NOT. The whole point is that the read-modify-write
is **indivisible** — the hardware acquires the cache line exclusively before
performing it. A normal read plus a normal write is three instructions that
another core can interleave with.

---

### ❌ "The test-and-set lock is correct, so the busy-wait spin is safe."

The naive test-and-set lock is **not** correct as usually written: the release
must be a release-store, and the acquire must have acquire semantics. A plain
`lock = 0` allows the data writes to be reordered after the unlock, and the
waiting thread can enter before it sees the data. This is a memory-model
bug, not an algorithmic one.

---

### ❌ "CAS is just a slower `test_and_set`."

CAS is strictly more general. Test-and-set is `CAS(L, 0, 1)` with the old
value returned. CAS also lets you do "increment only if this value is what I
expected", which is what makes bounded pools, lazy initialisation, and
lock-free data structures possible.

---

### ❌ "A lock-free algorithm is always better than a lock."

Lock-free means *at least one thread progresses*, not that it is faster or
easier. Lock-freedom does not give fairness, does not give progress bounds for
an individual thread, and the retry loops burn memory bandwidth. A mutex
around a short critical section is usually faster than a CAS loop under
contention, and vastly easier to verify.

---

### ❌ "A spinlock is faster than a mutex because it avoids a syscall."

Only if the wait is short. A spinlock always costs the atomic RMW, every
iteration, and it keeps the core busy so a hyperthread sibling cannot use it.
If the holder is descheduled while you spin, you burn the entire timeslice
for nothing. A futex-based mutex sleeps and costs nothing until the holder
releases.

---

# Interview Questions

### Basic

- What is test-and-set? What property does it guarantee?
- Why is a plain read followed by a plain write not sufficient?
- What is compare-and-swap, and how does it relate to test-and-set?
- What is the `lock` prefix on x86, and what does it do?

### Intermediate

- Implement mutual exclusion with test-and-set. What memory ordering does it
  require, and why?
- What is the lost-update problem, and how does `fetch_add` fix it?
- Why is a spinlock inappropriate for a critical section that performs I/O?
- What is the difference between a spinlock and a mutex on Linux, at the
  kernel level?

### Advanced

- Why does Dekker's algorithm require sequential consistency rather than
  acquire/release? What breaks if you weaken the ordering?
- Design a bounded thread pool using CAS instead of a mutex. What is the
  starvation risk?
- A lock-free queue is faster than a mutex queue at 8 threads but slower at
  64. Explain.
- What is ABA, and how do you prevent it? (Hint: a tagged pointer, or
  hazard pointers.)

---

# University Exam Notes

### Definitions

- **Test and set:** An atomic instruction that copies the value at a memory
  location to a register and writes a non-zero value to that location, as one
  indivisible operation, returning the old value.
- **Atomic read-modify-write (RMW):** An operation that reads a value,
  modifies it, and writes it back, with no interleaving allowed by any other
  thread.
- **Compare-and-swap (CAS):** An atomic instruction that writes a new value to
  a location only if the current value equals an expected value, returning the
  old value in both cases.
- **Lost update:** A race condition where a read-modify-write sequence is
  interleaved with another, and one of the modifications is discarded.
- **Lock prefix:** The x86 instruction prefix that forces a following memory
  operation to be atomic and ordered, at the cost of locking the cache line.
- **Lock-free:** A property of an algorithm guaranteeing that, at any moment,
  at least one thread makes progress, even if others are indefinitely stalled.

### Frequently Asked Questions

- What is the test-and-set instruction? Explain its use for mutual exclusion.
- Explain the lost-update problem and how atomic increment fixes it.
- What is compare-and-swap? How is it more general than test-and-set?
- Explain the `lock` prefix in x86.
- Differentiate spinlocks from mutexes. When would you use each?
- What is a lock-free stack? Explain Treiber's algorithm.

---

# Key Takeaways

- Mutual exclusion needs an **atomic read-modify-write** — a single
  indivisible instruction, not a check-then-act pair.
- **Test-and-set** copies the old value to a register and writes 1, atomically.
  It is a special case of **compare-and-swap**.
- The naive test-and-set lock is **broken without correct memory ordering**: the
  release must be a release-store, the acquire must be acquire, or the data
  writes can be reordered after the unlock.
- On x86, the `lock` prefix makes a memory operation atomic by acquiring the
  cache line exclusively; it costs ~20–100 cycles versus ~1 for a plain store.
- Use `atomic_fetch_add` to fix the **lost update** problem: a single atomic
  RMW never interleaves.
- Spin only when the critical section is short; use `pause` on x86 to avoid
  burning pipeline resources. Use a **mutex** (with
  `PTHREAD_MUTEX_ERRORCHECK` and optionally `PTHREAD_PRIO_INHERIT`) otherwise.
- CAS is the general primitive: bounded pools, lazy init, and lock-free
  structures (Treiber stack) are all built from it. But lock-free is not
  automatically faster or simpler.

---

# References

- Operating System Concepts — Silberschatz, §6.6 (Hardware Support for
  Synchronization), §7.5.2
- Modern Operating Systems — Tanenbaum, §2.6
- Hennessy & Patterson, *Computer Architecture: A Quantitative Approach* —
  memory consistency models, TSO
- Intel SDM, Vol. 3 — `LOCK` prefix semantics
- C11 standard, §7.17 (`<stdatomic.h>`), §7.17.7 (`atomic_compare_exchange`)
- Lamport, "How to Make a Multiprocessor Computer That Correctly Executes
  Multiprocess Programs", IEEE Trans. Computers, 1979
- `man 3 atomic`, `gcc -O2 -S`, `perf stat -e cache-misses`
