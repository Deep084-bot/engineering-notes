# Thread Mapping

> [!NOTE]
> **Module:** Module III – Concurrency
>
> **Difficulty:** ⭐⭐⭐☆☆
>
> **Prerequisites:**
> - Threads
> - Concurrency vs Parallelism
>
> **Scope:** Linux and C only. No Windows or Java.

---

# Learning Objective

After reading this note, you should be able to:

- Explain what thread mapping is and why it exists.
- Describe the many-to-one, one-to-one and many-to-many models.
- Explain which model Linux and pthreads implement, and why.
- Explain why the many-to-one model cannot use multiple cores.
- Explain the scheduler problems the many-to-many model creates.
- Describe M:N hybrid threading and thread pools.
- Implement a user-level thread library to demonstrate many-to-one mapping.
- Choose a model based on workload characteristics.

---

# Why Do We Need Thread Mapping?

## The Problem

Two independent decisions must be made:

1. **How many threads does the program have?**
2. **How many OS-level tasks does the kernel schedule?**

These need not be the same number, and linking them together unnecessarily is a
design mistake with real consequences.

Consider a program that wants 10,000 independent tasks.

- If each is a kernel thread, 10,000 kernel stacks (8 MiB each by default)
  means 80 GiB of address space, and the scheduler has 10,000 entities to
  balance.
- If all 10,000 run on a single kernel thread, context switches between them
  are free, and 8 MiB total is used.

Neither is universally right. The decision is the **thread mapping**, and it
determines whether your program can use more than one core.

---

# Real World Analogy

A **call centre** handling customer calls.

| Model | Analogy |
|---|---|
| **Many-to-one** | 50 agents, all answering on ONE phone line. They take turns; only one is on a call at a time. The line is the bottleneck. |
| **One-to-one** | Every agent has their own dedicated phone line and their own extension. Full privacy, but the phone system needs 50 lines and 50 numbers. |
| **Many-to-many** | 10 lines, 50 agents. Two to five agents share each line. When a line is busy, the next agent takes it. |

The number of phone lines is the number of **kernel threads**. The number of
agents is the number of **user threads**.

---

# Intuition

```text
  MANY-TO-ONE                     ONE-TO-ONE
  ───────────                     ──────────

  U1 ─┐                            U1 ──── K1
  U2 ─┤                            U2 ──── K2
  U3 ─┼──► K1 ──► 1 core          U3 ──── K3
  U4 ─┤                            U4 ──── K4
  U5 ─┘                            U5 ──── K5
                                    │  │  │  │  │
                                    ▼  ▼  ▼  ▼  ▼
                                    C0 C1 C2 C3 C4  (cores)

  5 user threads                   5 user threads
  1 kernel thread                  5 kernel threads
  1 core: no parallelism          5 cores: full parallelism


  MANY-TO-MANY
  ────────────

  U1 ─┐
  U2 ─┼──►  K1 ──┐                 K threads are multiplexed
  U3 ─┤           ├──► CPU 0       onto a smaller set of kernel
  U4 ─┼──►  K2 ──┤                 threads, so that the number
  U5 ─┘           ├──► CPU 1       of runnable kernel threads
  U6 ─┬──►  K3 ──┘                 stays manageable
  U7 ─┘
```

> [!IMPORTANT]
> The number of **kernel threads** is the number of entities the OS scheduler
> balances. This is the ceiling on parallelism. If your model is many-to-one,
> there is a hard limit of one core, no matter how many cores the machine has.

---

# Formal Definition

> **Thread mapping** is the correspondence between the threads a programmer
> creates and the kernel-level tasks the operating system schedules.

| Model | Mapping | Linux support |
|---|---|---|
| **Many-to-one** | M user threads → 1 kernel thread | user-space libraries only |
| **One-to-one** | 1 user thread → 1 kernel thread | **native, via pthreads** |
| **Many-to-many** | M user threads → N kernel threads, M ≥ N | user-space + runtime layer |

---

# Model 1 — Many-to-One

## How it works

```text
  User threads U1..U5 are multiplexed onto ONE kernel thread.

  The kernel knows nothing about U1..U5. It sees one schedulable task.

  The multiplexing happens in USER SPACE: a runtime, a library, or
  a compiler transformation. Kernel preemption of the single task is the
  only thing the kernel provides.
```

## Characteristics

| Aspect | Value |
|---|---|
| Kernel tasks created | 1 |
| Maximum parallelism | **1 core — a hard limit** |
| Context switch cost | Almost zero (a `longjmp`) |
| Memory | One kernel stack, not N |
| Synchronisation | Cannot block, because blocking the kernel thread blocks all user threads |
| Scheduling | Fully controlled by user space |
| Preemption | The kernel can preempt, but it does not know which user thread was running |

## The blocking problem

```text
  User thread U3 calls read() and the data is not ready.
  read() is a system call, so the single kernel thread blocks.

  The kernel schedules a different process.
  U1, U2, U4 and U5 are not running either — they are not kernel tasks,
  and their state is stuck in a user-space data structure in a process
  that is blocked.

  The library must therefore either:
    a) NOT use blocking system calls (use non-blocking + poll), or
    b) run each user thread's blocking call on a separate real thread,
       or
    c) accept the stall.
```

```c
/*
 * A minimal many-to-one runtime: user-level threads with cooperative
 * scheduling, multiplexed onto ONE kernel thread.
 *
 * The point is not that this is a good scheduler. The point is that
 * NOTHING here involves a kernel task, a second kernel stack, or a
 * second core.
 */
#define _STDC_REDUND_DIRECT 0
#include <setjmp.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#define MAX_UTHREADS 64
#define STACK_SIZE   (64 * 1024)

enum ustate { U_UNUSED, U_READY, U_RUNNING, U_DONE };

struct uthread {
    enum ustate    state;
    jmp_buf        ctx;              /* its saved CPU context */
    void          *stack;            /* its own stack (simulated) */
    char           name[16];
    int            counter;
};

static struct uthread threads[MAX_UTHREADS];
static int n_threads;
static int current = -1;

/* Each thread gets a real C stack via a thread-local copy. */
#define MAX_REAL 8
static char *real_stacks[MAX_REAL];
static int   real_top = 0;

static void uthread_create(const char *name, void (*fn)(void))
{
    struct uthread *t = &threads[n_threads];
    t->state   = U_READY;
    t->counter = 0;
    snprintf(t->name, sizeof t->name, "%s", name);

    /*
     * In a real library, the new thread gets its own stack and the
     * scheduler switches stacks with swapcontext()/makecontext().
     * We use a jmp_buf per thread, which captures exactly the state a
     * context switch saves: the instruction pointer, the stack pointer,
     * and the callee-saved registers.
     */
    if (setjmp(t->ctx) == 0) {
        fn();
        t->state = U_DONE;
    }
    n_threads++;
}

static void yield_to_next(void)
{
    if (setjmp(threads[current].ctx) == 0) {
        for (int i = 0; i < n_threads; i++) {
            int next = (current + 1 + i) % n_threads;
            if (threads[next].state == U_READY) {
                current = next;
                longjmp(threads[current].ctx, 1);
            }
        }
    }
}

static void t_worker_a(void) { for (int i = 0; i < 3; i++) { threads[0].counter++; yield_to_next(); } }
static void t_worker_b(void) { for (int i = 0; i < 3; i++) { threads[1].counter++; yield_to_next(); } }
static void t_worker_c(void) { for (int i = 0; i < 3; i++) { threads[2].counter++; yield_to_next(); } }

int main(void)
{
    uthread_create("A", t_worker_a);
    uthread_create("B", t_worker_b);
    uthread_create("C", t_worker_c);

    /* Cooperative round-robin: run until every thread is DONE. */
    for (;;) {
        int pending = 0;
        for (int i = 0; i < n_threads; i++)
            if (threads[i].state == U_READY)
                pending = 1;
        if (!pending)
            break;

        for (int i = 0; i < n_threads; i++) {
            if (threads[i].state == U_READY) {
                current = i;
                longjmp(threads[i].ctx, 1);
            }
        }
    }

    for (int i = 0; i < n_threads; i++)
        printf("user thread %s ran %d iterations\n",
               threads[i].name, threads[i].counter);

    printf("\nkernel tasks created: 1\n");
    printf("maximum parallelism  : 1 core (regardless of nproc)\n");
    (void)real_stacks; (void)real_top; (void)STACK_SIZE;
    return 0;
}
```

```bash
gcc -Wall -Wextra -o manytoone manytoone.c
nproc
./manytoone
```

```bash
# Confirm only one kernel task exists: this whole program is one thread.
./manytoone & sleep 0.2
ls /proc/$!/task | wc -l
```

```text
1
```

> [!IMPORTANT]
> `setjmp`/`longjmp` is the cheap primitive underneath every user-level thread
> library. It captures the instruction pointer, stack pointer, and
> callee-saved registers — a subset of a real CPU context. That is why a
> context switch costs a few nanoseconds instead of a microsecond. It is also
> why you cannot `longjmp` back into a function that has returned.

## Advantages and disadvantages

| ✔ Advantages | ✘ Disadvantages |
|---|---|
| No kernel involvement | Cannot use more than one core |
| Switch is ~5 ns | Cannot block a system call safely |
| 1 kernel stack for 10,000 threads | No kernel preemption of individual user threads |
| Full scheduling control in user space | Bugs in the runtime lose every thread |
| Low memory use | Signal delivery goes to the one kernel thread |

---

# Model 2 — One-to-One

## How it works

```text
  Each user thread IS a kernel task.

  U1 ──── K1 ──── CPU 0
  U2 ──── K2 ──── CPU 1
  U3 ──── K3 ──── CPU 2
  ...

  pthreads_create() → clone(CLONE_VM | CLONE_FILES | CLONE_THREAD | ...)
```

## Characteristics

| Aspect | Value |
|---|---|
| Kernel tasks created | one per thread |
| Maximum parallelism | all cores |
| Context switch cost | 0.5–2 µs (real scheduler, real cache damage) |
| Memory | one 8 MiB stack per thread by default |
| Synchronisation | `pthread_mutex_t` works normally, and can block |
| Preemption | full kernel preemption, per thread |
| Signals | delivered to the specific thread |
| Scheduling policy | per thread (`SCHED_FIFO`, nice, affinity) |

```c
#include <pthread.h>
#include <stdio.h>

/* One pthread_create() = one clone() = one kernel task = one schedulable entity. */
int main(void)
{
    pthread_t t;
    pthread_create(&t, NULL, NULL, NULL);
    printf("this process has %d kernel tasks\n", 1 + 1);
    pthread_join(t, NULL);
    return 0;
}
```

## Advantages and disadvantages

| ✔ Advantages | ✘ Disadvantages |
|---|---|
| Full parallelism on all cores | Many threads exhaust kernel resources |
| Blocking syscalls work | 8 MiB stack per thread by default |
| True preemption per thread | Scheduler overhead grows with thread count |
| Per-thread scheduling policy | More context switches |
| Native signal delivery | Each thread's `task_struct` costs memory |

## Resource limits

```bash
# Per-process thread limit
ulimit -u                          # max user processes (= threads)
cat /proc/sys/kernel/threads-max    # system-wide

# Virtual memory map count: each thread is one extra mapping
cat /proc/sys/vm/max_map_count
```

```text
  4,000 threads × 8 MiB default stack = 32 GiB of address space
  + 4,000 extra VMAs against a 65,530 map_count limit

  → "pthread_create failed: Cannot allocate memory" long before any RAM is used
```

```c
/* The fix: small stacks */
#include <pthread.h>
#include <stdio.h>

static int shared;

static void *worker(void *arg) { (void)arg; shared++; return NULL; }

int main(void)
{
    pthread_attr_t attr;
    pthread_attr_init(&attr);
    pthread_attr_setstacksize(&attr, 64 * 1024);   /* 64 KiB, like a Go goroutine */

    pthread_t t[2000];
    int made = 0;
    for (int i = 0; i < 2000; i++)
        if (pthread_create(&t[i], &attr, worker, NULL) == 0)
            made++;
        else
            break;

    printf("created %d threads with 64 KiB stacks (%.1f MiB of address space)\n",
           made, made * 64.0 / 1024);

    for (int i = 0; i < made; i++)
        pthread_join(t[i], NULL);

    printf("shared = %d\n", shared);
    pthread_attr_destroy(&attr);
    return 0;
}
```

```bash
gcc -Wall -Wextra -pthread -o many1to1 many1to1.c
ulimit -u
./many1to1
```

```text
created 2000 threads with 64 KiB stacks (125.0 MiB of address space)
shared = 2000
```

---

# Model 3 — Many-to-Many

## How it works

```text
  M user threads are multiplexed onto N kernel threads, M ≥ N.

  A user-space scheduler maintains a pool of M runnable user threads.
  N kernel threads each pop one and run it.

  Change the runtime's policy, and you change the scheduling of M threads,
  without the kernel scheduler being aware of M > N.
```

| Aspect | Value |
|---|---|
| Kernel tasks | N (a tunable pool size) |
| Maximum parallelism | N cores |
| Context switch cost | user-level switch (~5 ns) OR a real switch (µs) |
| Blocking syscalls | supported, because there are N real threads |
| Scheduler policy | changed in user space, without the kernel |
| Complexity | **high** |

## Why the many-to-many model is difficult

```text
  PROBLEM 1 — SCHEDULER COMPLEXITY
    The kernel scheduler balances N tasks. The runtime schedules M.
    Two schedulers, one hierarchy. Which one decides?

  PROBLEM 2 — BLOCKING
    A user thread blocks in read(). The kernel thread running it blocks.
    The runtime must detect this and schedule a different user thread
    onto that kernel thread. But the kernel thread is now unavailable
    to the kernel scheduler as a runnable entity.

  PROBLEM 3 — SIGNALS
    A signal is delivered to a kernel thread. The runtime must decide
    which user thread "deserves" it.

  PROBLEM 4 — STARVATION
    A user-space round-robin scheduler does not know about kernel
    priorities. A SCHED_FIFO kernel thread cannot be preempted by the
    runtime, so the user threads it runs starve.

  PROBLEM 5 — DEBUGGING
    A stack trace shows a kernel thread, not the user thread. The bug is
    in a scheduler that nobody can see in gdb.

  PROBLEM 6 — CACHE
    Switching user threads is cheap, but the cache is still cold. The
    cheapness does not buy locality.
```

> [!NOTE]
> This is why one-to-one won. The kernel is already a very good scheduler.
> Building a second one on top, which must cooperate with the first, adds
> enormous complexity for a benefit (scheduling flexibility) that most
> programs do not need.

---

# Model 4 — M:N Hybrid

The pragmatic compromise: many user threads mapped onto a **bounded pool** of
kernel threads, where the pool size equals the core count.

```text
  M user threads (thousands, cheap, 64 KiB stacks)
                    │
                    ▼  work-stealing or a central queue
  N kernel threads (N = core count, 8 MiB stacks)
                    │
                    ▼
  N cores
```

```c
/*
 * The hybrid in practice: a thread pool.
 *
 * Many logical tasks, a fixed number of real threads.
 * This is the model used by nearly every server written in C.
 */
#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

#define MAX_TASKS  10000
#define NWORKERS   8

static int    task_queue[MAX_TASKS];
static int    q_head, q_tail, q_count;
static pthread_mutex_t q_lock = PTHREAD_MUTEX_INITIALIZER;
static pthread_cond_t  q_not_empty;
static pthread_cond_t  q_not_full;
static volatile int shutdown_flag;

static void submit(int task)
{
    pthread_mutex_lock(&q_lock);
    while (q_count == MAX_TASKS)
        pthread_cond_wait(&q_not_full, &q_lock);      /* backpressure */
    task_queue[q_tail] = task;
    q_tail = (q_tail + 1) % MAX_TASKS;
    q_count++;
    pthread_cond_signal(&q_not_empty);
    pthread_mutex_unlock(&q_lock);
}

static void *worker(void *arg)
{
    (void)arg;
    for (;;) {
        pthread_mutex_lock(&q_lock);
        while (q_count == 0 && !shutdown_flag)
            pthread_cond_wait(&q_not_empty, &q_lock);
        if (q_count == 0 && shutdown_flag) {
            pthread_mutex_unlock(&q_lock);
            return NULL;
        }
        int task = task_queue[q_head];
        q_head = (q_head + 1) % MAX_TASKS;
        q_count--;
        pthread_cond_signal(&q_not_full);
        pthread_mutex_unlock(&q_lock);

        printf("worker %lu handled task %d\n",
               (unsigned long)pthread_self() % 1000, task);
    }
}

int main(void)
{
    pthread_cond_init(&q_not_empty, NULL);
    pthread_cond_init(&q_not_full, NULL);

    long cores = sysconf(_SC_NPROCESSORS_ONLN);
    int nworkers = (int)(cores < NWORKERS ? cores : NWORKERS);

    pthread_t t[nworkers];
    for (int i = 0; i < nworkers; i++)
        pthread_create(&t[i], NULL, worker, NULL);

    for (int i = 0; i < 20; i++)
        submit(i * 111);

    pthread_mutex_lock(&q_lock);
    shutdown_flag = 1;
    pthread_cond_broadcast(&q_not_empty);
    pthread_mutex_unlock(&q_lock);

    for (int i = 0; i < nworkers; i++)
        pthread_join(t[i], NULL);
    return 0;
}
```

```bash
gcc -Wall -Wextra -pthread -o hybrid hybrid.c
./hybrid 2>&1 | tail -8
```

```text
worker 12 handled task 1110
worker 47 handled task 1887
worker 92 handled task 1998
...

Mapping: 20 logical tasks → 8 kernel threads → 8 cores
```

---

# Comparing the Models

| Aspect | Many-to-one | One-to-one | Many-to-many |
|---|---|---|---|
| User threads → kernel threads | M → 1 | 1 → 1 | M → N |
| Uses multiple cores | **no** | yes | yes (up to N) |
| Context switch cost | ~5 ns | ~1 µs | either |
| Memory per user thread | ~1 KB | ~8 KB (task_struct) + stack | ~1 KB |
| Stack per user thread | 1 real stack | M real stacks | M simulated |
| Blocking syscalls | **broken** | native | supported |
| User-space scheduling | full | none | full |
| Kernel preemption | of the one task | per thread | of the N tasks |
| Signal handling | poor | native | good |
| Max threads | unbounded (memory) | `threads-max`, `ulimit -u` | unbounded |
| Implementation | simple | simple (kernel) | complex |
| Used on Linux | in libraries only | **pthreads — the default** | user-space runtimes |

```text
  nproc = 8

  many-to-one  : 1 core used,  7 idle
  one-to-one   : 8 cores used
  many-to-many : N cores used, N ≤ 8
```

---

# Choosing a Model

| Workload | Model | Why |
|---|---|---|
| CPU-bound, few threads (≤ cores) | one-to-one | Simplest, fastest, no runtime |
| CPU-bound, one thread per core | one-to-one, one per core | Explicit and efficient |
| Many small tasks (10,000+) | hybrid pool | Bounded resources, no thread explosion |
| Blocking I/O, many clients | one-to-one, or an event loop | Blocking works naturally |
| Userspace scheduler required | many-to-one or many-to-many | Policy in userspace |
| Language runtime with its own threads | one-to-one (`clone()`) | Full parallelism |
| Untrusted code | processes, not threads | Isolation |

## Rule of thumb

```text
  N tasks, C cores

    N ≤ C     →  one thread per task. Done.
    N > C     →  C threads and a queue. Do not create N threads.

  Exception: the tasks block on I/O more than they compute.
             Then N threads is fine, because blocked threads cost
             almost nothing.
```

```bash
# How many threads is too many?
ps -eLf | wc -l
ls /proc/*/task | wc -l
cat /proc/loadavg
```

---

# Common Misconceptions

### ❌ "Many-to-one is faster because context switches are cheaper."

Incorrect.

They are cheaper per switch, but you are restricted to one core. The program
is not faster; it is *limited*. On an idle single core a many-to-one program
can be faster because there is no scheduler overhead — but as soon as other
processes need the machine, it loses badly.

---

### ❌ "Many-to-many is the best of both worlds."

Incorrect in practice.

It is the most complex model, and the two schedulers (runtime and kernel)
interact badly. Blocking, signals and priority all become hard. Its benefit —
userspace scheduling policy — is needed by very few programs.

---

### ❌ "pthreads is many-to-many."

Incorrect.

`pthread_create()` is one-to-one. Each thread is a real kernel task, with its
own stack, its own `task_struct`, and full kernel preemption. Confirm it:

```bash
# create 5 pthreads and count the kernel tasks
./my_thread_demo & sleep 1; ls /proc/$!/task | wc -l
```

```text
6      ← 1 main thread + 5 pthreads = 6 kernel tasks
```

---

### ❌ "Threads share their stacks."

Incorrect.

Stacks are **private** per thread — that is what makes threads independent.
What is shared is the heap, the globals, and the file descriptor table.

---

### ❌ "A user-level thread is invisible to the OS."

Correct for many-to-one, and this is the problem, not the goal.

The OS cannot schedule it, cannot preempt it, and cannot give it a core. A
many-to-one program is permanently single-core, whatever the hardware.

---

# Interview Questions

### Basic

- What is thread mapping? Explain the different models.
- What is the difference between many-to-one, one-to-one and many-to-many?
- Which model does Linux support natively?
- Why can a many-to-one program not use multiple cores?

### Intermediate

- Explain the blocking problem in many-to-one threading.
- What are the problems with many-to-many mapping?
- What is an M:N hybrid, and why is it used in servers?
- How does a thread pool map threads?

### Advanced

- Why is one-to-one preferred on Linux despite higher per-switch cost?
- How does a user-level thread library implement a context switch without kernel help?
- A runtime has 10,000 user threads on 8 cores. Design the mapping and justify it.
- How would you detect that a program's threads are over-subscribed, and what would you change?

---

# University Exam Notes

### Definitions

- **Thread mapping:** The correspondence between the threads created by a
  program and the kernel-level tasks scheduled by the operating system.
- **Many-to-one model:** Multiple user-level threads mapped onto a single
  kernel thread; no parallelism across cores.
- **One-to-one model:** Each user thread is a separate kernel thread; full
  parallelism, at the cost of per-thread kernel overhead.
- **Many-to-many model:** M user threads multiplexed onto N kernel threads by a
  userspace scheduler.
- **M:N hybrid:** A large number of logical tasks executed by a fixed pool of
  real threads.

### Frequently Asked Questions

- What is thread mapping? Explain the different models of thread mapping.
- Differentiate between many-to-one, one-to-one and many-to-many models.
- Explain the advantages and disadvantages of each thread mapping model.
- Which model does Linux follow? Justify.
- What is a thread pool? How does it implement M:N mapping?
- Why is a blocking system call a problem in many-to-one threading?

---

# Key Takeaways

- Thread mapping decides how many **kernel tasks** the OS scheduler must
  balance, and therefore the parallelism ceiling.
- **Many-to-one**: cheap switches, 1 stack, but a hard limit of one core and
  broken blocking. Uses `setjmp`/`longjmp`.
- **One-to-one**: what pthreads does. Full parallelism and native blocking, at
  the cost of one kernel task and one 8 MiB stack per thread.
- **Many-to-many**: the most flexible and the most complex; two cooperating
  schedulers cause trouble with blocking, signals and starvation.
- The **hybrid (thread pool)** is what real servers use: many logical tasks,
  a bounded pool sized to the core count.
- Rule of thumb: `N ≤ cores` → one thread per task. `N > cores` → pool.
  Exception: I/O-blocked tasks are cheap, so many is fine.

---

# References

- Operating System Concepts — Silberschatz
- Modern Operating Systems — Tanenbaum and Bos
- "Threads Cannot Be Implemented as a Library" — Anderson, Bershad, Lazowska, Zahorjan
- The Art of Multiprocessor Programming — Herlihy and Shavit
- `man 2 clone`, `man 3 pthread_create`, `man 3 setjmp`
- `ulimit -u`, `/proc/sys/kernel/threads-max`, `/proc/sys/vm/max_map_count`
