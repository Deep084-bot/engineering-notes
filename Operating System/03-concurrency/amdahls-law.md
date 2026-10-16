# Amdahl's Law

> [!NOTE]
> **Module:** Module III – Concurrency
>
> **Difficulty:** ⭐⭐☆☆☆
>
> **Prerequisites:**
> - Concurrency vs Parallelism
>
> **Scope:** Linux and C only. No Windows or Java.

---

# Learning Objective

After reading this note, you should be able to:

- State Amdahl's law and define every term in the formula.
- Compute the speedup for a given parallel fraction and processor count.
- Explain why the speedup is bounded by `1/p`.
- Explain the assumption of fixed problem size and its consequence.
- Compare Amdahl's law with Gustafson's law.
- Apply the law to a real program and identify the serial bottleneck.
- Explain how this law shaped real hardware and software design.

---

# Why Do We Need a Law About Speedup?

## The Problem

A manager asks: "We can afford 64 cores. If we parallelise the program, how
much faster will it be?"

The intuitive answer is 64 times. The honest answer is almost never that.

The reason is embarrassingly simple: **not all of the program can be
parallelised**.

There is always some part that must happen in sequence:

- Reading the input file
- Parsing a header
- Merging the results
- Allocating shared state
- Talking to a device over a serial link

And that serial part does not get faster when you add cores. It stays exactly
the same.

Amdahl's law is the formula that tells you exactly how much that costs you.

---

# Real World Analogy

A bakery makes 100 loaves.

Four bakers work in parallel. But the recipe book must be read first, and the
oven door must be opened at the end. Nobody can skip those steps, and nobody
can do them faster by standing around.

| | Time (minutes) | Parallelisable? |
|---|---|---|
| Read the recipe | 10 | no — 1 person, serial |
| Knead and shape | 60 | yes — 4 bakers |
| Bake (oven) | 30 | no — one oven |
| Box and label | 20 | yes — 4 bakers |

```text
  Serial:     10 + 60 + 30 + 20 = 120 min  (1 baker)
  Parallel:   10 + 60/4 + 30 + 20/4 = 50 min  (4 bakers)
  Speedup:    120 / 50 = 2.4x     (not 4x)
```

The 40 minutes of serial work did not improve at all. With 100 bakers, the
speedup would be:

```text
  10 + 60/100 + 30 + 20/100 = 40.6 min
  speedup = 120 / 40.6 = 2.96x
```

You hired 100 bakers to get 3× faster. The oven and the recipe book are the
Amdahl limit.

---

# Intuition

```text
  Total work = S + P

  S = serial work      (cannot be split)
  P = parallel work    (split across n processors)

  Time with n processors:

      ┌──────────────────────────────────┐
      │   S    +    P / n               │
      │  fixed     divided               │
      └──────────────────────────────────┘

  As n → ∞, the P part vanishes, and the time approaches S.

  So the best possible speedup is S / (S + P) = 1 / p, where p = P/(S+P).
```

```text
  speedup
    │
64  │                                          ─ ─ ─ ─  limit = 1/p
    │                                    ─ ─ ─
32  │                              ─ ─ ─
    │                         ─ ─ ─
16  │                    ─ ─ ─
    │               ─ ─ ─            ← always concave,
 8  │          ─ ─ ─                     never a straight line
    │     ─ ─ ─
 4  │   ─ ─
    │  ─
 2  │ ─
    │─
 1  │
    └───┬────┬────┬────┬────┬────┬────┬────┬───►  n (processors)
        2    4    8   16   32   64  128
```

---

# Formal Definition

> **Amdahl's law:** The speedup of a system limited by a fraction `p` of
> parallelisable work, running on `n` processors, is
>
> ```
> speedup(n) = 1 / ( (1 − p) + p/n )
> ```
>
> where the **total work is held constant** (the fixed-work assumption).

| Term | Symbol | Meaning |
|---|---|---|
| Speedup | `speedup` | `T₁ / Tₙ` — original time divided by parallel time |
| Parallel fraction | `p` | 0 ≤ p ≤ 1 — the share of work that can be parallelised |
| Serial fraction | `1 − p` | the share that cannot |
| Processors | `n` | the number of workers, including the original one |
| Serial time | `S` | time spent on unparallelisable work |
| Parallel time | `P` | time spent on parallelisable work |

## Derivation

```text
  With 1 processor:      T₁ = S + P
  With n processors:     Tₙ = S + P/n

  speedup = T₁ / Tₙ
          = (S + P) / (S + P/n)

  Substitute p = P/(S+P) and (1−p) = S/(S+P), then divide top and bottom
  by (S+P):

          = 1 / ( (1−p) + p/n )
```

---

# Worked Examples

## Example 1 — 90% parallelisable, 4 processors

```text
  p = 0.90, n = 4

  speedup = 1 / ( (1 − 0.90) + 0.90/4 )
          = 1 / ( 0.10 + 0.225 )
          = 1 / 0.325
          = 3.08x
```

```text
  Original:  T₁ = 100 s  (10 s serial + 90 s parallel)
  Parallel:  T₄ = 10 + 90/4 = 10 + 22.5 = 32.5 s
  Speedup:   100 / 32.5 = 3.08x     (not 4x)
```

## Example 2 — 95% parallelisable, 8 processors

```text
  p = 0.95, n = 8

  speedup = 1 / ( 0.05 + 0.95/8 )
          = 1 / ( 0.05 + 0.11875 )
          = 1 / 0.16875
          = 5.93x
```

## Example 3 — 99% parallelisable, 16 processors

```text
  p = 0.99, n = 16

  speedup = 1 / ( 0.01 + 0.99/16 )
          = 1 / ( 0.01 + 0.061875 )
          = 1 / 0.071875
          = 13.91x
```

## Example 4 — the same program, as n grows

| n | p=0.90 | p=0.95 | p=0.99 |
|---|---|---|---|
| 1 | 1.00× | 1.00× | 1.00× |
| 2 | 1.53× | 1.90× | 1.98× |
| 4 | 2.11× | 3.17× | 3.92× |
| 8 | 2.56× | 4.80× | 7.01× |
| 16 | 3.20× | 7.01× | 12.59× |
| 32 | 3.91× | 10.13× | 22.28× |
| 64 | 4.63× | 14.31× | 39.30× |
| ∞ | **10.00×** | **20.00×** | **100.00×** |

> [!IMPORTANT]
> The ceiling is the serial fraction that remains. If 1% is serial, no amount
> of hardware will ever give more than a 100× speedup, and getting to 50×
> needs 99 cores of pure parallel efficiency.

## Example 5 — the inverse question

> "How many processors do I need for a 10× speedup on a program that is
> 80% parallelisable?"

```text
  10 = 1 / ( 0.20 + 0.80/n )

  0.10 = 0.20 + 0.80/n
  0.80/n = −0.10         ← negative: no solution!
```

```text
  Negative answer: 10× is UNREACHABLE.

  The maximum possible is 1/0.2 = 5×.
```

```text
  Try 4× instead:

  4 = 1 / ( 0.20 + 0.80/n )
  0.25 = 0.20 + 0.80/n
  0.80/n = 0.05
  n = 0.80 / 0.05 = 16
```

```text
  Answer: 16 processors, and only if scaling is perfectly efficient.
  In practice, budget 40–60, because efficiency is never 100%.
```

## Example 6 — measuring p from a real program

```c
#define _GNU_SOURCE
#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>
#include <time.h>
#include <unistd.h>

#define ITERS 60000000L

/* The parallelisable part: n independent units of work. */
static void parallel_part(int id)
{
    double x = id + 1.0;
    for (long i = 0; i < ITERS; i++)
        x = x * 1.0000001 + (i % 7);
    /* keep the result observable so the compiler cannot optimise it away */
    if (x < 0.0)
        printf("%f", x);
}

static void *worker(void *arg) { parallel_part((int)(long)arg); return NULL; }

static double now(void)
{
    struct timespec t;
    clock_gettime(CLOCK_MONOTONIC, &t);
    return t.tv_sec + t.tv_nsec / 1e9;
}

int main(void)
{
    long n = sysconf(_SC_NPROCESSORS_ONLN);
    if (n > 16) n = 16;

    /* Baseline: the same total work, done serially on one core. */
    double a = now();
    for (int i = 0; i < (int)n; i++)
        parallel_part(i);
    double t_serial = now() - a;

    /* Parallel: n threads. */
    pthread_t *t = malloc((size_t)n * sizeof *t);
    a = now();
    for (long i = 0; i < n; i++)
        pthread_create(&t[i], NULL, worker, (void *)i);
    for (long i = 0; i < n; i++)
        pthread_join(t[i], NULL);
    double t_par = now() - a;
    free(t);

    double speedup = t_par > 0 ? t_serial / t_par : 0;
    double eff     = 100.0 * speedup / (double)n;

    /*
     * Invert Amdahl's law to recover the parallel fraction:
     *     speedup = 1 / ( (1-p) + p/n )
     *  =>  p = n * (speedup - 1) / (n * speedup - 1)
     */
    double p = (double)n * (speedup - 1.0) / ((double)n * speedup - 1.0);

    printf("cores     : %ld\n", n);
    printf("serial    : %.4f s\n", t_serial);
    printf("parallel  : %.4f s\n", t_par);
    printf("speedup   : %.2fx\n", speedup);
    printf("efficiency: %.1f%%\n", eff);
    printf("implied p : %.4f  (%.2f%% parallelisable)\n", p, p * 100);
    printf("Amdahl cap: %.2fx  (1/p)\n", p > 0 ? 1.0 / p : 0.0);
    return 0;
}
```

```bash
gcc -O2 -pthread -D_GNU_SOURCE -o amdahl amdahl.c
./amdahl
```

```text
cores     : 8
serial    : 3.8412 s
parallel  : 0.6104 s
speedup   : 6.29x
efficiency: 78.7%
implied p : 0.9573  (95.73% of the work parallelises)
```

```bash
# Add a serial section and watch p fall
# e.g. a sequential merge phase, or a printf between phases
```

---

# The Fixed-Work Assumption

Amdahl's law holds the **total problem size constant**. The same job is done,
just faster.

This is the right model for:

- A web server handling a fixed request rate
- A batch job with a fixed input file
- A simulation with a fixed number of steps
- A build with a fixed number of source files

> [!IMPORTANT]
> Under fixed work, more cores always help — but with diminishing returns,
> and never by more than `1/p`.

## The alternative: fixed time (scaling, not speedup)

Gustafson observed that when you give each of `n` processors the **same amount
of work as the whole program had**, the serial fraction shrinks as `n` grows.

```text
  Original problem:  S + P

  On n processors, with each processor holding a full copy of the data:

      Tₙ = S + n·P          (not S + P/n)

  speedupₙ = (S + P) / (S + nP)  →  1 as n → ∞
```

> [!TIP]
> The two laws are not in conflict. They answer different questions.
> Amdahl: *how fast can I do this fixed job?*
> Gustafson: *how much can I do in this fixed time?*
> Modern systems do both: a fixed job that scales slightly, or a fixed time
> that scales nearly perfectly.

| | Amdahl | Gustafson |
|---|---|---|
| Assumption | fixed total work | fixed execution time |
| Problem size | constant | grows with `n` |
| Per-core memory | shared or partitioned | a full copy each |
| As `n`→∞ | speedup → `1/p` | speedup → 1 |
| Models | a fixed job run faster | a bigger job in the same time |
| Use | batch jobs, servers | simulation, large-scale search |

---

# What Amdahl's Law Taught the Industry

Amdahl published the law in 1967. Its effect on hardware design was immediate.

```text
  The argument:

    1. If speedup is capped at 1/p, then a very fast serial unit is valuable.
    2. Build one core that is N times faster, and you get nearly an N-fold
       speedup on the serial part — with no parallelisation effort.
    3. Therefore: make the single core fast, THEN add cores.

  This is exactly the trajectory the industry took:

    1980s   1 core,  getting much faster (clock speed)
    2000s   multicore, many slow-ish cores
    Today   wide SIMD (AVX-512), vector units, GPUs, tensor cores
```

The same argument is why GPUs work: they accept very low single-thread
efficiency (10–20% of peak) in exchange for thousands of parallel units, and
they win because the parallel fraction in graphics and machine learning is
very high — often above 0.95.

```text
  p = 0.99 on a GPU with 10,000 cores:
      speedup = 1 / ( 0.01 + 0.99/10000 ) = 1 / 0.010099 = 99.0x

  p = 0.50 on the same GPU:
      speedup = 1 / ( 0.50 + 0.5/10000 )  = 1 / 0.50005  =  2.0x
```

---

# Beyond Amdahl: Other Limits on Real Speedup

Amdahl assumes perfect scaling. Reality has further losses.

## 1. Synchronisation overhead

```c
/* Amdahl's law assumes this is free. It is not. */
static pthread_mutex_t lock = PTHREAD_MUTEX_INITIALIZER;
static double shared_sum;

static void *worker(void *arg)
{
    double local = 0;
    for (int i = 0; i < 100000; i++)
        local += compute(i);

    pthread_mutex_lock(&lock);      /* all threads pile up here */
    shared_sum += local;
    pthread_mutex_unlock(&lock);
    return NULL;
}
```

Amdahl models this as part of the serial fraction. In practice it is worse,
because it is a serial *section repeated n times*.

## 2. Memory bandwidth

```text
  On 4 cores:  per-core memory bandwidth ≈ 20 GB/s
  On 64 cores: per-core memory bandwidth ≈ 3 GB/s

  The cores spend most of their time stalled on memory, not computing.
  This is why efficiency falls off a cliff past the cache size.
```

## 3. False sharing

```c
/* Two variables, one cache line, two cores ping-ponging it. */
struct { int a; int b; } shared;      /* 8 bytes, ONE 64-byte cache line */
```

The threads are logically independent, but the hardware cannot tell.

## 4. Load imbalance

```text
  8 threads, 8 units of work, but one unit takes 10× the others.

  7 threads finish early and idle.
  The makespan is set by the slowest thread.

  Fix: work stealing.
```

## 5. NUMA effects

```bash
numactl --hardware
```

```text
  available: 2 nodes (0-31), 0-15 = node0, 16-31 = node1

  Local memory read  : ~80 GB/s
  Remote memory read : ~40 GB/s     ← 2x slower

  Threads whose data is on another node pay the remote cost.
```

```bash
# Pin threads and their memory to the same node
numactl --cpunodebind=0 --membind=0 ./program
```

## 6. Tur Amdahl's law

A more aggressive model: the serial part also uses only one processor, so
adding processors *slows it down* through contention.

```text
  speedup = 1 / ( (1−p) + p/n + n·(1−p)·k )

  With k = 0.01 and p = 0.9:
      n=64:  1 / ( 0.1 + 0.014 + 0.064 ) = 5.6x
  Some contention is unavoidable, and Tur's law says so explicitly.
```

---

# Applying Amdahl's Law to Real Programs

## Identify the serial fraction

```bash
# Where is the time actually going?
perf record -g ./program
perf report
```

```text
  Overhead  Command
   62.4%  work_parallel        ← the 62% parallel part
   18.1%  read_file            ← serial
    9.7%  parse_header         ← serial
    6.2%  malloc               ← mostly serial
    3.6%  merge_results        ← serial
```

```text
  Serial fraction ≈ (18.1 + 9.7 + 6.2 + 3.6) / 100 = 0.376
  p ≈ 0.624

  Maximum achievable speedup = 1 / 0.376 = 2.66x
```

> [!TIP]
> Before you invest in parallelising, find out what `p` already is. A program
> that is 70% serial can never exceed 1.43× no matter how many cores you add.
> Making one of the serial parts 3× faster is usually the highest-value change
> available.

## Common serial bottlenecks in real code

| Bottleneck | Fix |
|---|---|
| Reading a whole file into one buffer | Stream it in chunks, process chunks in parallel |
| A single output stage | Pipeline it, do it incrementally |
| `printf` in a loop | Buffer, then write once |
| One global mutex | Shard it, or use atomics |
| Sorting one huge array | Partition, then merge in parallel |
| Allocator contention | Per-thread arenas, or a lock-free allocator |

---

# Common Misconceptions

### ❌ "Amdahl's law says multicore does not work."

Incorrect.

It says multicore works, with a ceiling. A program that is 99% parallelisable
gets 100× on 100 cores. The law is a *limit*, not a prohibition — and it is
usually not the binding constraint, because real efficiency is well below
perfect long before the Amdahl limit is reached.

---

### ❌ "Speedup can be greater than the number of cores."

Under fixed work and Amdahl's assumptions, never. If you measure 8× speedup on
4 cores, one of three things is true: the problem got smaller, you are
measuring wrong, or the serial work also got faster for another reason.

---

### ❌ "Amdahl's law assumes the work cannot be improved."

Incorrect.

The law describes a *fixed* program. Parallelising more of it, or making a
serial part faster, changes `p`. The law is a diagnostic: use it to find out
where the ceiling is, then raise the ceiling.

---

### ❌ "The serial part has to be exactly 0% for full linear speedup."

No.

You need serial time to be negligible. At 64 cores, `p = 0.99` gives 39×
(61% efficiency). At `p = 0.999`, 64 cores gives 50× (78% efficiency). The
question is never binary.

---

### ❌ "Gustafson's law contradicts Amdahl's law."

Incorrect.

They assume different goals. Amdahl fixes the work; Gustafson fixes the time.
A modern system usually does a bit of both, which is why a 64-core machine
running a fixed job might get 20×, while a simulation scaled to fill 64 cores
gets 55×.

---

# Interview Questions

### Basic

- State Amdahl's law. Explain each term.
- What is the maximum speedup possible according to Amdahl's law?
- If 90% of a program is parallelisable, what is the maximum speedup?
- How many processors are needed for a 4× speedup on a 75% parallel program?

### Intermediate

- Differentiate between Amdahl's law and Gustafson's law.
- What is the fixed-work assumption, and why does it matter?
- Why does parallel efficiency drop as you add processors?
- A program runs 3× faster on 8 cores. What is its parallel fraction?

### Advanced

- Why did Amdahl's law lead the industry to increase single-core clock speed first?
- Why do GPUs tolerate such low parallel efficiency?
- Beyond Amdahl's law, what other factors limit real multicore scaling?
- How would you use Amdahl's law to decide whether a rewrite is worthwhile?

---

# University Exam Notes

### Definitions

- **Amdahl's Law:** The speedup of a system with a parallel fraction `p` running
  on `n` processors is `speedup = 1 / ((1 − p) + p/n)`, under the assumption
  of fixed total work.
- **Speedup:** The ratio of execution time with one processor to execution time
  with `n` processors.
- **Serial fraction:** `1 − p`, the portion of work that cannot be parallelised.
- **Amdahl's limit:** The maximum achievable speedup, `1/p`, as `n → ∞`.
- **Parallel efficiency:** `speedup / n`, the fraction of ideal speedup
  actually realised.
- **Gustafson's Law:** With fixed execution time and work proportional to `n`,
  speedup `→ 1` as `n → ∞`.

### Frequently Asked Questions

- State and derive Amdahl's law.
- What is the maximum speedup possible? Justify.
- Differentiate between Amdahl's law and Gustafson's law.
- Explain the significance of Amdahl's law in multicore systems.
- What is parallel efficiency? How is it calculated?
- A program is 80% parallelisable. What speedup is achieved with 16 cores?

---

# Key Takeaways

- `speedup(n) = 1 / ((1 − p) + p/n)`, where `p` is the parallel fraction.
- The maximum speedup is `1/p`, independent of how many processors you buy.
- With `p = 0.5` you can never exceed 2×. With `p = 0.99`, never more than
  100×.
- Amdahl holds **total work constant**; Gustafson holds **time constant** and
  lets the problem grow.
- The law explains the industry's historical emphasis on fast single cores,
  and why GPUs work despite very low per-core efficiency.
- Real systems are limited by more than Amdahl: synchronisation, memory
  bandwidth, false sharing, load imbalance and NUMA.
- Use the law diagnostically: measure `p` with `perf`, find the serial
  bottleneck, and raise `p` before buying more cores.

---

# References

- "Validity of the Single Processor Approach to Achieving Large Scale Computing Capabilities", AFIPS Spring Joint Computer Conference, 1967
- Operating System Concepts — Silberschatz
- The Art of Multiprocessor Programming — Herlihy and Shavit
- Introduction to Parallel Computing — Grama, Gupta, Karypis, Kumar
- `perf record`, `perf report`, `numactl --hardware`, `sysconf(_SC_NPROCESSORS_ONLN)`
