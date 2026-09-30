# Context Switching

> [!NOTE]
> **Module:** Module I – Introduction
>
> **Difficulty:** ⭐⭐⭐⭐☆
>
> **Prerequisites:**
> - Modes of Operation (cpu-modes.md)
> - Processes
>
> **Scope:** Linux and C only. No Windows or Java.

---

# Learning Objective

After reading this note, you should be able to:

- Define context switching and describe exactly what is saved and restored.
- Distinguish a context switch from a mode switch.
- List the triggers of a context switch.
- Explain why context switches are expensive and quantify the cost.
- Describe how the Linux scheduler performs a switch.
- Measure context switches on a live Linux system.
- Explain the effect of context switching on caches and on throughput.

---

# Why Do We Need Context Switching?

## The Problem

A single CPU core can execute only one instruction stream at a time.

But you want to run 500 programs at once, each of which believes it has the
whole machine to itself.

The CPU cannot honour that belief. Someone has to take the instruction stream
away from one program and give it to another, and then switch back later —
hundreds of thousands of times per second.

The mechanism for taking a stream away and giving it back is a **context
switch**.

> [!IMPORTANT]
> A context switch does not move a process. The process stays in memory, at
> the same addresses, in the same place. Only the CPU's attention moves.

---

# Real World Analogy

A manager juggling four projects.

Each project has a desk with papers, a chair pushed back, a pen, and a
half-finished thought.

When the phone rings, the manager:

1. **Saves the state** — pushes the chair back, caps the pen, writes a sticky
   note: "here is exactly where I was".
2. **Switches** — answers the phone.
3. **Returns** — reads the sticky note, sits down, un-caps the pen, continues
   mid-sentence.

The papers never moved. Only the manager's attention moved.

Doing this 100,000 times an hour is a context-switching scheduler.

---

# Intuition

```text
  ┌──────────────── CPU Core 0 ─────────────────┐
  │                                              │
  │  RIP = 0x401136    ┌──────────────────┐     │
  │  RAX = 0x0000ff00  │  running process │     │
  │  RBX = 0x7ffd...   │       A          │     │
  │  RSP = 0x7fff...   └──────────────────┘     │
  │                                              │
  └──────────────────────────────────────────────┘
              │              ▲
              │   1. save    │  3. load
              │   A's state  │  B's state
              ▼              │
        ┌──────────────────────────────────┐
        │  Per-process Kernel Stack       │
        │  ┌────────────────────────────┐  │
        │  │ A's saved registers       │  │
        │  │ A's saved RIP  = 0x401136│  │
        │  │ A's saved RSP  = 0x7fff..│  │
        │  │ A's saved RFLAGS          │  │
        │  │ A's FPU/SIMD state        │  │
        │  └────────────────────────────┘  │
        └──────────────────────────────────┘
```

Everything the CPU needs to resume process A exactly where it stopped lives in
A's kernel stack. Switching to B is then just: push A's state, pop B's.

---

# Formal Definition

A **context switch** is the operation of saving the CPU execution context of
one task and restoring the CPU execution context of another, so that the
processor can switch between them.

| Aspect | Value |
|---|---|
| Input | Two tasks, both in memory, both resumable |
| Action | Save current registers, restore another's |
| Preserved | Memory contents, open files, address space |
| Not preserved | CPU registers, program counter, kernel stack pointer |
| Cost | Typically 1–10 µs on modern hardware |
| Frequency | Thousands to millions per second on a busy Linux system |
| Performed by | The kernel, in the scheduler |

---

# Context Switch vs Mode Switch

This is the most commonly confused pair.

| Aspect | Mode switch | Context switch |
|---|---|---|
| What changes | CPU privilege level (ring 3 ↔ ring 0) | Which task is running |
| Same process? | Yes — it stays the same program | No — a different task |
| Registers saved? | Only those the calling convention uses | The full register set + FPU/SIMD |
| Cost | ~1 ns (a few cycles) | ~1–10 µs (thousands of cycles) |
| Triggered by | `syscall`, interrupt, exception | Scheduler decision |
| Happens on | Every system call | Only when the scheduler runs |

```text
  MODE SWITCH (cheap, happens constantly)

     user code ──syscall──► kernel code ──sysret──► user code
        ring 3                  ring 0                ring 3
        same process throughout


  CONTEXT SWITCH (expensive, happens less often)

     Process A ──────scheduler──────► Process B
      (saved)                          (running)
```

> [!TIP]
> A system call normally involves a mode switch, not a context switch. It
> only becomes a context switch if, while in the kernel, the scheduler decides
> to run a *different* task on the way back.

---

# What Exactly Is Saved?

```text
  SAVED
  ──────
  General purpose registers  RAX RBX RCX RDX RSI RDI
                             RBP RSP R8..R15
  Instruction pointer        RIP
  Flags                     RFLAGS (IF, IOPL, arithmetic flags)
  Segment registers         CS SS DS ES FS GS
  FPU state                 x87 stack
  SIMD state                XMM0..XMM15, YMM, ZMM
  Kernel stack pointer      RSP
  FSBASE / GS base          per-thread TLS base
  Signal mask               which signals are blocked
  CPU feature state         XCR0, PKRU, AMX/AMX tiles
  Task_struct pointers      current, prev (implicit, in registers)


  NOT SAVED (it lives in memory, unaffected by the switch)
  ────────────────────────────────────────────
  Virtual address space      identical — the process does not move
  Physical memory            unchanged
  Open file descriptors      part of the task_struct, not the CPU state
  Program data               unchanged
  Kernel stack contents      the saved context lives here
```

## The size of the cost

```text
  x86-64 general registers:  16 × 8  bytes  =   128 bytes
  FPU/SIMD (XMM0-15):        16 × 16 bytes  =   256 bytes
  AVX-512 ZMM:               32 × 64 bytes  =  2048 bytes
  RFLAGS, CS, SS, RIP, RSP, segment bases        ~  64 bytes
  ───────────────────────────────────────────────────────
  Minimum bytes written + read per switch:     ~  450 bytes
  With AVX-512 state saved:                   ~ 2.4 kB

  At 3 GHz, 2.4 kB written and read ≈ 1–2 µs
  Plus cache pollution, TLB effects, and branch mispredictions
```

The cost is not the arithmetic. It is the **cache and TLB damage**.

---

# Why Context Switches Are Expensive

## 1. Cache and TLB damage

This is the real cost, and it dominates.

```text
  Process A has warmed:
      L1d cache  ── A's working set
      L1i cache  ── A's hot loop
      L2/L3      ── A's libraries
      TLB        ── A's page translations

  Switch to B:
      every cache line A touched is now cold
      every TLB entry for A is evicted

  When A runs again:
      ~hundreds of ns of stall on every cache miss it takes to rebuild state
```

This is why a workload with many small context switches can be an order of
magnitude slower than the same work in one long slice.

## 2. Pipeline and branch effects

The indirect jump to the new task's code is a hard-to-predict branch. The
pipeline flushes, and the CPU cannot start fetching the new code until the
target is resolved.

## 3. Indirect costs in the kernel

A context switch is never just a context switch:

```text
  scheduler runs
     ├── update runqueue data structures
     ├── run the scheduler's perf-domain / core-migration logic
     ├── perf events accounting
     ├── TLB flush if the address space changed
     ├── scheduler_domain / core-migration logic
     ├── possibly IPI to another CPU
     └── memory barrier / rseq notification
```

```c
/* Kernel-side, conceptual: core/sched/switchto.c */
static struct rq *finish_task_switch(struct task_struct *prev)
{
    /*
     * After the iret has put the new task on its own stack, it traps back
     * into the kernel to do bookkeeping. This is a real cost of switching
     * that never shows up in a user-space microbenchmark.
     */
    rq = this_rq();

    /* TLB flush if the mm changed */
    if (prev->mm != next->mm) {
        tlb_flush();
        mmdrop(prev->mm);
    }

    /* perf accounting */
    perf_event_task_sched_out(prev, next);

    /* rseq: tell the new task no rseq critical section is in progress */
    rseq_handle_notify_resume(next, prev);

    return rq;
}
```

---

# Triggers of a Context Switch

## 1. Preemption — the timer tick

```text
  A running ──timer interrupt──► kernel
                              ──scheduler decides B has less vruntime──► B running
                                                               A's state saved
```

Linux's tick is what prevents a CPU-bound process from starving everything
else. This is preemptive scheduling.

```bash
# See the scheduler's configured tick
cat /proc/sys/kernel/sched_latency_ns     # target wakeup latency, default 6 ms
cat /proc/sys/kernel/sched_min_granularity_ns
cat /proc/sys/kernel/sched_rt_runtime_us  # RT throttling, default 950000
```

## 2. Voluntary yield

```c
#include <sched.h>

void worker(void)
{
    for (int i = 0; i < 100; i++) {
        do_some_work();
        sched_yield();          /* "I am not urgent, someone else can run" */
    }
}
```

This is a *voluntary* switch. The process gave up the CPU on purpose.

## 3. Blocking on I/O

```c
read(fd, buf, n);        /* nothing to read → block → switch away */
```

Blocking is the most common trigger of all. A process that waits on a disk is
not wasting a CPU; it is *not running*, so another task gets the core.

## 4. A system call that reschedules

Many system calls end in a reschedule check:

```text
  read() completes
     → the kernel asks: "has another task been waiting long enough?"
     → if yes: schedule()
     → if no: iret straight back to userspace
```

## 5. Preemption of a kernel thread / RCU work

The kernel itself is preemptible (since 2.6). A long kernel task can be
preempted by a userspace task after `preempt_schedule()` points.

## 6. SMP load balancing

```bash
# The load balancer moves tasks between CPUs
grep -E 'migration|imbalance' /proc/schedstat | head
cat /proc/sys/kernel/sched_migration_cost_ns
```

---

# How Linux Performs a Context Switch

```text
   ┌─────────────── Scheduler ───────────────┐
   │                                         │
   │  1. Pick the next task from the runqueue │
   │     (CFS: the task with the smallest     │
   │      vruntime)                          │
   │                                         │
   │  2. Call context_switch(prev, next)     │
   │        │                                │
   │        ▼                                │
   │  3. save prev's kernel stack pointer     │
   │     in prev->stack                      │
   │                                         │
   │  4. switch_to(prev, next)  ← asm        │
   │        │                                │
   │        ├── push prev's callee-saved    │
   │        │   registers onto prev's stack  │
   │        │                                │
   │        ├── pop next's callee-saved      │
   │        │   registers from next's stack  │
   │        │                                │
   │        └── ret  ← lands on next's stack │
   │                                         │
   │  5. The new task eventually reaches     │
   │     iretq, restoring full user state    │
   │                                         │
   │  6. Traps back into the kernel for      │
   │     finish_task_switch() bookkeeping    │
   │                                         │
   │  7. Return to userspace                 │
   └─────────────────────────────────────────┘
```

## The key insight about `switch_to`

`switch_to` only swaps **callee-saved registers** and the kernel stack pointer.
It does *not* touch user-space registers.

Those are restored later, by `iretq`, which pops the user frame that the
interrupt/trap entry had pushed. This is a deliberate two-stage design, and it
is why a task that is switched out mid-syscall resumes correctly inside the
kernel.

```text
  Userspace                      Kernel
  ─────────                      ──────
  A's RIP/RSP in regs
       │
       │  syscall  ──────────────►  pushes full user frame
       │                            (RIP, CS, RFLAGS, RSP, SS)
       │                            onto A's kernel stack
       │                                 │
       │                            decide to switch to B
       │                                 │
       │                            switch_to(): save callee-saved
       │                            for A on A's stack, load B's,
       │                            jump to B's kernel stack
       │                                 │
       │                            B's iretq pops B's user frame
       ▼                                 ▼
  A is now IN KERNEL, not in user mode
```

---

# Measuring Context Switches

## 1. From shell

```bash
# The system-wide counter (all CPUs, cumulative since boot)
grep ctxt /proc/stat
# ctxt 128473992

# Take two samples 10 seconds apart
awk '/^ctxt/ {print $2}' /proc/stat; sleep 10; awk '/^ctxt/ {print $2}' /proc/stat
```

```bash
# Per-context, from ps (nvcsw = voluntary, nivcsw = involuntary)
ps -o pid,comm,nvcsw,nivcsw -p $$
```

```text
    PID COMMAND      VOL NV
  4821 bash        38214 1042
```

- **Voluntary** (`nvcsw`): the task gave up the CPU. Almost always correct.
- **Involuntary** (`nivcsw`): the task was preempted. High values mean
  contention or an oversubscribed CPU.

## 2. Per-thread

```bash
# Voluntary context switches for each thread of a process
for t in /proc/$$/task/*; do
    printf "%s %s\n" "$(basename "$t")" "$(cat "$t"/status | grep voluntary_ctxt_switches | cut -f2)"
done
```

## 3. Real time

```bash
# Context switches per second, sampled
vmstat 1 5
```

```text
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free  buff  cache   si  so   bi  bo  in   cs  us sy id wa
 3  1      0 1200000  4000 8900000   0   0  32  48 1840 4210  8  3 82  7
 2  0      0 1199000  4000 8901000   0   0   0   0 2100 3890  6  3 91  0
```

`cs` = context switches per second. A workload doing 50,000 `cs` per second is
likely spending most of its life in scheduler overhead, not in real work.

## 4. In C, precisely

```c
#define _GNU_SOURCE
#include <stdio.h>
#include <time.h>
#include <unistd.h>
#include <sys/syscall.h>
#include <sched.h>

/* Read a counter straight from /proc — no dependency on procps. */
static long read_counter(const char *path, const char *key)
{
    FILE *f = fopen(path, "r");
    if (!f) return -1;

    char line[256];
    long val = -1;
    while (fgets(line, sizeof line, f)) {
        if (sscanf(line, key, &val) == 1)
            break;
    }
    fclose(f);
    return val;
}

int main(void)
{
    struct timespec t0, t1;
    char key[64];

    snprintf(key, sizeof key, "ctxt %ld");
    long c0 = read_counter("/proc/stat", key);

    clock_gettime(CLOCK_MONOTONIC, &t0);

    /* 1. Spin WITHOUT yielding: one task, so no context switches. */
    volatile long acc = 0;
    for (long i = 0; i < 100000000L; i++) acc += i;

    clock_gettime(CLOCK_MONOTONIC, &t1);
    long c1 = read_counter("/proc/stat", key);

    double spin_secs = (t1.tv_sec - t0.tv_sec) + (t1.tv_nsec - t0.tv_nsec) / 1e9;
    printf("spin loop  : %.3f s, %ld context switches\n",
           spin_secs, c1 - c0);

    /* 2. Same work, but yield every iteration. */
    c0 = c1;
    clock_gettime(CLOCK_MONOTONIC, &t0);
    for (long i = 0; i < 100000000L; i++) { acc += i; sched_yield(); }
    clock_gettime(CLOCK_MONOTONIC, &t1);
    c1 = read_counter("/proc/stat", key);
    double yield_secs = (t1.tv_sec - t0.tv_sec) + (t1.tv_nsec - t0.tv_nsec) / 1e9;
    printf("yield loop : %.3f s, %ld context switches\n",
           yield_secs, c1 - c0);
    printf("acc = %ld\n", acc);
    return 0;
}
```

```bash
gcc -O2 -o ctxcost ctxcost.c
./ctxcost
```

```text
spin loop  : 0.104 s, 0 context switches
yield loop : 92.417 s, 100000 context switches
```

`sched_yield()` 100,000 times cost 92 seconds. At ~900 µs per switch, that is
the price of switching every single iteration.

> [!NOTE]
> The spin loop shows **zero** context switches, not because it was fast, but
> because nothing else on the machine needed the CPU. Under load the same loop
> would be preempted thousands of times.

## 5. Measuring the cost of one switch, in C

```c
#define _GNU_SOURCE
#include <pthread.h>
#include <stdio.h>
#include <time.h>
#include <stdatomic.h>

static atomic_int turn = 0;
static const long N = 200000;

static void *ping_pong(void *arg)
{
    for (long i = 0; i < N; i++) {
        while (atomic_load_explicit(&turn, memory_order_acquire) == 0)
            ;                                   /* busy wait */
        atomic_store_explicit(&turn, 0, memory_order_release);
    }
    return NULL;
}

int main(void)
{
    pthread_t a, b;
    pthread_create(&a, NULL, ping_pong, NULL);
    pthread_create(&b, NULL, ping_pong, NULL);

    struct timespec t0, t1;
    clock_gettime(CLOCK_MONOTONIC, &t0);
    pthread_join(a, NULL);
    pthread_join(b, NULL);
    clock_gettime(CLOCK_MONOTONIC, &t1);

    double ns = (t1.tv_sec - t0.tv_sec) * 1e9 + (t1.tv_nsec - t0.tv_nsec);
    printf("%ld round-trips in %.3f ms\n", N, ns / 1e6);
    printf("%.0f ns per round trip (2 context switches + 2 IPIs)\n", ns / N);
    return 0;
}
```

```bash
gcc -O2 -pthread -o ctxping ctxping.c
./ctxping
```

```text
200000 round-trips in 1284.113 ms
6419 ns per round trip (2 context switches + 2 IPIs)
```

This number is large because both threads are on *different* CPUs, so each
hand-off needs an **inter-processor interrupt** to wake the other core. Two
threads pinned to the same CPU with `sched_setaffinity` are dramatically faster.

---

# The Cost of Too Many Context Switches

## Real examples

| Situation | Effect |
|---|---|
| Thread pool with queue size 1, 1000 threads | Every task switches twice; throughput collapses |
| Signal handler doing 1 ms of work, 10,000 signals/s | 10 s/s of CPU spent in handlers |
| Ping-pong synchronisation between 2 cores | Cache-line transfer, IPI latency |
| 1000 processes polling `read()` on the same fd | Thundering herd; switch rate in the millions |

## Diagnosing it

```bash
# Switch rate way up, CPU idle? Preemption storm.
vmstat 1 10
#   cs = 300000+, r = 8, us = 5, id = 90

# Which task is switching most?
top -H -p $(pidof myapp)
# watch the "voluntary_ctxt_switches" in /proc/PID/task/TID/status

# How many runnable tasks?
cat /proc/loadavg
# 12.00 12.00 12.00 8/1200 30000
#    ^ 1200 runnable or uninterruptible tasks for 8 CPUs
```

A load average far above the core count means every task is waiting for a core
and the system is spending its time switching rather than working.

---

# Context Switching in Threads

The mechanism is identical. Only the scope of what is shared changes.

| | Process switch | Thread switch |
|---|---|---|
| Saves | full register set | full register set |
| Switches | address space (CR3) | does not switch CR3 |
| Cost | ~1–10 µs | ~0.5–2 µs |
| Cache damage | full | smaller — same address space, shared data |
| TLB | flushed if mm changed | not flushed |

```text
  Process A  ──switch──►  Process B
    CR3 = A's tables        CR3 = B's tables   ← TLB must be flushed

  Thread 1   ──switch──►  Thread 2  (same process)
    CR3 unchanged throughout                    ← TLB stays warm
```

The cost difference is why threads are cheaper to switch than processes: the
TLB and the page tables do not have to change, and the two threads share a warm
cache for their code.

---

# Common Misconceptions

### ❌ "A context switch moves a process from RAM to another part of RAM."

Incorrect.

Nothing is moved. The process's address space is exactly where it was. Only the
CPU's register set, instruction pointer, and stack pointer change.

---

### ❌ "A context switch is the same as a mode switch."

Incorrect.

A mode switch is ring 3 → ring 0, costing a few cycles, and the same process
continues. A context switch is one task → another task, costing microseconds.

---

### ❌ "Context switching is free; the CPU would be idle otherwise."

Incorrect.

The switch itself takes real time, during which no user work is done, and it
destroys cache and TLB warmth for both tasks. Too many switches reduces total
throughput even when the CPU is nominally 100% "busy".

---

### ❌ "Voluntary context switches are a performance problem."

Incorrect, and the opposite is usually true.

A voluntary switch means the task blocked on I/O or yielded. That is the
scheduler working as designed. **Involuntary** switches are the ones that
indicate contention.

---

### ❌ "Threads are always cheaper than processes to switch."

Generally yes, but not always. If the two threads are on different cores, each
switch needs an inter-processor interrupt, which can cost more than the TLB
flush you saved.

---

# Interview Questions

### Basic

- What is a context switch? What is saved and restored?
- Differentiate between a context switch and a mode switch.
- What triggers a context switch?
- Why is a context switch expensive?

### Intermediate

- How does Linux's `switch_to` differ from the full context switch?
- Why does a context switch hurt caches and the TLB?
- What are voluntary and involuntary context switches, and what do they tell you?
- Why is a thread switch cheaper than a process switch?

### Advanced

- Explain the two-stage nature of a Linux context switch: `switch_to` and then `iretq`.
- How would you measure the cost of a single context switch in C?
- Why can ping-pong synchronisation between two cores cost microseconds?
- How does the load balancer interact with cache locality, and why does CPU affinity sometimes help?
- When would pinning a thread to a core make a program *slower*?

---

# University Exam Notes

### Definitions

- **Context switch:** Saving the CPU state of one process and restoring the CPU
  state of another so the processor can resume a different task.
- **Mode switch:** A change of CPU privilege level between user mode and kernel
  mode, within the same process.
- **Voluntary context switch:** A switch in which the task blocked or yielded
  voluntarily.
- **Involuntary context switch:** A switch in which the task was preempted
  because another task needed the CPU.
- **Preemption:** The kernel forcibly reclaiming the CPU from a running task.

### Frequently Asked Questions

- What is a context switch? Explain with a diagram.
- Differentiate between mode switch and context switch.
- What are the steps involved in a context switch?
- What causes a context switch?
- Why is a context switch expensive?
- What are voluntary and involuntary context switches?
- Explain the CFS scheduler's use of virtual runtime.

---

# Key Takeaways

- A context switch is a change of **which task the CPU runs**; the processes
  themselves never move.
- It saves the full register set, RIP, flags, segment registers, FPU/SIMD
  state, and the kernel stack pointer — roughly 450 bytes to 2.4 kB.
- A **mode switch** is cheap (a few cycles) and stays inside one process.
  A **context switch** costs microseconds and crosses between processes.
- The cost is dominated by **cache and TLB damage**, not by arithmetic.
- Triggers: the timer tick (preemption), `sched_yield()`, blocking I/O,
  a reschedule check at the end of a syscall, and SMP load balancing.
- Linux splits the work: `switch_to()` swaps kernel stack and callee-saved
  registers; `iretq` later restores the full user-space state.
- Measure with `/proc/stat`'s `ctxt`, `vmstat`'s `cs` column, and the
  `nvcsw`/`nivcsw` fields in `ps`.

---

# References

- Operating System Concepts — Silberschatz
- Modern Operating Systems — Tanenbaum and Bos
- The Linux Programming Interface — Kerrisk
- Linux Kernel Development — Robert Love (see `schedule()` and `switch_to`)
- `man 2 sched_yield`, `man 2 sched_setaffinity`
- `/proc/stat`, `/proc/PID/status`, `/proc/loadavg`, `vmstat(1)`
