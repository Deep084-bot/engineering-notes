# Concurrency vs Parallelism

> [!NOTE]
> **Module:** Module III – Concurrency
>
> **Difficulty:** ⭐⭐☆☆☆
>
> **Prerequisites:**
> - Threads
> - CPU Modes (cpu-modes.md)
>
> **Scope:** Linux and C only. No Windows or Java.

---

# Learning Objective

After reading this note, you should be able to:

- Define concurrency and parallelism precisely and separately.
- Explain why the two words describe different things.
- Explain how the operating system provides concurrency on a single core.
- Explain how parallelism arises on multiple cores.
- Explain why concurrency problems exist even on a single-core machine.
- Read and evaluate a program's scaling behaviour.
- Explain the relationship between concurrency, parallelism and Amdahl's law.

---

# Why Do We Need to Distinguish These Words?

## The Problem

"It is concurrent" and "it is parallel" are used interchangeably in job
descriptions, product marketing, and most blog posts.

They are not the same thing, and confusing them leads to real engineering
mistakes:

- Claiming a program "uses all your cores" when it is single-threaded.
- Fixing a **concurrency bug** by adding cores, making it worse.
- Concluding that a program cannot be made concurrent.
- Quoting the wrong formula for speedup.

Concurrency is about **structure**. Parallelism is about **execution**.

---

# Real World Analogy

**Concurrency** is a single chef cooking four dishes.

The chef is one person. Only one dish is being worked on at a time. But the
chef *interleaves* — chopping vegetables for dish 1 while rice boils for dish
2, stirring dish 3 while dish 1's oven runs. All four are "in progress". The
work is concurrent, and no two things happen at the same instant.

**Parallelism** is four chefs, one per dish.

All four dishes are worked on simultaneously. The work is parallel, and it is
also concurrent.

```text
  CONCURRENCY                          PARALLELISM
  ───────────                          ───────────
  1 chef, 4 dishes                     4 chefs, 4 dishes
  interleaved                          simultaneous
  progress without simultaneity        simultaneity

  A concurrent program MAY be parallel
  A parallel program is NECESSARILY concurrent
  A concurrent program need NOT be parallel
```

---

# Intuition

```text
  CONCURRENCY = "structure"
  ─────────────────────────
  The program is decomposed into independently progressing tasks
  whose relative order is not fixed by the program text.

        task A ──┐
        task B ──┼── interleaved in time
        task C ──┘


  PARALLELISM = "execution"
  ─────────────────────────
  Two or more tasks execute during the SAME instant.

        CPU core 0:  │ A │   │ A │   │ A │
        CPU core 1:  │ B │ B │   │   │ B │     ← overlap in time
                                 ▲
                          parallelism happens HERE
```

> [!IMPORTANT]
> Concurrency is a property of the **program**. It is decided when you write
> the code, and it is present whether or not the machine has more than one core.
>
> Parallelism is a property of the **execution**. It depends on the hardware and
> on the scheduler, and the same concurrent program may or may not be parallel.

---

# Formal Definitions

| Term | Definition |
|---|---|
| **Concurrency** | The ability of a program to be structured as multiple independently progressing tasks, where the interleaving of those tasks is not fixed in advance. A property of program design. |
| **Parallelism** | The simultaneous execution of two or more tasks. A property of execution on a given machine. |

```text
  Concurrency   = decomposition  +  interleaving (unordered)
  Parallelism   = decomposition  +  simultaneity
```

## Gobble vs Shower: the distinction that makes it click

```text
  One person, one bowl of candy, eating one at a time
  ──────────────────────────────────────────────────
  GOBBLE     = consume items one after another, all by me
             = CONCURRENT (many items, one consumer, interleaved)
             = NOT parallel

  SHOWER     = consume items one at a time, but I am one of several consumers
               working through the same pile at the same time
             = CONCURRENT and PARALLEL
```

The items are the tasks. The consumers are the cores.

---

# Concurrency Without Parallelism: The Single-Core Machine

A single core can only run one thread of execution at a time. So how can a
program be concurrent?

```text
  Single core, time flows to the right

  │ thread A  ███░░░░░░░░░░░░░░░██░░░░░░░░░░░░░░░░░░░░░██
  │ thread B  ░░░░░░░░░░░░░░░██░░░░░░░░░░░░░░░░░░░░░░██░░░░░░░░
  │ thread C  ░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░██░░░░
  │ scheduler ████████████████████████████████████████████████████
  └──────────┬───────────────────────────────────────────────────┬──►
             0 ms                                               30 ms

  At NO instant are two threads running.
  At every instant MULTIPLE threads are "in progress".
```

This is **concurrency without parallelism**, and it is genuinely useful.

## Why it is useful

```text
  The key: while thread A is BLOCKED, the CPU is not wasted.

  thread A:  compute(1 ms) → read disk (10 ms, BLOCKED) → compute(1 ms)
  thread B:                    compute(11 ms)
  thread C:                    compute(11 ms)

  Single core, 13 ms of compute, 10 ms of overlapped I/O
  Without concurrency:  A takes 12 ms, then B 11 ms, then C 11 ms = 34 ms
  With concurrency:     A and B and C all done in about 13 ms
```

The speedup is not from parallelism. It is from **overlapping I/O with
computation**.

```c
/*
 * 4 files, each requiring a slow read.
 * Single-threaded: total time = sum of all reads.
 * Threaded on 1 core:  total time ≈ the longest single read.
 */
```

## The operating system is the concurrency engine

Concurrency on a single core is implemented entirely by the kernel:

```text
  threads in one program  ──►  scheduler  ──►  one CPU core
       (concurrent)             (time-slices
                                 and switches)
```

The scheduler performs context switches thousands of times per second,
interleaving the threads. No two run at the same instant. The program is
nonetheless concurrent.

---

# Parallelism: The Multi-Core Machine

On a multicore system, threads can run on different cores at the same instant.

```text
  4 cores, 4 threads

  CPU core 0:  │ thread A ████████████████████████████████████│
  CPU core 1:  │ thread B ████████████████████████████████████│
  CPU core 2:  │ thread C ████████████████████████████████████│
  CPU core 3:  │ thread D ████████████████████████████████████│
  └────────────┴─────────────────────────────────────────────┴──►
                ▲
                └── the same instant: all four are running.
                    This is parallelism.
```

```c
#include <stdio.h>
#include <pthread.h>
#include <time.h>

#define NTASKS 4

static double work(int n, int iters)
{
    double x = 1.0;
    for (int i = 0; i < iters; i++)
        x = x * 1.0000001 + (n % 7);
    return x;
}

static void *run(void *arg)
{
    long n = (long)arg;
    work((int)n, 80000000);
    return NULL;
}

static double now(void)
{
    struct timespec t;
    clock_gettime(CLOCK_MONOTONIC, &t);
    return t.tv_sec + t.tv_nsec / 1e9;
}

int main(void)
{
    double t0, t1;

    t0 = now();
    for (long i = 0; i < NTASKS; i++)
        run((void *)i);
    t1 = now();
    printf("serial   : %.3f s\n", t1 - t0);

    pthread_t t[NTASKS];
    t0 = now();
    for (long i = 0; i < NTASKS; i++)
        pthread_create(&t[i], NULL, run, (void *)i);
    for (int i = 0; i < NTASKS; i++)
        pthread_join(t[i], NULL);
    t1 = now();
    printf("threaded : %.3f s\n", t1 - t0);

    return 0;
}
```

```bash
gcc -O2 -pthread -o par par.c
nproc
./par
```

```text
# on an 8-core machine
serial   : 3.104 s
threaded : 0.412 s
speedup  : 7.53x  (near-linear: the work is embarrassingly parallel)
```

```text
# on a 1-core machine (force it)
taskset -c 0 ./par
```

```text
serial   : 3.104 s
threaded : 3.219 s   ← SLOWER, not faster
```

> [!IMPORTANT]
> On a single core, the threaded version is slightly **slower** than serial.
> The threads still do all the work, and now they also pay for context
> switches and for cache and TLB damage. This is the clearest demonstration
> that concurrency and parallelism are different properties.

---

# The Four Relationships

| Program | Machine | Concurrent? | Parallel? |
|---|---|---|---|
| Sequential | single core | no | no |
| Sequential | multi-core | no | no (unless the compiler vectorises or auto-parallelises) |
| **Concurrent** | single core | **yes** | **no** |
| **Concurrent** | multi-core | **yes** | **yes** |

```text
  Sequential  + multicore  + an auto-vectorising compiler
  → the CPU may execute several arithmetic lanes at once.
    That is parallelism, but it is HARDWARE parallelism, not thread parallelism.
```

## The intermediate case: instruction-level parallelism

```c
double a[1000], b[1000], c[1000];

for (int i = 0; i < 1000; i++)
    c[i] = a[i] + b[i];
```

```text
  Single-threaded, single-core. The C source is sequential.

  But a modern x86 core has multiple execution ports and a 512-bit vector
  unit. It can execute FOUR of these additions at once, or 8 floats per
  AVX-512 instruction.

  This is parallelism, without a single thread of yours.
```

```bash
# confirm the vectorisation
gcc -O3 -march=native -fopt-info-vec -o vec c.c
```

```text
c.c:4:5: optimized: loop vectorized using 32 byte vectors
c.c:4:5: optimized: loop vectorized using 64 byte vectors
```

---

# Concurrency Problems Exist Even on One Core

This is the counter-intuitive part that trips people up.

```text
  "There is only one core. How can two threads interfere?"

  Answer: the INTERRUPT happens at any point, including between two
  instructions of your read-modify-write.
```

```c
static int counter = 0;

void increment(void)          { counter++; }     /* run 1,000,000 times */
```

```text
  Thread A: MOV counter → RAX   (reads 5)
            ← TICK INTERRUPT, scheduler switches to B →
  Thread B: MOV counter → RBX   (reads 5)
            ADD 1, RBX
            MOV RBX → counter   (writes 6)
            ← scheduler switches back to A →
  Thread A: ADD 1, RAX
            MOV RAX → counter   (writes 6)

  Final: 6, not 7. On ONE core. The race is real.
```

> [!IMPORTANT]
> A race condition is a defect in the **program**, not a property of the
> hardware. It is a bug on a single core and on sixteen cores. Adding cores does
> not fix it; it makes the window much wider.

---

# Squeezing Parallelism: More Cores Is Not Automatically Better

## The limit: Amdahl's law

```text
  speedup(n) = 1 / ( (1 − p) + p/n )

  p = fraction of the work that is parallelisable
  n = number of processors
```

| p (parallel fraction) | n=2 | n=4 | n=8 | n→∞ |
|---|---|---|---|---|
| 0.50 | 1.33× | 1.60× | 1.78× | **2.00×** |
| 0.90 | 1.82× | 2.31× | 2.71× | **10.00×** |
| 0.95 | 1.90× | 2.56× | 3.17× | **20.00×** |
| 0.99 | 1.98× | 3.80× | 7.01× | **100.00×** |
| 0.999 | 2.00× | 3.98× | 7.94× | **1000.00×** |

At p = 0.95, going from 8 to 64 cores improves the speedup from 3.17× to only
4.82×. You pay for 8 more cores and get 1.5× more throughput.

See [amdahls-law.md](amdahls-law.md) for the full treatment.

## Beyond Amdahl: other losses

| Loss | Effect |
|---|---|
| Synchronisation | Lock contention serialises threads |
| False sharing | Threads invalidate each other's cache lines for no reason |
| Memory bandwidth | More cores contend for the same DRAM |
| Load imbalance | One slow thread sets the finish time |
| Context switch cost | 1–10 µs, paid thousands of times |
| NUMA | Remote memory is 1.5–2× slower than local |

## Measuring scaling in C

```c
#define _GNU_SOURCE
#include <pthread.h>
#include <stdio.h>
#include <time.h>
#include <stdlib.h>

#include <unistd.h>
static double work(double seed, long iters)
{
    double x = seed;
    for (long i = 0; i < iters; i++)
        x = x * 1.0000001 + (i % 7);
    return x;
}

static void *run(void *arg)
{
    work((double)(long)arg, 50000000L);
    return NULL;
}

static double now(void)
{
    struct timespec t;
    clock_gettime(CLOCK_MONOTONIC, &t);
    return t.tv_sec + t.tv_nsec / 1e9;
}

int main(void)
{
    long maxn = sysconf(_SC_NPROCESSORS_ONLN);
    if (maxn > 16) maxn = 16;            /* keep the demo quick */

    /* 1. Baseline: serial */
    double t0 = now();
    for (long i = 0; i < maxn; i++)
        work((double)i, 50000000L);
    double serial = now() - t0;
    printf("serial        : %.3f s  (speedup 1.00)\n", serial);

    /* 2. For each thread count, measure */
    for (long n = 2; n <= maxn; n *= 2) {
        pthread_t *tid = malloc(n * sizeof *tid);
        t0 = now();
        for (long i = 0; i < n; i++)
            pthread_create(&tid[i], NULL, run, (void *)i);
        for (long i = 0; i < n; i++)
            pthread_join(tid[i], NULL);
        double elapsed = now() - t0;

        printf("%2ld threads   : %.3f s  (speedup %.2fx, efficiency %.0f%%)\n",
               n, elapsed, serial / elapsed, 100.0 * serial / elapsed / n);
        free(tid);
    }
    return 0;
}
```

```bash
gcc -O2 -pthread -D_GNU_SOURCE -o scale scale.c
./scale
```

```text
serial        : 4.204 s  (speedup 1.00)
 2 threads    : 2.108 s  (speedup 1.99, efficiency 100%)
 4 threads    : 1.061 s  (speedup 3.96, efficiency 99%)
 8 threads    : 0.545 s  (speedup 7.71, efficiency 96%)
16 threads    : 0.334 s  (speedup 12.59, efficiency 79%)
```

```text
  Efficiency below 100% is the real story:

    4 threads  → 99%  : the work is perfectly parallel and fits in cache
    16 threads → 79%  : the working set no longer fits in L2/L3, so every
                        core is pulling data from shared DRAM.
                        Plus context switch overhead.

  This is not a bug in the program. It is the memory hierarchy.
```

---

# Why Concurrency Is Hard

```text
  1. DETERMINISM
     The interleaving is not fixed, so the same program gives different
     results. A bug appears once in 10,000 runs.

  2. SHARED STATE
     Every shared variable is a potential race. There is no compiler
     warning that reliably finds them.

  3. ATOMICITY
     A single source statement is many machine instructions.
     counter++ is three.

  4. MEMORY ORDERING
     The compiler reorders. The CPU reorders. The store buffer defers.
     C11's memory model exists because this is a real, observable problem.

  5. BLOCKING
     A thread holding a lock and waiting for another is a deadlock.
     No output, no crash, no log. Just a hang.

  6. SCHEDULER BEHAVIOUR
     Your bug only appears under load, when the scheduler interleaves
     differently. It will not reproduce on your machine.
```

## Why it gets worse on many cores

```text
  1 core,  2 threads : 1 possible interleaving order that matters
  1 core,  N threads : N! possible orders
  4 cores, 2 threads : every interleaving of every instruction, from any core
  64 cores, 64 threads: astronomically many

  Testing cannot find these. Reasoning must.
```

---

# Choosing Your Model

| If the program is… | Then it is… |
|---|---|
| Written with `fork()` and pipes, no shared state | Concurrent at the process level |
| Written with `pthread_create()` and shared memory | Concurrent at the thread level |
| Written with `fork()` **and** shared memory | Dangerous: see [threading-issues.md](threading-issues.md) |
| Single-threaded but blocking on I/O | Sequential, and slow |
| Single-threaded with `-O3 -march=native` | Sequential, but internally parallel |
| Using `aio_write()` or `io_uring` | Asynchronous: concurrent without threads |
| A server with an event loop (`epoll`) | Highly concurrent, single-threaded |

```text
  Three ways to get concurrency in C on Linux:

    1. THREADS       pthreads, one per task
                      simplest; needs locks for shared state

    2. PROCESSES     fork + pipe / socket
                      isolation; IPC cost; no shared state to lock

    3. ASYNCHRONY    epoll / io_uring, single-threaded
                      no locks needed; harder to reason about ordering
```

---

# Common Misconceptions

### ❌ "Concurrent means simultaneous."

Incorrect.

Concurrency means *independently progressing*, not *simultaneous*. A single-core
machine runs a concurrent program by switching between its threads. That is
concurrency with zero parallelism.

---

### ❌ "Parallelism implies concurrency, but not the reverse."

The first half is right. The second half is true for **programming models**,
and needs care for **execution**: a single-threaded program with an
auto-vectorising compiler executes in parallel without any concurrency in the
source. So parallelism can also arise from hardware the programmer did not ask
for.

---

### ❌ "If my program is concurrent, more cores will make it faster."

Incorrect, for three reasons.

1. If the parallel fraction p is low, Amdahl's law caps the speedup.
2. If it is already fast, overheads dominate and it gets *slower*.
3. Synchronisation cost grows with core count.

---

### ❌ "Single-threaded code has no concurrency bugs."

Incorrect.

A data race is a bug in the program. It can manifest on a single core, because
an interrupt can arrive between the read and the write of a read-modify-write.
Adding cores widens the window; it does not create the bug.

---

### ❌ "Concurrency is only about speed."

Incorrect.

Concurrency is equally about **structure and responsiveness**: overlapping
network I/O with computation, serving many clients from one process, keeping a
UI responsive during a long download. Most useful concurrent programs are not
faster — they are more responsive.

---

# Interview Questions

### Basic

- Differentiate between concurrency and parallelism with an example.
- Can a program be concurrent but not parallel? Give an example.
- What is the difference in structure between a concurrent and a parallel program?
- How many cores does concurrency require?

### Intermediate

- How does a single-core operating system provide concurrency?
- Why is it useful to overlap I/O with computation even without parallelism?
- What is instruction-level parallelism, and how is it different from thread parallelism?
- Explain Amdahl's law and its implication for adding cores.

### Advanced

- A program is concurrent but gets slower when you add cores. Explain every possible cause.
- How would you measure the parallel efficiency of a program?
- When is an event loop (`epoll`) better than a thread pool?
- Why does data race detection require special tooling rather than testing?

---

# University Exam Notes

### Definitions

- **Concurrency:** The ability of a program to be structured as multiple
  independently progressing tasks whose interleaving is not fixed in advance.
- **Parallelism:** The simultaneous execution of two or more processes or
  threads on a multicore system.
- **Concurrent system:** A system in which multiple computations can progress
  over overlapping intervals of time, without necessarily executing
  simultaneously.
- **Amdahl's Law:** The observation that speedup is limited by the serial
  fraction of the work: `speedup = 1 / ((1−p) + p/n)`.
- **Parallel efficiency:** Speedup divided by the number of processors, as a
  percentage.

### Frequently Asked Questions

- Differentiate between concurrency and parallelism.
- Explain concurrency with an example.
- Is concurrency possible on a single-processor system? Justify your answer.
- What is Amdahl's law? Explain its significance.
- Differentiate between concurrent and parallel programs.
- Explain the problems faced while implementing concurrency.

---

# Key Takeaways

- **Concurrency** is about program *structure*: independently progressing
  tasks with an unfixed interleaving.
- **Parallelism** is about *execution*: two or more tasks in the same instant.
- A concurrent program **need not be parallel**. A single core interleaves
  threads; that is concurrency with no parallelism.
- Concurrency is useful even with one core, because it overlaps **blocking I/O
  with computation**.
- Parallelism arises from multiple cores and also from the CPU itself
  (vectorisation, multiple issue, out-of-order execution).
- Amdahl's law caps the speedup at `1/p`, so adding cores past a point buys
  almost nothing.
- Parallel **efficiency** below 100% is usually memory bandwidth, not logic.
- Concurrency bugs exist on a single core. Adding cores widens the window; it
  does not create the defect.

---

# References

- Operating System Concepts — Silberschatz
- Modern Operating Systems — Tanenbaum and Bos
- The Art of Multiprocessor Programming — Herlihy and Shavit
- "Amdahl's Law" — Gene Amdahl, AFIPS Proceedings, 1967
- `man 3 pthread_create`, `man 2 epoll_create1`, `man 2 clone`
- `nproc`, `lscpu`, `perf stat -e cycles,instructions ./program`
