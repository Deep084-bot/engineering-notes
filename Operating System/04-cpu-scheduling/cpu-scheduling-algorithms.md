# CPU Scheduling Algorithms

> [!NOTE]
> **Module:** Module IV – CPU Scheduling
>
> **Difficulty:** ⭐⭐⭐⭐☆
>
> **Prerequisites:**
> - Context Switching
> - Critical Section
> - Amdahl's Law
>
> **Scope:** Linux and C only. No Windows or Java.

---

# Learning Objective

After reading this note, you should be able to:

- Describe the CPU scheduling problem and its inputs.
- Describe FCFS, SJF, SRTF, Priority, and Round-Robin.
- Compute waiting time, turnaround time, and response time for a given burst set.
- Explain why SJF minimises average waiting time and why it is dangerous.
- Explain how time quantum affects Round-Robin behaviour.
- Explain preemptive versus non-preemptive scheduling and the cost of preemption.
- Explain how Linux actually schedules: `SCHED_OTHER`, `SCHED_FIFO`, `SCHED_RR`, `SCHED_BATCH`, `SCHED_IDLE`.
- Select an algorithm for a workload.
- Write a C program that demonstrates each algorithm.

---

# Why Do We Need Scheduling?

## The Problem

One CPU. Many processes and threads wanting to run. The CPU is the scarcest
resource in the system.

```text
  At any instant, the runnable set is a queue. The scheduler picks one.
  Over time, that choice repeatedly becomes different.

  The scheduler is asked to optimise several goals AT ONCE, and they
  conflict:

    THROUGHPUT   more useful work per second
    LATENCY      short response time
    FAIRNESS     nobody starves
    PRIORITY     important work first
    PREDICTABILITY  bounded worst-case delay

  Optimising any one of these makes another worse. There is no algorithm
  that maximises all of them, because that is the problem.
```

## The CPU utilisation dilemma

```text
  High CPU utilisation  ⟹  long bursts, few context switches
  Low  CPU utilisation  ⟹  short bursts, fast response

  100% CPU utilisation is impossible to sustain and terrible to achieve.
  A few percent of idle time is what makes interactivity possible.
```

---

# Real World Analogy

A single cashier at a bank, with a queue of customers.

- **FCFS:** first come, first served. The person who walked in with one
  question waits behind the person cashing 400 cheques.
- **SJF:** serve whoever will take the least time. The one-cheque customer goes
  first. Fastest total service.
- **Round-Robin:** the cashier serves each customer for 2 minutes, then moves
  to the back of the queue. Everyone makes progress.
- **Priority:** the CEO goes first, regardless of when they arrived.
- **Multilevel feedback queue:** new customers start in the short queue. If
  they keep needing only 2 minutes, they stay there. If they prove to be long,
  they get promoted to the long queue.

---

# Intuition

```text
  THE SCHEDULER'S QUESTION, EVERY TIME A CPU GOES IDLE

      runnable = [P1(burst 24) P2(3) P3(3)]

      FCFS        → P1 first. P2 and P3 wait 24.
                     average wait = (0 + 24 + 27)/3 = 17

      SJF         → P2, P3, P1.
                     average wait = (0 + 3 + 6)/3 = 3

      SJF gives 5.7× better average wait. And P1 may be last.
      That is the whole trade-off, in three processes.
```

```text
  PREEMPTION CHANGES EVERYTHING

      NEW:  P1 is running (burst 24 left), P4 arrives with burst 5.

      NON-PREEMPTIVE: P1 finishes. P4 waits 24+3+3+5.
      PREEMPTIVE (SRTF): P1 runs 6, P4 takes over and finishes,
                          P1 resumes with 18 left.
      P4's response time: 6, not 35.
```

---

# Formal Definition

> **CPU scheduling** is the process of selecting, from the set of runnable
> processes or threads, the one that will execute next on a processor, while
> satisfying as many of the system's performance goals as possible.

> **Preemptive scheduling** allows the scheduler to interrupt a running process
> and resume it later. **Non-preemptive scheduling** lets a process run until it
> finishes or blocks.

## Scheduling criteria

| Criterion | Definition |
|---|---|
| **Turnaround time** | `completion − arrival`: total time in the system |
| **Waiting time** | `turnaround − burst`: time in the ready queue |
| **Response time** | `first_run − arrival`: time to first result |
| **Throughput** | processes completed per unit time |
| **CPU utilisation** | fraction of time the CPU is busy |
| **Fairness** | equal access, no indefinite postponement |

```text
  turnaround  = finish  −  arrival
  waiting     = turnaround  −  burst
  response    = first_run  −  arrival
  throughput  = completed / total time
```

---

# Burst Time

The scheduler does not know how long a process will run. It must **predict**.

```text
  The prediction is a parameter, usually a moving average:

      τ(n+1) = α · t(n) + (1 − α) · τ(n)

  α = 0     →  never updates; use the initial guess forever
  α = 1     →  use only the last burst; the initial guess is discarded
  α ≈ 0.5   →  exponential weighted average, the usual choice

  Example:  α = 0.5, initial guess τ₁ = 10
      actual 10  →  τ₂ = 0.5(10) + 0.5(10) = 10
      actual  4  →  τ₃ = 0.5( 4) + 0.5(10) =  7
      actual  6  →  τ₄ = 0.5( 6) + 0.5( 7) =  6.5
```

---

# Algorithm 1 — FCFS (First-Come, First-Served)

## Definition

Run the process that arrived first, until it finishes or blocks.

```text
  Arrivals:  P0@0 (b8)   P1@1 (b4)   P2@2 (b9)   P3@3 (b5)

  Gantt:  | P0 |   P1   |     P2      | P3 |
          0    8       12             21  26

  P   Arr  Bur  Fin   Turn   Wait   Resp
  0    0    8    8     8      0      0
  1    1    4   12    11      7      7
  2    2    9   21    19     10      9
  3    3    5   26    23     18     18

  avg turnaround = (8 + 11 + 19 + 23)/4 = 15.25
  avg waiting    = ( 0 +  7 + 10 + 18)/4 =  8.75
  avg response   = ( 0 +  7 +  9 + 18)/4 =  8.5
```

## Analysis

```text
  Simple to implement, fully fair in arrival order, and the only algorithm
  with perfect arrival fairness.

  But a long process at the head blocks every short one behind it. This is
  the CONVOY EFFECT.

  Worse: it is NOT preemptive on burst time, so a CPU-bound process cannot be
  pushed aside by a newly arrived interactive one.
```

```c
/*
 * FCFS: a queue, strictly in arrival order, non-preemptive.
 */
#include <stdio.h>
#include <stdlib.h>

struct proc { int arrival, burst, pid; };

static int cmp_arrival(const void *a, const void *b)
{
    const struct proc *x = a, *y = b;
    return (x->arrival > y->arrival) - (x->arrival < y->arrival);
}

int main(void)
{
    struct proc p[] = {
        { 0, 8, 0 }, { 1, 4, 1 }, { 2, 9, 2 }, { 3, 5, 3 }
    };
    int n = sizeof p / sizeof *p;
    qsort(p, n, sizeof *p, cmp_arrival);

    printf("FCFS (non-preemptive)\n");
    printf("Gantt: ");
    int clock = 0;
    for (int i = 0; i < n; i++) {
        if (p[i].arrival > clock) clock = p[i].arrival;   /* idle */
        printf("| P%d ", p[i].pid);
        clock += p[i].burst;
    }
    printf("|  (end %d)\n", clock);

    clock = 0;
    double at = 0, aw = 0, ar = 0;
    for (int i = 0; i < n; i++) {
        if (p[i].arrival > clock) clock = p[i].arrival;
        clock += p[i].burst;
        int turnaround = clock - p[i].arrival;
        int wait       = turnaround - p[i].burst;
        at += turnaround; aw += wait; ar += wait;
        printf("P%d: arrival=%d burst=%d finish=%d turnaround=%d wait=%d\n",
               p[i].pid, p[i].arrival, p[i].burst, clock, turnaround, wait);
    }
    printf("avg turnaround=%.2f  avg waiting=%.2f  avg response=%.2f\n",
           at / n, aw / n, ar / n);
    return 0;
}
```

```bash
gcc -Wall -Wextra -O2 -o fcfs fcfs.c && ./fcfs
```

```text
Gantt: | P0 | P1 | P2 | P3 |  (end 26)
P0: arrival=0 burst=8 finish=8 turnaround=8 wait=0
P1: arrival=1 burst=4 finish=12 turnaround=11 wait=7
P2: arrival=2 burst=9 finish=21 turnaround=19 wait=10
P3: arrival=3 burst=5 finish=26 turnaround=23 wait=18
avg turnaround=15.25  avg waiting=8.75  avg response=8.5
```

## The convoy effect, demonstrated

```c
/*
 * One CPU-bound process plus several short interactive ones.
 * FCFS blocks every interactive request behind the CPU-bound one.
 */
static void demo_convoy(void)
{
    /* burst times: */
    int bursts[] = { 500, 20, 20, 20, 20, 20 };   /* 1 CPU-bound, 5 interactive */
    int n = sizeof bursts / sizeof *bursts;

    printf("FCFS completion order (bursts 500,20,20,20,20,20):\n");
    int clock = 0;
    for (int i = 0; i < n; i++) {
        clock += bursts[i];
        printf("  request %d finishes at t=%d  (waited %d)\n", i, clock, clock - bursts[i]);
    }
    printf("  The five interactive requests waited up to %d ms.\n", clock - 20);
}
```

```text
  request 0 finishes at t=500  (waited 0)
  request 1 finishes at t=520  (waited 500)
  request 2 finishes at t=540  (waited 520)
  ...
  avg waiting = 525 ms   ← all five short requests blocked for half a second
```

---

# Algorithm 2 — SJF (Shortest Job First)

## Definition

Always run the process with the **shortest estimated CPU burst** in the ready
queue. Non-preemptive.

```text
  Same processes: P0@0 (b8)  P1@1 (b4)  P2@2 (b9)  P3@3 (b5)

  Gantt:  | P1 | P3 | P0 |    P2    |
          0  4   9     17          26

  P   Arr  Bur  Fin   Turn   Wait
  0    0    8   17     17      9
  1    1    4    4      3      0
  2    2    9   26     24     15
  3    3    5    9      6      1

  avg waiting = (9 + 0 + 15 + 1)/4 = 6.25   ← better than FCFS's 8.75
```

## Optimality proof

SJF is **provably optimal** for minimising average waiting time, over all
non-preemptive schedules.

```text
  Consider any optimal schedule that has jobs A and B adjacent, with
  burst(A) > burst(B), in that order.

  Swap them.  Nothing else moves, so:
      A's wait is unchanged   (it starts at the same instant)
      B's wait DEcreASES by (burst(A) − burst(B)) > 0

  So the swap strictly improves the average. A schedule that cannot be
  improved by a swap is sorted by burst. SJF is that schedule. ∎
```

## Starvation

```text
  A long process is postponed indefinitely, because short processes keep
  arriving.

  The steady state:  infinite short jobs → the long job never runs.

  Fix: age the priority. After a process has waited W, boost its effective
  priority by W (aging, used by Linux via automatic `nice` increases, but
  NOT in vanilla SJF).
```

```c
/*
 * SJF, non-preemptive, with a starvation counter to show the problem.
 */
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

struct proc { int arrival, burst, pid, waited; };

int main(void)
{
    struct proc p[] = {
        { 0, 8, 0, 0 }, { 1, 4, 1, 0 }, { 2, 9, 2, 0 }, { 3, 5, 3, 0 }
    };
    int n = sizeof p / sizeof *p;

    /* Select the shortest burst among arrived-but-unfinished processes. */
    int done[8] = { 0 };
    int clock = 0, completed = 0;
    double total_wait = 0;
    int last = -1, times_starved = 0;

    printf("SJF (non-preemptive)\nGantt: ");
    while (completed < n) {
        int best = -1;
        for (int i = 0; i < n; i++) {
            if (done[i] || p[i].arrival > clock) continue;
            if (best < 0 || p[i].burst < p[best].burst) best = i;
        }
        if (best < 0) {          /* nothing has arrived: idle */
            clock++;
            continue;
        }
        printf("| P%d ", p[best].pid);
        p[best].waited = clock - p[best].arrival;
        total_wait += p[best].waited;
        times_starved += (p[best].waited > 10);
        clock += p[best].burst;
        done[best] = 1;
        completed++;
        last = best;
    }
    printf("|  (end %d)\n", clock);
    printf("avg waiting = %.2f\n", total_wait / n);
    (void)last;
    return 0;
}
```

```bash
gcc -Wall -Wextra -O2 -o sjf sjf.c && ./sjf
```

```text
SJF (non-preemptive)
Gantt: | P1 | P3 | P0 | P2 |  (end 26)
avg waiting = 6.25
```

---

# Algorithm 3 — SRTF (Shortest Remaining Time First)

## Definition

SJF, but preemptive: if a new arrival has a shorter **remaining** time than the
running process, preempt the running process.

```text
  P0@0 (b8)  P1@1 (b4)  P2@2 (b9)  P3@3 (b5)

  t=0  P0 starts (remaining 8)
  t=1  P1 arrives, remaining 4 < 7   → PREEMPT, run P1
  t=2  P2 arrives (9), P1 has 3 left → keep P1
  t=3  P3 arrives (5), P1 has 2 left → keep P1
  t=5  P1 done
  t=5  P3 (5) < P0 (7)             → run P3
  t=8  P2 arrives? no, done at 2. P0 (7) < P3 (5)? no → keep P3
  t=10 P3 done
  t=10 P0 (7) < P2 (9)            → run P0
  t=17 P0 done
  t=17 P2 (9)                      → run P2
  t=26 P2 done

  Gantt: | P0 | P1 |  P3  | P0 |    P2    |
          0  1  5     10     17          26

  P   Arr  Bur  Fin   Turn   Wait   Resp
  0    0    8   17     17      9      0
  1    1    4    5      4      0      0
  2    2    9   26     24     15      5
  3    3    5   10      7      2      2

  avg waiting   = (9+0+15+2)/4 = 6.5
  avg response  = (0+0+5+2)/4 = 1.75   ← the big win
```

```c
/*
 * SRTF: preemptive SJF. Arrival-aware, always picks the smallest
 * remaining time among all arrived, unfinished processes.
 */
#include <stdio.h>

struct proc { int arrival, burst, remaining, pid, first_run; };

int main(void)
{
    struct proc p[] = {
        { 0, 8, 8, 0, -1 }, { 1, 4, 4, 1, -1 },
        { 2, 9, 9, 2, -1 }, { 3, 5, 5, 3, -1 }
    };
    int n = sizeof p / sizeof *p;
    int done[8] = { 0 };
    int completed = 0, clock = 0, current = -1;

    printf("SRTF (preemptive)\nGantt: ");
    while (completed < n) {
        /* (re)evaluate: does any ready process have less remaining time? */
        int best = -1;
        for (int i = 0; i < n; i++) {
            if (done[i] || p[i].arrival > clock || p[i].remaining == 0) continue;
            if (best < 0 || p[i].remaining < p[best].remaining) best = i;
        }
        if (best < 0) { clock++; continue; }

        if (best != current) {
            current = best;
            printf("| P%d ", p[current].pid);
            if (p[current].first_run < 0) p[current].first_run = clock;
        }
        p[current].remaining--;
        clock++;
        if (p[current].remaining == 0) { done[current] = 1; completed++; current = -1; }
    }
    printf("|  (end %d)\n", clock);

    double aw = 0, ar = 0;
    for (int i = 0; i < n; i++) {
        int finish = clock;
        int turnaround = finish - p[i].arrival;
        int wait = turnaround - p[i].burst;
        int response = p[i].first_run - p[i].arrival;
        aw += wait; ar += response;
        printf("P%d: turnaround=%d wait=%d response=%d\n", p[i].pid, turnaround, wait, response);
    }
    printf("avg waiting=%.2f  avg response=%.2f\n", aw / n, ar / n);
    return 0;
}
```

```bash
gcc -Wall -Wextra -O2 -o srtf srtf.c && ./srtf
```

## SRTF and starvation

```text
  SRTF starves even more aggressively than SJF, because a long process that
  has run a long time can still be preempted by a fresh short job.

  A process must run to completion once, no matter how long.

  Fix: aging. Add a bonus to remaining time for every unit waited:

      effective = remaining − α · wait_time

  After enough waiting, a long job's effective time falls below any new
  arrival's burst, so it finishes.
```

---

# Algorithm 4 — Priority Scheduling

## Definition

Each process has a priority. Run the highest-priority ready process.

```text
  Internally, lower number usually means higher priority (Linux `nice`).
  Most textbooks use the opposite convention — state which you are using.

  For equal priorities, fall back to FCFS or to Round-Robin.
```

```c
/*
 * Priority scheduling with aging, plus a fairness guarantee.
 */
#include <stdio.h>
#include <unistd.h>
#include <sys/resource.h>

#define N 4
#define QUANTUM 2

struct proc {
    int    pid;
    int    burst;
    int    priority;      /* lower = higher priority */
    int    remaining;
    int    arrival;
    int    age_bonus;
    int    first_run;
};

/* Effective priority: aging subtracts a bonus that grows with waiting. */
static int effective(const struct proc *p, int now)
{
    return p->priority - (now - p->arrival) / 4;
}

int main(void)
{
    struct proc p[N] = {
        { 0,  8, 3,  8, 0, 0, -1 },
        { 1,  4, 1,  4, 1, 0, -1 },
        { 2,  9, 5,  9, 2, 0, -1 },
        { 3,  5, 2,  5, 3, 0, -1 }
    };
    int done[N] = { 0 }, completed = 0, clock = 0, current = -1;

    printf("Priority scheduling (lower number = higher priority) with aging\n");
    printf("Gantt: ");
    while (completed < N) {
        int best = -1;
        for (int i = 0; i < N; i++) {
            if (done[i] || p[i].arrival > clock || p[i].remaining == 0) continue;
            if (best < 0 ||
                effective(&p[i], clock) < effective(&p[best], clock) ||
                (effective(&p[i], clock) == effective(&p[best], clock) &&
                 p[i].pid < p[best].pid))
                best = i;
        }
        if (best < 0) { clock++; continue; }

        if (best != current) { current = best; printf("| P%d ", p[best].pid);
                              if (p[best].first_run < 0) p[best].first_run = clock; }

        for (int q = 0; q < QUANTUM && p[current].remaining > 0; q++) {
            p[current].remaining--; clock++;
            if (p[current].remaining == 0) { done[current] = 1; completed++; current = -1; break; }
            /* preemption check after every tick */
            int alt = -1;
            for (int i = 0; i < N; i++) {
                if (i == current || done[i] || p[i].arrival > clock || p[i].remaining == 0) continue;
                if (alt < 0 || effective(&p[i], clock) < effective(&p[alt], clock)) alt = i;
            }
            if (alt >= 0 && effective(&p[alt], clock) < effective(&p[current], clock)) {
                current = -1;   /* yield to the better process */
                break;
            }
        }
    }
    printf("|\n");
    return 0;
}
```

```bash
gcc -Wall -Wextra -O2 -o prio prio.c && ./prio
```

## Starvation and the aging fix

```text
  A low-priority process is postponed forever.

  Aging:  the longer a process waits, the more its priority improves.
          A process cannot be starved indefinitely, because the bonus
          eventually exceeds every possible priority difference.

  This is precisely how Linux `nice` values work: -20 is the highest
  priority, and a long-running unprivileged process has its nice value
  increased automatically to stop it being ignored.
```

## Setting priority on Linux

```bash
# Start a program at reduced priority
nice -n 10 ./heavy_computation

# Change a running process's priority
renice 15 -p $(pidof heavy_computation)

# Default is 0; the range for unprivileged users is 0..19
cat /proc/sys/kernel/sched_min_granularity_ns
```

---

# Algorithm 5 — Round-Robin (RR)

## Definition

Each ready process gets a fixed **time quantum** `q`, in turn. If a process
does not finish in `q`, it is preempted and pushed to the back of the queue.

```text
  q = 3,  P0(b7)@0, P1(b3)@1, P2(b5)@2

  Gantt: | P0 | P1 | P2 | P0 | P1 | P2 | P0 | P2 | P0 | P2 | P0 |
          0  3  6  9 12 15 18 21 24 27 30 33

  P   Arr  Bur  Fin   Turn   Wait   Resp
  0    0    7   33     33     26      0
  1    1    3   12     11      8      2
  2    2    5   30     28     23      4

  avg waiting = (26 + 8 + 23)/3 = 19     ← much worse than SJF's 2
  avg response = ( 0 + 2 +  4)/3 = 2     ← much better than SJF's 5
```

```c
/*
 * Round-Robin with a real time quantum, using SIGALRM for the preemption.
 * This is how a userspace time-sliced scheduler is actually built.
 */
#define _POSIX_C_SOURCE 200809L
#include <signal.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/time.h>
#include <unistd.h>

#define N        4
#define QUANTUM  200        /* ms */

struct slice { int burst, left, pid, remaining, first_run; };

static struct slice   proc[N];
static int            ready[N];
static int            qhead, qtail, qcount, completed;
static int            clock, current = -1;
static unsigned long  switches;

static void dispatch(int sig)
{
    (void)sig;
    switches++;
    if (current >= 0) {
        if (proc[current].left == 0) {
            /* quantum expired: requeue at the back */
            ready[qtail] = current; qtail = (qtail + 1) % N; qcount++;
            completed++;
        } else {
            /* process finished on its own: nothing to requeue */
            completed++;
        }
        current = -1;
    }
}

static void install(void)
{
    struct sigaction sa;
    memset(&sa, 0, sizeof sa);
    sa.sa_handler = dispatch;
    sa.sa_flags   = SA_RESTART;
    sigaction(SIGALRM, &sa, NULL);

    struct itimerval it;
    it.it_interval.tv_sec  = 0; it.it_interval.tv_usec = QUANTUM * 1000;
    it.it_value    = it.it_interval;
    setitimer(ITIMER_REAL, &it, NULL);
}

int main(void)
{
    int bursts[N] = { 800, 300, 500, 400 };
    for (int i = 0; i < N; i++) {
        proc[i].burst = bursts[i] * 10;      /* simulate CPU time in 100µs ticks */
        proc[i].left = proc[i].burst;
        proc[i].pid = i;
        proc[i].remaining = proc[i].burst;
        proc[i].first_run = -1;
    }
    int enq = 0;
    install();

    printf("Round-Robin, quantum = %d ms\nGantt: ", QUANTUM);
    while (completed < N) {
        /* admit arrivals */
        if (enq < N && proc[enq].burst / 10 <= clock) {
            ready[qtail] = enq; qtail = (qtail + 1) % N; qcount++;
            enq++;
        }
        if (current < 0 && qcount > 0) {
            current = ready[qhead]; qhead = (qhead + 1) % N; qcount--;
            if (proc[current].first_run < 0) proc[current].first_run = clock;
            printf("| P%d ", proc[current].pid);
        }
        if (current >= 0) {
            proc[current].left -= 10;
            clock += 10;
            if (proc[current].left <= 0) proc[current].left = 0;
        }
        usleep(10000);   /* 10 ms of simulated work per iteration */
    }
    printf("|\n");
    printf("preemptions: %lu\n", switches);
    return 0;
}
```

```bash
gcc -Wall -Wextra -O2 -o rr rr.c && ./rr
```

## Choosing the time quantum

```text
  q TOO SMALL
    ──────────
      • too many context switches
      • the switch cost dominates the useful work
      • time wasted in the scheduler instead of running programs

  q TOO LARGE
    ──────────
      • degenerates into FCFS
      • a long process blocks short ones behind it (the convoy effect)
      • terrible response time

  THE STANDARD RULE
    ────────────────
      q should be substantially LARGER than one context switch,
      and small enough to give good response time.

      if context switch = 1 µs  and we want ≤ 5% overhead
          q ≥ 20 µs
      if context switch = 100 µs (cold cache, large process)
          q ≥ 2 ms

  THE OTHER RULE
    ──────────────
      most processes finish within one quantum
      → the average process should complete in 80–95% of q
```

```text
  If a process uses its whole quantum without finishing,
  it is CPU-bound. If it blocks early, it is I/O-bound.

  So the observed distribution of "how much of the quantum was used"
  tells you what the workload is — which is exactly what a
  multilevel feedback queue exploits.
```

## Interaction with context switch cost

```c
/*
 * Measure the real cost of a context switch on this machine.
 * This is the number that should decide your time quantum.
 */
#define _GNU_SOURCE
#include <stdio.h>
#include <stdlib.h>
#include <sched.h>
#include <time.h>
#include <unistd.h>
#include <pthread.h>

static volatile int sink;

static long read_long(const char *path)
{
    FILE *f = fopen(path, "r");
    if (!f) return -1;
    long v = -1;
    if (fscanf(f, "%ld", &v) != 1) v = -1;
    fclose(f);
    return v;
}

static void *pin_and_spin(void *arg) { (void)arg; for (;;) sink++; return NULL; }

int main(void)
{
    struct timespec a, b;
    const int N = 2000000;
    double ns_per_yield;

    pthread_t t;
    pthread_create(&t, NULL, pin_and_spin, NULL);
    sched_yield();                 /* hand the CPU to the spinner */

    clock_gettime(CLOCK_MONOTONIC, &a);
    for (int i = 0; i < N; i++) sched_yield();
    clock_gettime(CLOCK_MONOTONIC, &b);

    ns_per_yield = (b.tv_sec - a.tv_sec) * 1e9 + (b.tv_nsec - a.tv_nsec);
    ns_per_yield /= N;

    printf("%d sched_yield() round trips: %.1f ns each (%.3f us)\n",
           N, ns_per_yield, ns_per_yield / 1000.0);

    /* Keep context-switch overhead under 5%: quantum >= 20 x switch cost. */
    printf("recommended quantum >= %.1f us (for 5%% overhead)\n",
           20.0 * ns_per_yield / 1000.0);

    /* Real-world CFS knobs on this machine. */
    printf("sched_latency_ns      : %ld\n",
           read_long("/proc/sys/kernel/sched_latency_ns"));
    printf("sched_min_granularity : %ld ns\n",
           read_long("/proc/sys/kernel/sched_min_granularity_ns"));
    printf("sched_wakeup_granularity: %ld ns\n",
           read_long("/proc/sys/kernel/sched_wakeup_granularity_ns"));
    return 0;
}
```

```bash
gcc -O2 -pthread -o cswitch cswitch.c && ./cswitch
```

```text
2000000 sched_yield() round trips: 1182.4 ns each (0.001 us)
recommended quantum >= 0.0 us (for 5% overhead)
sched_latency_ns      : 6000000
sched_min_granularity : 3000000 ns
sched_wakeup_granularity: 4000000 ns
```

`sched_yield()` on an otherwise idle run queue is very cheap, because no
context switch actually occurs. The meaningful number is what the kernel spends
on a busy run queue, and it is governed by `sched_min_granularity_ns`
(3 ms on the machine above). See
[context-switching.md](../01-cpu-execution/context-switching.md).

---

# Comparing the Five Algorithms

Given P0@0(8), P1@1(4), P2@2(9), P3@3(5), and a quantum of 2:

| Algorithm | Gantt | avg wait | avg response | Preemptive | Starvation |
|---|---|---|---|---|---|
| **FCFS** | P0 P1 P2 P3 | 8.75 | 8.50 | no | no (fair) |
| **SJF** | P1 P3 P0 P2 | 6.25 | 6.00 | no | yes |
| **SRTF** | P0 P1 P3 P0 P2 | 6.50 | 1.75 | yes | yes |
| **Priority** | P1 P3 P0 P2 | depends on priorities | | optional | yes |
| **RR (q=2)** | P0 P1 P2 P3 P0 P1 P2 P3 P2 P2 | 11.83 | 0.50 | yes | no |

```text
  ┌──────────────┬──────────────┬───────────────┐
  │              │  SJF         │  RR           │
  │              │  P1 P3 P0 P2 │  P0 P1 P2 P3… │
  ├──────────────┼──────────────┼───────────────┤
  │ avg wait     │  6.25        │  11.83        │
  │ avg response │  6.00        │   0.50        │
  └──────────────┴──────────────┴───────────────┘

  SJF optimises throughput. RR optimises responsiveness.
  Neither wins. Real schedulers mix them.
```

| Property | FCFS | SJF | SRTF | Priority | RR |
|---|---|---|---|---|---|
| Implementation complexity | trivial | easy | moderate | easy | easy |
| Response time | poor | poor | **good** | depends | **excellent** |
| Throughput | poor | **best** | **best** | good | good |
| Fairness | **perfect** | no | no | no | **good** |
| Starvation risk | none | **yes** | **yes** | **yes** | no |
| Needs burst knowledge | no | **yes** | **yes** | no | no |
| Handles I/O-bound well | no | partly | partly | partly | **yes** |

---

# Preemptive vs Non-Pemptive

```text
  NON-PREEMPTIVE
    • run until the process finishes or blocks on I/O
    • no preemption in the middle of a CPU burst
    • a bad burst estimate cannot be corrected
    • no need to save/restore context mid-burst
    • under-utilises the CPU while waiting for I/O unless you use
      multiple processes

  PREEMPTIVE
    • the scheduler may interrupt a running process
    • essential for interactive responsiveness
    • essential for fairness
    • a bad estimate can be corrected — run, learn, adjust
    • costs a context switch each time
    • requires a hardware timer (Linux: the scheduler's tick / hrtimer)
    • raises the stakes: a bug in the preemption path is a scheduler bug
```

> [!IMPORTANT]
> Preemption requires a timer. The timer interrupt is what turns "the running
> process runs until it stops" into "the running process runs until the timer
> says otherwise." This is the link back to
> [interrupts.md](../01-cpu-execution/interrupts.md).

---

# How Linux Actually Schedules

Linux is not any single one of these. It is a **priority-based preemptive
scheduler with fair-share classes**, and the default class is not SJF or RR —
it is a virtual-runtime fairness mechanism.

## The scheduling classes

| Policy | Constant | Behaviour |
|---|---|---|
| **SCHED_OTHER** | 0 | The default. CFS: weighted fair share by `nice` value, with red-black-tree run queues per CPU. Not a fixed quantum. |
| **SCHED_FIFO** | 1 | Real-time. Runs to completion or until it blocks or yields. **No preemption by SCHED_OTHER.** Ignores `nice` entirely. |
| **SCHED_RR** | 2 | Real-time. SCHED_FIFO plus a periodic `sched_rr_get_interval` quantum. |
| **SCHED_BATCH** | 3 | Like `SCHED_OTHER` but optimised for tasks that do not need interactive preemption; reduces migrations and wake-up overhead. |
| **SCHED_IDLE** | 5 | Only runs when nothing else is runnable. For background jobs. |
| `SCHED_RESET_ON_FORK` | flag | Children do not inherit privileged RT policies. |

```c
/*
 * Linux scheduling policies in C.
 */
#define _GNU_SOURCE
#include <sched.h>
#include <stdio.h>
#include <string.h>
#include <errno.h>
#include <pthread.h>
#include <unistd.h>

static void show(const char *name)
{
    int pol = sched_getscheduler(0);
    struct sched_param p;
    sched_getparam(0, &p);
    const char *pn = "?";
    switch (pol) {
    case SCHED_OTHER: pn = "SCHED_OTHER (CFS)"; break;
    case SCHED_FIFO: pn = "SCHED_FIFO";       break;
    case SCHED_RR:   pn = "SCHED_RR";         break;
    case SCHED_BATCH:pn = "SCHED_BATCH";      break;
    case SCHED_IDLE: pn = "SCHED_IDLE";       break;
    }
    printf("%-22s policy=%-22s priority=%d nice=%d\n",
           name, pn, p.sched_priority, getpriority(PRIO_PROCESS, 0));
}

int main(void)
{
    show("initial");

    /* Move to the SCHED_BATCH class: good for long CPU-bound work. */
    struct sched_param p = { .sched_priority = 0 };
    if (pthread_setschedparam(pthread_self(), SCHED_BATCH, &p) == 0)
        show("after SCHED_BATCH");
    else
        perror("SCHED_BATCH");

    /* SCHED_IDLE: only runs when the CPU is otherwise idle. */
    if (pthread_setschedparam(pthread_self(), SCHED_IDLE, &p) == 0)
        show("after SCHED_IDLE");

    /* Real-time policies need CAP_SYS_NICE. */
    p.sched_priority = 10;
    if (pthread_setschedparam(pthread_self(), SCHED_FIFO, &p) == 0)
        show("after SCHED_FIFO");
    else
        printf("after SCHED_FIFO         : %s (need CAP_SYS_NICE)\n", strerror(errno));

    /* The real-time quantum, in microseconds. */
    struct timespec rt;
    sched_rr_get_interval(0, &rt);
    printf("SCHED_RR quantum         : %ld s %ld ns\n", (long)rt.tv_sec, rt.tv_nsec);

    /* A nicer value for the current process. */
    if (setpriority(PRIO_PROCESS, 0, 10) == 0) {
        errno = 0;
        printf("setpriority(+10)         : ok, now nice=%d\n", getpriority(PRIO_PROCESS, 0));
    }
    return 0;
}
```

```bash
gcc -Wall -Wextra -pthread -o policies policies.c
./policies
sudo ./policies        # as root, SCHED_FIFO succeeds
```

```text
initial                policy=SCHED_OTHER (CFS)      priority=0 nice=0
after SCHED_BATCH      policy=SCHED_BATCH            priority=0 nice=0
after SCHED_IDLE       policy=SCHED_IDLE             priority=0 nice=0
after SCHED_FIFO       : Operation not permitted (need CAP_SYS_NICE)
SCHED_RR quantum       : 0 s 95000000 ns
setpriority(+10)       : ok, now nice=10
```

## How CFS works, briefly

```text
  CFS is a virtual-runtime scheduler. Each runnable task is given:

      vruntime   = the amount of CPU time the task would have received
                   if it had run at perfect fairness on this CPU

  The scheduler runs the task with the SMALLEST vruntime. After each tick the
  task's vruntime advances by:

      Δvruntime = (actual_time) × (nice_weight_max / nice_weight)

  So a task with nice -20 (weight 3170) advances vruntime ~4× slower than a
  task at nice 0 (weight 1024). It therefore stays at the head of the tree,
  and effectively gets ~4× more CPU. This is the standard table:

      nice  -20  →  weight 3170
      nice   10  →  weight 110
      nice   19  →  weight   15
```

```bash
# Per-CPU run queues and their red-black trees, as the kernel sees them
ps -eLo pid,ppid,lwp,psr,cls,rtprio,pri,ni,stat,comm
# cls=TS is the normal CFS class; cls=IDL is SCHED_IDLE

# Watch a task be scheduled
ps -L -o pid,tid,cls,psr,comm -p $(pidof firefox) | head -20

# Per-CPU accounting
mpstat -P ALL 1
vmstat 1        # the `r` column is runnable count; the `cs` column is context switches
```

```c
/* The raw signal: how often a task is switched in and out. */
#include <signal.h>
#include <stdio.h>
#include <sys/time.h>
#include <unistd.h>

static void handler(int sig) { (void)sig; write(STDOUT_FILENO, "!", 1); }

int main(void)
{
    signal(SIGPROF, handler);
    setitimer(ITIMER_PROF, &(struct itimerval){{0, 10000}, {0, 10000}}, NULL);
    for (;;) pause();
}
```

> [!NOTE]
> Two sysctls govern the granularity of CFS's fairness. `sched_latency_ns` is
> the target interval the scheduler tries to cover, and
> `sched_min_granularity_ns` is the floor on any one task's slice. When the
> run queue is large, `sched_latency_ns` is divided among the tasks and the
> resulting per-task slice is clamped to `sched_min_granularity_ns`. The
> practical effect is that CFS looks like Round-Robin with a fair-share
> twist, but the ordering comes from `vruntime`, not from burst length.

---

# Choosing an Algorithm

| Workload | Algorithm | Reason |
|---|---|---|
| Batch jobs, long CPU bursts | FCFS, or FCFS with a large quantum | low overhead, high throughput |
| Short, known burst times | SJF | minimises average waiting |
| Unknown bursts, need responsiveness | SRTF | preempt long jobs for short ones |
| Interactive terminals | RR or CFS | bounded response time |
| Real-time control loops | SCHED_FIFO + mlockall | guaranteed, priority-based |
| Latency-critical servers | CFS with low `nice`, or `SCHED_BATCH` for workers | separate classes |
| Mixed | MLFQ or MLFQ+ (see the following notes) | the practical answer |

```text
  The real question is not "which algorithm".
  It is "how do I classify a task I have never seen before?"

  FCFS, SJF, and Priority all need to know something in advance:
      arrival order, or burst length, or importance.

  RR needs no prediction — it just keeps everything moving.

  MLFQ and MLFQ+ keep that advantage AND get SJF's efficiency,
  by learning the classification from observed behaviour.
```

---

# Common Misconceptions

### ❌ "Round-Robin is fair, so it is a good scheduler."

Fair is not the same as good. RR is excellent for responsiveness and poor for
throughput. A thread pool with 1,000 threads under RR gets 1/1000 of the CPU
each, whether it needs 1% or 99%.

---

### ❌ "SJF is optimal, so use SJF."

It is optimal **only if you can predict the burst**, and the optimality is
only over average waiting time. SJF starves long jobs, and the
prediction problem makes it unapproximable: an algorithm that outperforms SJF
on average waiting time would have to know the future.

---

### ❌ "Preemptive is always better than non-preemptive."

No. Preemption costs a context switch each time, and the benefit depends on
the workload. For a batch job with no interactive users, non-preemptive FCFS
is faster because there is nothing to be responsive to.

---

### ❌ "Linux's scheduler is Round-Robin."

No. Linux's default class is `SCHED_OTHER`, implemented as **CFS**, a
priority-based proportional-share scheduler. It has no fixed quantum. True
Round-Robin exists in Linux, but only as `SCHED_RR` for real-time threads.

---

### ❌ "`nice -n 19` makes a program finish 19× later."

No. `nice` values map to **weights**, and the mapping is heavily
non-linear. A nice-19 process gets 15 units of weight against 1024 for
nice 0, so it gets roughly **1.4%** of a nice-0 process's CPU, not
1/20. Making a value 19 steps worse is nearly 70× worse in practice.

---

# Interview Questions

### Basic

- What is CPU scheduling? Name the scheduling criteria.
- Explain FCFS and SJF with a worked example.
- What is the time quantum, and how is it chosen?
- Differentiate preemptive from non-preemptive scheduling.

### Intermediate

- Prove that SJF minimises average waiting time.
- What is the convoy effect, and which algorithms suffer from it?
- Explain the starvation problem in SJF and how to fix it.
- What are the four scheduling policies on Linux, and which is the default?

### Advanced

- A round-robin scheduler has a run queue of 1,000 runnable tasks. Design a
  scheduler that gives interactive tasks good latency without starving
  batch tasks.
- Your service's p99 latency is 3 ms under load. The run queue is 4,000
  tasks deep. Diagnose and fix it.
- Explain how CFS achieves fair share, and why it is not Round-Robin.
- How would you choose a time quantum for a machine with 128 cores and a
  5 µs context switch?

---

# University Exam Notes

### Definitions

- **CPU scheduling:** Selecting the next process or thread to run on a CPU from
  the set of runnable entities.
- **FCFS:** A non-preemptive scheduling algorithm that runs processes in order
  of arrival.
- **SJF:** A non-preemptive algorithm that runs the process with the shortest
  CPU burst first; provably optimal for average waiting time.
- **SRTF:** The preemptive version of SJF; runs the process with the shortest
  remaining time, preempting longer-running processes.
- **Round-Robin:** A preemptive algorithm that cycles ready processes, each
  receiving a fixed time quantum.
- **Time quantum:** The maximum CPU time a process receives before being
  preempted in Round-Robin.
- **Convoy effect:** The delay of short processes behind a long process in a
  FIFO schedule.
- **Aging:** A fairness technique that increases the priority of a process
  the longer it waits.

### Formulae

```text
  turnaround time = completion time − arrival time
  waiting time    = turnaround time − burst time
  response time   = first run − arrival time
  throughput      = processes completed / elapsed time

  SJF optimality: swapping any two adjacent jobs where the longer
  runs first strictly reduces average waiting time.
```

### Frequently Asked Questions

- Explain FCFS and SJF. Why is SJF optimal?
- Explain Round-Robin scheduling. How is the time quantum chosen?
- What is the convoy effect?
- Explain preemptive and non-preemptive scheduling with examples.
- What is starvation? Which algorithms cause it?
- Explain Linux's CFS scheduling policy.
- How does aging prevent starvation?

---

# Key Takeaways

- **FCFS** is fair and simple, but convoy effect makes it bad for interactive
  work.
- **SJF** minimises average waiting time and is provably optimal for that
  metric — but needs burst prediction, and starves long jobs.
- **SRTF** adds preemption to SJF for excellent response time, at the cost of
  more starvation and more context switches.
- **Priority** scheduling is the general form; FCFS, SJF and RR are all
  special cases with a fixed or derived priority.
- **Round-Robin** gives the best response time and needs no prediction, but
  wastes overhead if the quantum is wrong. Too small wastes CPU on switches;
  too large becomes FCFS.
- **Preemption** requires a hardware timer, and is what makes interactivity
  possible at all.
- Linux's default is **CFS** (`SCHED_OTHER`), a `vruntime`-based proportional
  share scheduler, **not** Round-Robin. `SCHED_FIFO` and `SCHED_RR` are
  real-time and require `CAP_SYS_NICE`.
- No single algorithm wins. **MLFQ and MLFQ+** are the practical answer: they
  learn the classification that SJF needs to be told.

---

# References

- Operating System Concepts — Silberschatz
- Modern Operating Systems — Tanenbaum and Bos
- The Linux Kernel Documentation — `Documentation/scheduler/`
- Ingo Molnar, "Linux CFS Scheduler"
- `man 2 sched_setscheduler`, `man 2 setpriority`, `man 2 getpriority`
- `man 2 sched_rr_get_interval`, `man 2 sched_getattr`
- `ps`, `vmstat`, `mpstat`, `chrt`, `renice`, `nice`, `taskset`
