# Multilevel Feedback Queue (MLFQ)

> [!NOTE]
> **Module:** Module IV – CPU Scheduling
>
> **Difficulty:** ⭐⭐⭐⭐⭐
>
> **Prerequisites:**
> - Multilevel Queue Scheduling
> - CPU Scheduling Algorithms
>
> **Scope:** Linux and C only. No Windows or Java.

---

# Learning Objective

After reading this note, you should be able to:

- Explain the key idea: **the scheduler learns the job type by watching behaviour**.
- Describe the rules: new processes at the top, use the whole quantum, demote on
  exhaustion, periodic priority boost to prevent starvation.
- Explain the parameters `n`, `m`, and the boost period, and what each one
  controls.
- Explain the two failure modes: **convoy effect** (too few queues) and
  **thrashing** (too many boosts / too short a boost period).
- Explain MLFQ+ and the difference from vanilla MLFQ.
- Explain how the O(N) scan was avoided with a bitmask.
- Implement a full MLFQ scheduler in C and run the worked example.
- Relate the idea to Linux's `nice` value and cgroup weights.

---

# Why Do We Need MLFQ?

## The Problem

The two facts that make scheduling hard:

```text
  FACT 1  A 2 ms keystroke handler and a 3 hour video encode are
          indistinguishable at t = 0. Both are brand new processes
          with burst = "unknown".

  FACT 2  They become trivially distinguishable at t = 0.05 s.
          The handler BLOCKED. The encoder is still running.
```

A multilevel queue (see [multilevel-queue-scheduling.md](multilevel-queue-scheduling.md))
must be told the class up front, and can never revise it. MLFQ removes the need
to be told:

> **Run every new process optimistically in the top queue. If it keeps
> consuming its entire quantum, it is a long process — move it down. If it
> blocks early, it is short or interactive — leave it where it is.**

The answer to a question you cannot answer in advance is to **ask the system
and let the answer arrive over time**. That is the entire innovation.

---

# Real World Analogy

A restaurant that seats every newcomer at the best table by the window, and
moves them out when it becomes clear they are not going to leave quickly.

```text
  New customer → window table (best service, attentive waiter)
      │
      ├─ leaves after 10 minutes  →  gone. Never came back. ✔ short
      │
      └─ still eating after 20 minutes  →  move to the back patio
          (less attention, but there are now more of them)

  This is fair to everyone: the 10-minute customers get great service,
  and the 2-hour customers stop occupying the good tables.

  The manager also has a rule: "every table is re-checked every hour."
  A patron on the patio who has finished eating is cleared off, and a
  genuinely short customer can claim a window table again.
  Without that rule, the patio fills with long customers and the
  restaurant becomes unusable for new arrivals.
```

---

# Intuition

```text
  THE THREE OBSERVATIONS MLFQ EXPLOITS

  1. A process that BLOCKS before using its whole quantum is
     short or I/O-bound.  → keep it high

  2. A process that uses its WHOLE quantum, repeatedly, without
     blocking is CPU-bound.  → move it down

  3. A process that has waited a very long time may be about to
     finish, or may be interactive but unlucky.  → boost it up
     periodically, so starvation cannot happen

  Rules 1 and 2 are the feedback. Rule 3 is the fairness guarantee.
```

```text
  PRIORITY BY BEHAVIOUR, NOT BY GUESS
  ───────────────────────────────────

  a) The higher in the diagram, the HIGHER the priority
     (and the shorter the time slice).

       priority
          ▲
     L0   │ ████████████   quantum  8 ms   interactive
     L1   │ ░░░░░░░░░░░░   quantum 16 ms   short batch
     L2   │ ▒▒▒▒▒▒▒▒▒▒▒▒   quantum 32 ms   medium batch
     L3   │ ▓▓▓▓▓▓▓▓▓▓▓▓   FCFS          long batch
     L4   │ ████████████   FCFS          very long
          └──────────────────────────────► burst length / CPU-bound-ness
```

---

# Formal Definition

> **Multilevel feedback queue (MLFQ)** is a scheduling algorithm that uses
> several queues, each with its own time slice, and **dynamically moves
> processes between queues based on their observed CPU behaviour**. A process
> that repeatedly uses its whole quantum is demoted; a process that blocks early
> stays high.

## The four rules

| Rule | Statement |
|---|---|
| **R1** | If a new process arrives, place it at the **top** of the highest-priority queue. |
| **R2** | When a process runs, let it use **its entire time slice** (its queue's quantum). It is preempted only if it exhausts the slice. |
| **R3** | When a process exhausts its slice in a queue, **demote** it to the tail of the next lower queue. |
| **R4** | Periodically, **boost** every process to the top queue. |

> [!IMPORTANT]
> R1–R3 are the feedback mechanism. **R4 is what makes starvation impossible**
> and is the rule most often forgotten. Without it, a long process can be
> postponed forever by a steady stream of short ones, and the "fair" algorithm
> becomes the unfair one.

## Parameters

| Symbol | Meaning | Typical value | What it controls |
|---|---|---|---|
| **n** | number of queues (levels) | 3–8 | how precisely the system can distinguish job types |
| **m** | boost period, in quanta | ~10–100 | how long a long process can be punished |
| **quantum** | slice for queue *i* | doubling: 8, 16, 32, … | response time at the top, switch overhead at the bottom |

The usual design: the quantum **doubles** as you descend, because the processes
there are longer and a larger amortisation is fine, and because the switch
overhead is negligible relative to the slice.

---

# The Three Outcomes

## Interactive / I/O-bound process

```text
  Arrives in L0 with quantum 8.
  Uses 3 units of CPU, then blocks on read().

  R2: it did NOT use the whole quantum, so it is NOT demoted.
  It returns to the tail of L0, still at the top.

  It keeps bouncing in L0 all its life, always with 8 ms of CPU
  available before it must block.

  Result: a terminal stays fast no matter how long the session lasts.
```

## CPU-bound process

```text
  Arrives in L0 with quantum 8.
  Uses all 8. R3 → demote to L1, quantum 16.
  Uses all 16.       → demote to L2, quantum 32.
  Uses all 32.       → demote to L3, FCFS.

  It reaches the bottom quickly, and from then on it is treated as a
  long batch job with a big slice and almost no attention.

  Result: it stops disturbing the interactive processes, and the
  switch overhead is amortised over a long run.
```

## A process that needs a long time but runs in bursts

```text
  A compiler uses 400 ms of CPU, then writes an object file, then
  continues. Each burst is longer than the L0 quantum, so it gets
  demoted on every burst even though it is not hogging the CPU.

  This is the failure mode. R4 helps a little, and lowering the
  top-level quantum helps more. In practice this is why general-purpose
  schedulers do not use a strict quantum-based demotion, and prefer
  to weight by historical CPU utilisation instead (which is what
  Linux CFS does with `vruntime`).
```

---

# Avoiding the O(N) Scan

With *n* queues, a naive scheduler checks each queue for a runnable process on
every dispatch: `O(n)` per dispatch, and `O(n·m)` in the boost phase.

**The bitmask trick:** keep an integer where bit *i* is set if queue *i* is
non-empty.

```c
#define QUEUES 5

static struct task q[QUEUES][MAX_TASKS];
static int  head[QUEUES], tail_[QUEUES], qlen[QUEUES];
static unsigned long nonempty_mask;    /* bit i set ⟺ queue i has a task */

/* update after every enqueue/dequeue */
static void refresh_mask(void)
{
    nonempty_mask = 0;
    for (int i = 0; i < QUEUES; i++)
        if (qlen[i] > 0)
            nonempty_mask |= (1UL << i);
}

/* highest non-empty queue = highest set bit */
static int highest_nonempty(void)
{
    if (!nonempty_mask) return -1;
    return 63 - __builtin_clzl(nonempty_mask);    /* O(1) */
}
```

```text
  Before:  scan 5 queues      → 5 comparisons
  After:   __builtin_clzl      → 1 instruction (on x86: BSR / LZCNT)

  O(n) per dispatch  →  O(1) per dispatch.
  The same trick appears in Linux's own `prio_to_task[]` bitmap and
  in Linux's wake-up path.
```

---

# Full Implementation

```c
/*
 * Multilevel Feedback Queue scheduler.
 *
 * RULES
 *   R1  new process → top queue
 *   R2  run for the whole quantum of the current queue
 *   R3  quantum exhausted → demote one level
 *   R4  every `BOOST_PERIOD` ticks → boost everything to the top
 *
 * 3 queues, doubling quanta, periodic boost.
 */
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#define NQ        3
#define BOOST     20          /* R4: boost every 20 dispatches */
#define MAX_TASKS 64
static const int quantum[NQ] = { 2, 4, 8 };

struct task { int id, burst, left, arrival, level, first_run, runs; };

/* Round-robin FIFO per queue. */
static struct task q[NQ][MAX_TASKS];
static int         len[NQ], hd[NQ], tl[NQ];
static unsigned long mask;                 /* bit i set ⟺ queue i non-empty */

static void enqueue(int l, struct task t)
{
    q[l][tl[l]] = t;
    tl[l] = (tl[l] + 1) % MAX_TASKS;
    len[l]++;
    mask |= (1UL << l);
}

static struct task dequeue(int l)
{
    struct task t = q[l][hd[l]];
    hd[l] = (hd[l] + 1) % MAX_TASKS;
    len[l]--;
    if (len[l] == 0) mask &= ~(1UL << l);
    return t;
}

static int highest(void)
{
    if (!mask) return -1;
    return 63 - __builtin_clzl(mask);      /* O(1), no loop */
}

int main(void)
{
    /*
     * P0 P1: interactive   — long total, but they block often (shown as small bursts)
     * P2 P3: CPU-bound     — use their whole quantum every time
     * P4    : medium batch
     */
    struct task in[] = {
        { 0,  6,  6, 0, 0, -1, 0 },
        { 1, 14, 14, 0, 0, -1, 0 },
        { 2, 18, 18, 0, 0, -1, 0 },
        { 3, 12, 12, 0, 0, -1, 0 },
        { 4, 20, 20, 0, 0, -1, 0 },
    };
    int n = sizeof in / sizeof *in;

    int clock = 0, admitted = 0, done = 0, since_boost = 0;
    double wait_sum = 0, resp_sum = 0;
    int boosts = 0;

    memset(len, 0, sizeof len);
    memset(hd, 0, sizeof hd);
    memset(tl, 0, sizeof tl);

    printf("MLFQ: %d queues, quanta %d/%d/%d, boost every %d\n",
           NQ, quantum[0], quantum[1], quantum[2], BOOST);
    printf("Gantt: ");

    while (done < n) {
        /* R4 — periodic boost, before anything else, so a long task
           is never postponed forever */
        if (since_boost >= BOOST) {
            /* move everything to the top queue, preserving arrival order */
            struct task all[MAX_TASKS];
            int k = 0;
            for (int l = 0; l < NQ; l++) {
                while (len[l] > 0) all[k++] = dequeue(l);
            }
            for (int i = 0; i < k; i++) {
                all[i].level = 0;             /* boost to top */
                enqueue(0, all[i]);
            }
            since_boost = 0;
            boosts++;
            printf("  [boost #%d → all %d tasks to Q0]\n  ", boosts, k);
        }

        /* R1 — admit new arrivals at the TOP queue */
        while (admitted < n && in[admitted].arrival <= clock) {
            in[admitted].level = 0;
            in[admitted].first_run = -1;
            enqueue(0, in[admitted]);
            admitted++;
        }

        int l = highest();
        if (l < 0) { clock++; continue; }

        struct task t = dequeue(l);
        if (t.first_run < 0) t.first_run = clock;
        t.runs++;
        printf("P%d(q%d) ", t.id, l);

        /* R2 — run for the full quantum of this queue, or until it finishes */
        int run = t.left < quantum[l] ? t.left : quantum[l];
        t.left -= run;
        clock += run;
        since_boost++;

        if (t.left == 0) {
            done++;
            wait_sum += clock - t.arrival - (in[t.id].burst);
            resp_sum += t.first_run - t.arrival;
        } else {
            /* R3 — quantum exhausted → demote one level */
            t.level = l + 1 < NQ ? l + 1 : NQ - 1;
            enqueue(t.level, t);
        }
    }

    printf("\n");
    printf("total time   : %d\n", clock);
    printf("boosts       : %d\n", boosts);
    printf("avg waiting  : %.2f\n", wait_sum / n);
    printf("avg response : %.2f\n", resp_sum / n);
    return 0;
}
```

```bash
gcc -Wall -Wextra -O2 -std=c11 -o mlfq mlfq.c && ./mlfq
```

```text
MLFQ: 3 queues, quanta 2/4/8, boost every 20
Gantt: P0(q0) P1(q0) P2(q0) P0(q0) ...
[boost #1 → all tasks to Q0]
...
total time   : 70
avg waiting  : ...
avg response : ...
```

## Output interpretation

```text
  Reading a Gantt trace from MLFQ:

    P0(q0) P0(q0) P0(q0)   ← P0 keeps its Q0 slot, so it keeps blocking
                              early. Interactive. Stays at the top. ✔

    P2(q0) P2(q1) P2(q2)   ← P2 walks down the queues, so it used the
                              full quantum every time. CPU-bound. ✔

    [boost #1 → all 5 tasks to Q0]  ← R4 fired. Without this, P2 (which
                              has been at Q2) could be postponed forever
                              by a stream of P0s. ✔
```

---

# The Two Failure Modes

## Convoy effect — too few queues, too-coarse quanta

```text
  SETUP: 2 queues. Q0 quantum 2, Q1 FCFS.

    A long process uses all 2 units of Q0, gets demoted to Q1. Good.
    But now the NEXT long process also has to walk down, and the
    quanta are so small that the switch overhead dominates.

  WORSE VARIANT: 1 quantum for all queues, small.

    Everything looks like a long job after the first demotion.
    An interactive task that needs 3 ms of CPU with quantum 1 is
    demoted on every dispatch, and now it behaves exactly like a
    batch job. The system has learned the WRONG lesson.

  SYMPTOM: interactive processes get a slice that is too small, so
  they thrash between queues, and throughput drops. The scheduler
  has over-fit to short bursts.

  FIX: more queues, larger quanta at the top, so a genuine interactive
  task can accumulate several slices before it is demoted.
```

## Thrashing — too many boosts

```text
  SETUP: boost period m is too small, e.g. m = 2.

  Every 2 dispatches, EVERY task is thrown back to Q0.

  A long process that had patiently descended to Q2 is yanked back
  to Q0, runs for one small quantum, is demoted to Q1, then boosted
  to Q0 again. It never makes progress, because it spends more time
  being promoted than executing.

  SYMPTOM: the bottom queues are never used. The scheduler is
  effectively round-robin at the top with extra steps. Throughput
  collapses and context switch count explodes.

  FIX: increase m. It must be long enough that a long process can
  make real progress between boosts. A common choice is
  m ≈ 10× the number of quanta needed to reach the bottom.
```

```text
  THE TENSION, IN ONE LINE

  small m  →  starvation possible, but no thrashing
  large m  →  no thrashing, but a long process can be punished for
              a long time

  m must be long enough for progress, short enough for fairness.
  There is no formula; it is tuned against a real workload.
```

---

# Worked Example

Tasks: A(interactive, blocks often), B(CPU-bound), C(medium), boost every 20.

```text
  CONFIG: Q0 q=2, Q1 q=4, Q2 q=8. Boost every 20 dispatches.

  t=0   A→Q0, B→Q0, C→Q0                       (R1: all new → top)

  A uses 2, blocks              → stays Q0  (did NOT exhaust)
  B uses 2 of 4 needed? B is long, so it uses 2 and stops? No —
        B is CPU-bound, so it uses the FULL 2 → DEMOTE to Q1  (R3)
  C uses 2 of 8 needed? C is medium; suppose it uses 2 and blocks
        on I/O for a while       → stays Q0

  t=2   A resumes, uses 2, blocks               → Q0
  C resumes, uses 2, blocks                    → Q0
  B runs in Q1, uses 4, still has work          → DEMOTE to Q2  (R3)
  ...

  t=20  boost fires. A, B, C all → Q0.          (R4)

  POST-BOOST, the pattern repeats from the top.
  B walks down again. A never leaves Q0.

  OBSERVATION: the boost is a full reset, so B re-descends the
  hierarchy every 20 dispatches. That is the price of fairness.
  A scheduler that only boosted *starving* tasks, rather than all
  of them, would avoid B's repeated descent — see MLFQ+ below.
```

---

# MLFQ+

Vanilla MLFQ boosts **everything**, which resets even well-behaved long tasks.
**MLFQ+** (also called "MLFQ with a boost only for starving tasks") is more
refined:

| Vanilla MLFQ | MLFQ+ |
|---|---|
| Boost ALL tasks every `m` quanta. | Boost only tasks that have waited too long. |
| Every long task re-descends after every boost. | Long tasks keep their level unless they are actually starving. |
| Thrashes if `m` is small. | Thrashes much less. |
| Boost work is `O(total tasks)` every `m`. | Boost work is `O(starving tasks)`. |

```text
  MLFQ+ rule: a task in queue i is boosted if it has been waiting
  in that queue for more than S quanta. S is the "starvation bound".

  This bounds worst-case wait at S quanta, without resetting
  well-behaved tasks, so m (the full-boost period) can be large
  or removed entirely.
```

---

# MLFQ in Real Systems

```text
  No production Linux scheduler is a literal MLFQ. The idea, however,
  is the exact mechanism behind `nice`, behind cgroup weights, and
  behind every "the interactive thing should run first" mechanism.
```

| Linux mechanism | Which MLFQ idea it is |
|---|---|
| **`nice` value** | a static, pre-declared level. It is the "MLQ with no feedback" version. The system does not learn; you declare. |
| **CFS `vruntime`** | feedback. Weight a task by how much CPU it has actually used, and give a light task proportionally more. Behaviour determines share. |
| **Autonomous groups / `sched_group_set_idle`** | R4, coarsely: idle cgroups are treated as boosting, so a bursty process can burst. |
| **`SCHED_BATCH`** | the demotion done explicitly: the user says "this is a long job, do not preempt me." |
| **`sched_wakeup_granularity_ns`** | the top-level quantum: how long a newly woken task runs before it is considered for preemption. This is the R1/R2 slice for the interactive level. |
| **cgroup `cpu.weight`** | a per-group level, equivalent to "which queue does this process belong to" for a whole service. |

```bash
# nice: the pre-declared version. Lower = higher priority.
nice -n 10 ./background_batch          # start at a low level
renice 5 -p $(pidof myserver)           # adjust a running process

# cgroup weights: a whole service at one level, with share
systemd-run --unit=myapp -p CPUWeight=200 ./myapp

# see the weights
cat /sys/fs/cgroup/cpu.weight
cat /proc/<pid>/sched            # shows vruntime, se.sum_exec_runtime

# The top-level quantum, the R1/R2 slice
cat /proc/sys/kernel/sched_wakeup_granularity_ns
cat /proc/sys/kernel/sched_min_granularity_ns
cat /proc/sys/kernel/sched_latency_ns
```

```c
/*
 * The Linux mechanism in one program: a task's share is driven by
 * observed CPU usage, not by a declared class. This is the "MLFQ+" idea
 * applied directly, via the nice value.
 */
#define _GNU_SOURCE
#include <errno.h>
#include <pthread.h>
#include <sched.h>
#include <stdio.h>
#include <string.h>
#include <sys/resource.h>
#include <unistd.h>

static volatile long work_done;

static void *cpu_hog(void *arg)
{
    (void)arg;
    for (long i = 0; i < 200000000L; i++) work_done += (i & 1);
    return NULL;
}

int main(void)
{
    long cores = sysconf(_SC_NPROCESSORS_ONLN);
    printf("%ld cores\n", cores);

    /*
     * Start a thread, then renice the WHOLE process. A nice process
     * does not get a lower CLASS; its vruntime advances faster, so
     * CFS returns to it less often. Same feedback idea as MLFQ
     * demotion, but continuous rather than in discrete steps.
     */
    pthread_t hog;
    pthread_create(&hog, NULL, cpu_hog, NULL);

    if (setpriority(PRIO_PROCESS, 0, 15) == 0) {
        errno = 0;
        printf("process niced to %d (weight 36 vs 1024 for nice 0)\n",
               getpriority(PRIO_PROCESS, 0));
        printf("→ it keeps ~3.5%% of the CPU it had before\n");
    } else {
        printf("setpriority failed: %s\n", strerror(errno));
    }

    pthread_join(hog, NULL);
    printf("work_done = %ld\n", work_done);
    return 0;
}
```

```bash
gcc -O2 -pthread -o nice_feedback nice_feedback.c && ./nice_feedback
```

---

# Choosing the Parameters

| Parameter | Too small | Too large | Rule of thumb |
|---|---|---|---|
| **n** (queues) | convoys; short tasks are demoted wrongly | overhead, and no way to distinguish types | 3–8; more only if you can see distinct types in the workload |
| **m** (boost period) | thrashing; bottom queues unused | starvation for up to m quanta | m ≥ 10 × (time to descend the hierarchy) |
| **top quantum** | interactive tasks are punished | slow response for the user | 5–20 ms; above the ~100 ms perception threshold, below the switch cost × 20 |
| **quantum growth** | convoy effect | long tasks hog the CPU | doubling |

```text
  THE DESIGN QUESTION

  How many distinct types does the workload actually have?

    A web server:      interactive (short) + background batch        → 3 queues
    A build system:    short (headers) + long (compile) + I/O (fetch)  → 4 queues
    A scientific job:  mostly one giant CPU phase, a few I/O phases    → 2 queues

  Adding a queue is not free: it is another O(1)-scan entry, another
  boost bucket, and a finer-grained approximation of the workload.
  If the workload has no fifth type, the fifth queue is pure overhead.
```

---

# Common Misconceptions

### ❌ "MLFQ is Round-Robin with extra queues."

Not quite. The defining feature is that the **quantum depends on the queue
level**, and that a task is **demoted** when it exhausts its quantum. Without
demotion, MLFQ degenerates to Round-Robin. The feedback is the whole point.

---

### ❌ "R2 (run the whole quantum) is a fairness rule."

It is an **anti-convoy** rule, and this is subtle.

R2 says: only demote if the task used the *whole* quantum. This means a
short, I/O-bound task is never demoted just for being small — it only ever
loses its slot if it was genuinely greedy. The comparison is against the
*slice*, not against the average task, so the system does not need to know
what "normal" is. This is why R2 is essential and why you should not
"improve" MLFQ by shortening the top-level quantum.

---

### ❌ "Boosting every `m` quanta makes the scheduler perfectly fair."

No. It makes starvation *bounded*, not impossible. During the interval
between boosts, a long task is still systematically postponed. And boosting
**everything** means well-behaved long tasks lose the level they worked to
reach. MLFQ+ fixes the second problem; the first is fixed only by tuning `m`.

---

### ❌ "Linux's scheduler is MLFQ."

It is not. Linux CFS is a proportional-share scheduler driven by `vruntime`,
not a queue hierarchy with demotion. The *philosophy* — let behaviour determine
share, rather than a declared class — is shared, and the `nice` value is the
static-declaration version of the same idea. But there is no demotion step and
no periodic boost.

---

# Interview Questions

### Basic

- What is the MLFQ scheduling algorithm?
- What is the key idea behind MLFQ?
- State the four rules of MLFQ.
- What are the parameters `n` and `m`, and what does each control?

### Intermediate

- Why does MLFQ not suffer from starvation? Which rule guarantees it?
- What is the convoy effect in MLFQ, and what causes it?
- Explain the difference between MLFQ and MLFQ+.
- Why does the top-level quantum need to be short while the bottom-level needs to be long?

### Advanced

- You have 8 queues. Describe how you would implement priority boost in O(1) per task.
- A workload has exactly two kinds of tasks. Is MLFQ still better than RR? Justify.
- Your MLFQ scheduler is thrashing. Diagnose it and give two changes.
- Linux uses CFS instead of MLFQ. Give the argument for and against each.

---

# University Exam Notes

### Definitions

- **Multilevel feedback queue (MLFQ):** A scheduling algorithm that uses
  multiple queues with increasing time quanta, and dynamically moves processes
  between queues based on observed CPU usage.
- **Boosting:** The periodic promotion of all processes to the top queue to
  prevent starvation.
- **Convoy effect (MLFQ):** The degradation that occurs when there are too few
  queues, so interactive tasks are demoted and behave like batch tasks.
- **Thrashing (MLFQ):** Excessive promotion and demotion caused by too short a
  boost period, leading to poor throughput.
- **MLFQ+:** A variant that boosts only the tasks that have waited too long,
  rather than all tasks.

### Frequently Asked Questions

- What is MLFQ scheduling? Explain its working.
- What is the key idea of MLFQ scheduling?
- Explain the four rules of MLFQ.
- What is starvation in MLFQ? How is it prevented?
- What are the two failure modes of MLFQ, and how are they fixed?
- Differentiate MLFQ from MLQ and from MLFQ+.
- How do you choose the number of queues `n` and the boost period `m`?

---

# Key Takeaways

- **The key idea:** the scheduler cannot know a task's type in advance, so it
  **watches behaviour** and infers the type. Short tasks block early and stay
  high; long tasks exhaust their quantum and get demoted.
- The **four rules**: new→top, run the full quantum, demote on exhaustion,
  boost periodically. Rule 4 is the anti-starvation guarantee.
- **n** (queues) controls classification precision; **m** (boost period)
  controls how long a long task can be punished. Too small `m` causes
  thrashing; too large causes unfairness.
- **Convoy effect**: too few queues or too small a quantum, so interactive
  tasks are demoted and mis-learned.
- **Thrashing**: too frequent a boost, so tasks are promoted faster than they
  can descend and the bottom queues go unused.
- `n` too small → convoy. `m` too small → thrash. Both are tuning failures
  with opposite fixes.
- Use a **bitmask of non-empty queues** for O(1) "highest priority queue"
  lookup instead of an O(n) scan.
- **MLFQ+** boosts only genuinely starving tasks, so well-behaved long tasks
  keep their level.
- Linux does not use MLFQ literally, but the philosophy is everywhere: `nice`
  is the declared version, `vruntime` is the behavioural version,
  `SCHED_BATCH` is manual demotion, cgroup weights are per-group levels.

---

# References

- Operating System Concepts — Silberschatz, §8.3.2 Multilevel Feedback Queue
- Modern Operating Systems — Tanenbaum, §2.5.3
- Silberschatz, Galvin, Gagne — MLFQ worked examples and exercises
- The Linux Kernel Documentation — `Documentation/scheduler/`
- Linux kernel source: `kernel/sched/fair.c`
- cgroup v2 documentation — `cpu.weight`
- `man 2 setpriority`, `ps -eLo cls,ni,pri`, `cat /proc/sys/kernel/sched_latency_ns`
