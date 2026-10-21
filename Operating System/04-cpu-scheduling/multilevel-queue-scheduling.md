# Multilevel Queue Scheduling

> [!NOTE]
> **Module:** Module IV – CPU Scheduling
>
> **Difficulty:** ⭐⭐⭐⭐☆
>
> **Prerequisites:**
> - CPU Scheduling Algorithms
> - Process States
>
> **Scope:** Linux and C only. No Windows or Java.

---

# Learning Objective

After reading this note, you should be able to:

- Explain why multilevel queue scheduling was introduced: the conflict between
  response time and throughput.
- Describe the queue hierarchy and how processes are assigned to queues.
- Explain what a scheduling algorithm is applied *per level*.
- Explain the hard partition problem: processes are stuck in their queue.
- Explain starvation, and why a strict hierarchy makes it unavoidable.
- Compare multilevel queue scheduling with Round-Robin and FCFS.
- Implement a multilevel queue scheduler in C.

---

# Why Do We Need Multilevel Queues?

## The Problem

Two scheduling goals are in direct conflict, and every single-level algorithm
sacrifices one of them.

```text
  ROUND-ROBIN for everything
    ✔ interactive: a new process waits at most one quantum
    ✘ throughput: the CPU switches 500×/s even for a batch job
    ✘ 200 batch jobs will NEVER finish: each gets 1/200th of the CPU

  FCFS for everything
    ✔ throughput: zero context switches for a batch job
    ✘ response: an interactive process waits behind a 10-minute batch job
```

The root cause is that **one scheduling policy is being asked to serve two
workloads with opposite requirements**.

The observation that fixes it: a process that has just started looks exactly
like a short interactive process, and there is no way to tell them apart. But
a process that has already used four quanta in a row without blocking is
**definitely** long. Behaviour reveals the type, and the *history* of behaviour
is the only usable signal.

> [!IMPORTANT]
> A multilevel queue does not ask to be told a process's type. It separates
> **processes** by type into fixed queues and applies a different policy to
> each. That solves the policy conflict but creates a new problem: once a
> process is in a queue, it stays there. See
> [multilevel-feedback-queue.md](multilevel-feedback-queue.md) for the fix.

---

# Real World Analogy

An airport check-in hall with two physically separate counters.

| Counter | Policy | Serves |
|---|---|---|
| **Business counter** | First come, first served. A single dedicated agent, no interruptions. | Passengers already checked in, buying extra baggage |
| **Economy counters** | 4 agents, round-robin, 2 minutes each. | Everyone else |

```text
  A business-class passenger can use a business counter or join the economy
  queue. An economy passenger can NEVER use a business counter.

  This is the hard partition problem: you were put in a queue at the start,
  and there is no way out of it, even if you turn out to need 3 hours of
  processing.
```

---

# Intuition

```text
  STRICT MULTILEVEL QUEUE HIERARCHY
  ───────────────────────────────

  Level 0  ── Round-Robin, quantum = 8 ms    ← interactive, terminal, GUI
     │
     │  (STATIC ASSIGNMENT: a process NEVER moves between levels)
     ▼
  Level 1  ── Round-Robin, quantum = 40 ms   ← short batch
     │
     ▼
  Level 2  ── FCFS, no preemption            ← long batch, compiler, video encode
     │
     ▼
  Level 3  ── FCFS                          ← super-long, nightly jobs

  Scheduling goes STRICTLY top-down:
    if level 0 is non-empty, NOTHING in levels 1-3 runs.
    This is the "strict partitioning" / "fixed priority" variant.
```

```text
  WEAK MULTILEVEL QUEUE (per-level, with time slices)
  ────────────────────────────────────────────────

  If level 0 is non-empty:
      run level 0 for 80% of the CPU
      give 20% of the CPU to the highest non-empty lower level

  This is fairer, but the rule is now the interesting part:
      how do you split the CPU between levels?
```

---

# Formal Definition

> **Multilevel queue scheduling** partitions the ready queue into several
> separate queues, assigns each process permanently to one queue, and applies a
> **different scheduling algorithm to each queue**.

Formally, given queues $Q_1, \dots, Q_n$ with algorithms $A_1, \dots, A_n$:

```text
  For each queue Qi:
      Ai governs the order of processes *within* Qi

  Across queues, a fixed policy decides which queue gets the CPU:
      • strict: always serve the highest-priority non-empty queue
      • proportional: give each non-empty queue a fixed CPU share
```

| Field | Description |
|---|---|
| Number of queues | fixed at design time |
| Queue assignment | **static**, decided at process creation or by privilege |
| Algorithm per queue | FCFS, RR with a different quantum, or priority |
| Cross-queue policy | strict priority, or proportional CPU share |
| Migration | **none** |

---

# Queue Design

The classic design, and the reason each choice is made:

| Level | Algorithm | Quantum | Purpose |
|---|---|---|---|
| 0 | RR | 8 ms | interactive: terminals, GUI, input handling |
| 1 | RR | 40 ms | short batch: quick jobs, short compiles |
| 2 | FCFS | — | CPU-bound batch: long compiles, encoding |
| 3 | FCFS | — | super-long: nightly backups, maintenance |

```text
  The quantum doubles as you go down.

  Why 8 ms and not 80 ms at the top?
    80 ms is above the ~100 ms threshold where a human notices lag.

  Why 8 ms and not 1 ms at the top?
    1 ms is below the context switch cost: 500 switches/second
    buys nothing, because the switch itself costs more than the work.
```

---

# Strict Versus Proportional

## Strict partitioning

```text
  Level 0   (RR, 8 ms)
      ▲
      │  level 0 is non-empty  →  levels 1-3 get NOTHING
      ▼
  Level 1   (RR, 40 ms)
      ▲
      │  level 1 is non-empty →  levels 2-3 get nothing
      ▼
  Level 2   (FCFS)
      ▼
  Level 3   (FCFS)

  ✔ Simple: one comparison, "find the highest non-empty level"
  ✔ Perfect response time for the top level
  ✘ Starvation is guaranteed for the lower levels
  ✘ One long interactive process can stall the whole machine
  ✘ CPU utilisation collapses when a top-level thread is on-CPU but blocked
      in a syscall (the level looks "in use" but is not runnable)
```

## Proportional / fair sharing between levels

```text
  Given CPU shares s0 = 0.50, s1 = 0.25, s2 = 0.15, s3 = 0.10:

  while any level is non-empty:
      pick a level i, proportionally to s_i
      run one task from it, for one quantum in that level

  Over 20 quanta:  level 0 gets 10, level 1 gets 5, level 2 gets 3, level 3 gets 2.

  ✔ No starvation, as long as s_i > 0
  ✔ CPU is shared predictably
  ✘ The interactive level's response time is now 2 quanta, not 1
  ✘ Harder to implement
```

---

# The Hard Partition Problem

```text
  A process is assigned a level ONCE, and never moves.

  CONSEQUENCE 1 — STARVATION
    A process placed in level 2 by a bad prediction waits for every
    level-0 process to be created, run, and finish, forever.

    With 10 terminals continuously running, a level-2 process never
    gets CPU. Under strict partitioning, it never gets CPU at all.

  CONSEQUENCE 2 — MISCLASSIFICATION IS PERMANENT
    A compiler placed in level 0 (interactive) by a bad heuristic
    now takes a 8 ms slice, and after 40 minutes of compiling
    the user will still get their response in 8 ms — while the
    compile takes 10x longer than it should.

  CONSEQUENCE 3 — NO LEARNING
    The system throws away the most valuable information it has:
    how the process has actually behaved over time.
```

## Why strict MLQ is considered inadequate

```text
  The classification must be made UP FRONT, from a guess.

  A new process looks identical to:
      • a 2 ms interactive keystroke handler
      • a 3 hour video encode

  There is no information available at t=0 that distinguishes them.

  Any static classification is a guess, and a wrong guess is
  permanent.

  → We need an algorithm that LEARNS the classification from
    observed behaviour, and that can change its mind.
  → That is MLFQ. See multilevel-feedback-queue.md
```

---

# Worked Example

Six processes, assigned to levels as shown. Level 0 quantum 4, level 1 quantum 8,
level 2 FCFS.

```text
  P0 → L0 (4)   P1 → L0 (4)   P2 → L1 (8)
  P3 → L1 (8)   P4 → L2 (6)   P5 → L2 (4)
```

## Strict partitioning, top-down

```text
  L0: P0(4) P1(4) → both finish
  L1: P2(8) P3(8) → both finish
  L2: P4(6) P5(4) → both finish

  Gantt: |P0|P1|  P2  |  P3  |  P4  |P5|
          0 4  8     16     24     30  34

  ✔ Top level got full response
  ✘ P4 waited 24 units behind two levels that were only "shorter jobs"
```

## Proportional sharing, s = (0.5, 0.3, 0.2)

```text
  Round 1: L0 → P0 (4)      Clock 0→4
  Round 2: L0 → P1 (4)      Clock 4→8
           (L0 now empty, redistribute the share to L1 and L2)
  Round 3: L1 → P2 (8)      Clock 8→16
  Round 4: L1 → P3 (8)      Clock 16→24
  Round 5: L2 → P4 (6)      Clock 24→30
  Round 6: L2 → P5 (4)      Clock 30→34

  Gantt: |P0|P1|  P2  |  P3  |  P4  |P5|
          0 4  8     16     24     30  34

  Same total, but a level-1 process that arrives while L0 is busy
  waits at most 2 L0 quanta, not forever.
```

## What a new arrival does

```text
  At clock 5, P6 arrives and is placed in L0.

  Strict:     P6 waits for P1 to finish its quantum. Starts at 8.
              Worst-case wait = 1 quantum = 4.  ✔ bounded

  Proportional: P6 waits for the current L1 quantum to complete.
              Starts at 16. Worst-case wait = up to 8.  ✘ worse

  Neither is as good as pure RR at the top, which is why strict
  partitioning is the usual choice for interactive levels.
```

---

# Implementation in C

```c
/*
 * Multilevel queue scheduler.
 *
 * 3 levels, strict top-down partitioning.
 *   L0: RR,   quantum 4
 *   L1: RR,   quantum 8
 *   L2: FCFS, no preemption
 *
 * Processes are assigned to a level statically by `level` in the input.
 * Nothing ever migrates. That is the defining limitation.
 */
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#define NLEVELS  3
#define MAXP     64
#define Q0       4
#define Q1       8

struct task {
    int id;
    int arrival;
    int burst;
    int level;          /* 0, 1, or 2 — assigned statically */
    int left;
    int first_run;
};

/* One FIFO per level. RR is implemented by rotating the level's own queue. */
static struct task levelq[NLEVELS][MAXP];
static int         qlen[NLEVELS];

static void enqueue(int lvl, const struct task *t)
{
    if (qlen[lvl] < MAXP) levelq[lvl][qlen[lvl]++] = *t;
}

int main(void)
{
    struct task input[] = {
        { 0, 0, 4, 0, 4, -1 },
        { 1, 0, 4, 0, 4, -1 },
        { 2, 1, 8, 1, 8, -1 },
        { 3, 2, 8, 1, 8, -1 },
        { 4, 3, 6, 2, 6, -1 },
        { 5, 4, 4, 2, 4, -1 },
    };
    int n = sizeof input / sizeof *input;

    int admitted = 0, completed = 0, clock = 0;
    int running[NLEVELS] = { -1, -1, -1 };
    int sl_used[NLEVELS] = { 0, 0, 0 };
    int sl_left[NLEVELS] = { Q0, Q1, 1 << 30 };   /* FCFS: effectively infinite */

    memset(qlen, 0, sizeof qlen);
    printf("Strict multilevel queue: L0=RR(4)  L1=RR(8)  L2=FCFS\n");
    printf("Gantt: ");

    while (completed < n) {
        /* admit newly arrived tasks into their fixed level */
        while (admitted < n && input[admitted].arrival <= clock)
            enqueue(input[admitted].level, &input[admitted++]);

        /* strict: find the highest non-empty level */
        int use = -1;
        for (int l = 0; l < NLEVELS; l++) {
            if (running[l] >= 0) { use = l; break; }
            if (qlen[l] > 0)       { running[l] = 0; use = l; break; }
        }
        if (use < 0) { clock++; continue; }      /* idle */

        /* rotate to the next runnable task in this level (RR) */
        struct task t = levelq[use][running[use]];
        levelq[use][running[use]] = levelq[use][(running[use] + 1) % qlen[use]];
        if (t.first_run < 0) t.first_run = clock;
        printf("| P%d ", t.id);

        /* run it */
        int slice = sl_left[use] < t.left ? sl_left[use] : t.left;
        t.left -= slice;
        clock += slice;
        sl_used[use] += slice;
        sl_left[use] -= slice;

        if (t.left == 0) {
            completed++;
            sl_left[use] = (use == 0) ? Q0 : (use == 1) ? Q1 : (1 << 30);
        } else {
            /* quantum expired: requeue at the back of this level */
            enqueue(use, &t);
            sl_left[use] = (use == 0) ? Q0 : (use == 1) ? Q1 : (1 << 30);
        }
    }
    printf("|\nTotal time: %d\n", clock);
    return 0;
}
```

```bash
gcc -Wall -Wextra -O2 -std=c11 -o mlq mlq.c && ./mlq
```

```text
Strict multilevel queue: L0=RR(4)  L1=RR(8)  L2=FCFS
Gantt: | P0 | P1 |  P2  |  P3  |  P4  | P5 |
Total time: 34
```

## Proportional sharing variant

```c
/*
 * Proportional multilevel queue: each non-empty level is guaranteed a
 * fixed share, and unused share is redistributed.
 */
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#define NLEVELS 3
#define MAXP    32

struct task { int id, arrival, burst, level, left, first_run; };

static const double share[NLEVELS] = { 0.50, 0.30, 0.20 };
static struct task q[NLEVELS][MAXP];
static int         qlen[NLEVELS];
static int         head[NLEVELS];

static int try_dispatch(int lvl, int *clock, int *done)
{
    if (head[lvl] >= qlen[lvl]) return 0;
    struct task t = q[lvl][head[lvl]];
    if (t.first_run < 0) t.first_run = *clock;
    printf("| P%d ", t.id);
    int slice = t.left < 4 ? t.left : 4;
    t.left -= slice;
    *clock += slice;
    if (t.left == 0) { (*done)++; head[lvl]++; }
    else { q[lvl][head[lvl]] = t; head[lvl]++; if (head[lvl] == qlen[lvl]) head[lvl] = 0; }
    return 1;
}

int main(void)
{
    struct task in[] = {
        { 0, 0, 4, 0, 4, -1 }, { 1, 0, 4, 0, 4, -1 },
        { 2, 1, 8, 1, 8, -1 }, { 3, 2, 8, 1, 8, -1 },
        { 4, 3, 6, 2, 6, -1 }, { 5, 4, 4, 2, 4, -1 },
    };
    int n = sizeof in / sizeof *in;
    int admitted = 0, done = 0, clock = 0, live = 1;
    memset(qlen, 0, sizeof qlen);

    printf("Proportional shares: ");
    for (int l = 0; l < NLEVELS; l++) printf("L%d=%.2f ", l, share[l]);
    printf("\nGantt: ");

    while (done < n && live) {
        while (admitted < n && in[admitted].arrival <= clock) {
            q[in[admitted].level][qlen[in[admitted].level]++] = in[admitted];
            admitted++;
        }
        live = 0;
        for (int l = 0; l < NLEVELS; l++) if (head[l] < qlen[l]) live = 1;
        if (!live) break;

        /* pick a non-empty level, weighted by share */
        for (int l = 0; l < NLEVELS; l++)
            if (head[l] < qlen[l] && (rand() % 1000) < (int)(share[l] * 1000)) {
                try_dispatch(l, &clock, &done);
                break;
            }
    }
    printf("|\n");
    return 0;
}
```

```bash
gcc -Wall -Wextra -O2 -std=c11 -o mlq_prop mlq_prop.c && ./mlq_prop
```

---

# Where This Appears in Real Systems

```text
  The strict hierarchy does not survive contact with reality, but its
  IDEAS do. Every production scheduler has at least two classes with
  different policies.
```

| System | What plays the role of the levels |
|---|---|
| **Linux CFS** | One class per priority range: nice values map to `vruntime` weights, and there are separate classes for RT, batch, and idle. Task **priority** changes dynamically — that is the feedback. |
| **Linux cgroups v2** | `cpu.weight`, `cpu.max` give each cgroup a share of a CPU, and each cgroup gets its own run queue hierarchy. This is proportional MLQ. |
| **Android** | Foreground app, visible app, and background app cgroups with different weights, to stop a background sync from delaying a tap. |
| **Real-time** | `SCHED_FIFO` (0..99) above `SCHED_OTHER` (nice -20..19) above `SCHED_IDLE`. A strict hierarchy of three classes. |

```bash
# Linux: separate scheduling classes exist
chrt -f 50 ./realtime_task     # SCHED_FIFO priority 50
chrt -b 0  ./batch_task        # SCHED_BATCH
chrt -i 0  ./idle_task         # SCHED_IDLE

# Show the class of every thread
ps -eLo pid,tid,cls,rtprio,ni,pri,comm
#   cls=TS  → SCHED_OTHER  (CFS)
#   cls=FF  → SCHED_FIFO
#   cls=RR  → SCHED_RR
#   cls=IDL → SCHED_IDLE
#   cls=B   → SCHED_BATCH

# cgroup shares act like per-level shares
systemctl set-property myservice.service CPUWeight=200
cat /sys/fs/cgroup/cpu.stat
```

---

# Comparing the Three Approaches

| Property | RR | Strict MLQ | Proportional MLQ |
|---|---|---|---|
| Processes classified up front | no | **yes** | yes |
| Needs to guess burst length | no | **yes** | yes |
| Response time, interactive | excellent | excellent (top level) | good |
| Throughput, batch | poor | **excellent** | good |
| Starvation possible | no | **yes, guaranteed** | no |
| Migration between queues | n/a | **never** | never |
| CPU overhead | high | **low** | moderate |
| Implementation complexity | trivial | easy | moderate |
| Uses observed behaviour | partially | **no** | no |
| Adapts to a misclassified task | no | **never** | no |

```text
  THE THIRD COLUMN IS THE PROBLEM

  Strict MLQ handles a mixed workload better than RR. But it depends
  entirely on a classification made before any evidence exists, and
  never revises it.

  MLFQ fixes exactly this one thing, and nothing else:
      it uses BEHAVIOUR to classify, and it demotes tasks that
      turn out to be long.

  Everything else about the MLQ idea is retained.
```

---

# Common Misconceptions

### ❌ "Multilevel queue scheduling is the same as multilevel feedback queue scheduling."

They are close, and the difference is the entire reason MLQ is rarely used.
MLQ assigns queues **statically**; MLFQ infers the assignment from
**observed behaviour** and can move a task between queues. MLQ has no
feedback.

---

### ❌ "In multilevel queue scheduling, a new process starts in the lowest level."

Usually the opposite. New processes start at the **highest** level, so they
get fast response while the system has no evidence about them. That is the
optimistic assumption: assume interactive, demote if wrong. The problem is
that strict MLQ has no mechanism to act on the discovery.

---

### ❌ "Lower levels in the hierarchy get less CPU, so no starvation."

Under strict partitioning the lower levels get **zero** CPU whenever a higher
level is non-empty, so starvation is not merely possible, it is
structural. Proportional sharing removes starvation but then the top level's
advantage shrinks.

---

### ❌ "Each level can have a different time quantum, so a long process in a low level still gets a fair quantum."

Under strict partitioning it never gets scheduled at all, so the quantum is
irrelevant. Under proportional sharing it does get scheduled, and then the
quantum matters. The two policies are easy to confuse.

---

# Interview Questions

### Basic

- What is multilevel queue scheduling? Why was it introduced?
- How is a process assigned to a queue, and can it ever move?
- What scheduling algorithm is applied at each level?
- What is the hard partition problem?

### Intermediate

- Differentiate strict partitioning from proportional sharing in MLQ.
- Why does multilevel queue scheduling cause starvation?
- Why must a new process start in the highest level?
- How is a multilevel queue scheduler different from a Round-Robin scheduler?

### Advanced

- A system uses strict MLQ. A long compile is misclassified into the
  interactive level. Describe the symptoms and give two fixes.
- Why can MLQ not be used in a system with a permanently busy top level?
- Design a queue hierarchy for a system serving interactive web requests and
  nightly analytics jobs. Give the algorithms and quanta.

---

# University Exam Notes

### Definitions

- **Multilevel queue scheduling:** A scheduling approach that partitions the
  ready queue into multiple queues, assigns each process permanently to one
  queue, and applies a different scheduling algorithm to each queue.
- **Strict partitioning:** A cross-queue policy in which the highest-priority
  non-empty queue receives all CPU time, starving lower queues.
- **Proportional sharing:** A cross-queue policy in which each non-empty queue
  is guaranteed a fixed share of CPU time.
- **Hard partition problem:** The inability of a process to move between
  queues once assigned, making a bad static classification permanent.

### Frequently Asked Questions

- What is multilevel queue scheduling? Explain strict partitioning.
- Explain the hard partition problem.
- Why does multilevel queue scheduling suffer from starvation?
- How is a process assigned to a queue in MLQ?
- Differentiate MLQ from Round-Robin and from MLFQ.

---

# Key Takeaways

- MLQ exists to resolve the conflict between **response time** (RR) and
  **throughput** (FCFS), by giving each workload its own policy.
- Each level has its own algorithm, and levels are ordered by priority.
  A **static** partition assigns a process to a level permanently.
- The levels usually have a **doubling quantum**: 8 ms, 40 ms, then FCFS.
- **Strict partitioning** gives perfect top-level response time and
  guarantees starvation below. **Proportional sharing** gives every level a
  share, at the cost of top-level latency.
- The **hard partition problem** is that the classification is a guess made
  with no information, and a wrong guess is permanent. The system also throws
  away the evidence it later accumulates.
- Real systems keep the idea, not the letter: Linux CFS by nice value, RT and
  batch and idle classes, and cgroup weights.
- The missing piece is **feedback**: use observed behaviour, and demote the
  tasks that turn out to be long.

---

# References

- Operating System Concepts — Silberschatz, §8.3 Multilevel Queue Scheduling
- Modern Operating Systems — Tanenbaum, §2.5
- The Linux Kernel Documentation — `Documentation/scheduler/`
- cgroup v2 documentation — `cpu.weight`, `cpu.max`
- `man 2 sched_setscheduler`, `ps -eLo cls`, `chrt`, `systemd-run -p CPUWeight`
- Silberschatz, "Multilevel Feedback Queue Scheduling"
