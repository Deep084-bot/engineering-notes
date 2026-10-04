# Process Control Block (PCB)

> [!NOTE]
> **Module:** Module II – Processes
>
> **Difficulty:** ⭐⭐⭐☆☆
>
> **Prerequisites:**
> - Processes
>
> **Scope:** Linux and C only. No Windows or Java.

---

# Learning Objective

After reading this note, you should be able to:

- Define a PCB and explain why the OS needs one.
- List what a PCB stores and why each field matters.
- Explain the concept of process control blocks and process queues.
- Describe the Linux implementation: `task_struct`.
- Use `/proc/PID/` to read the PCB from user space.
- Explain the link between the PCB and a context switch.
- Explain how the scheduler uses the PCB.

---

# Why Do We Need a Process Control Block?

## The Problem

The scheduler needs to answer three questions, thousands of times per second:

1. Which processes exist?
2. What state is each one in?
3. When this process was last running, what was in its registers?

It cannot answer any of these by reading the process's own memory.

Consider: to know where process A was, you must read A's CPU registers. But
reading A's registers means loading them into the CPU — and there is only one
CPU, currently running process B.

The information must live somewhere *outside* the process. It must survive
while the process is not running. It must be reachable from the kernel's
scheduler without touching the process itself.

That storage is the **Process Control Block**.

---

# Real World Analogy

A hospital has 40 patients.

When a doctor makes rounds, they carry a clipboard with one card per patient:

```text
  ┌──────────────────────────────────────────────┐
  │  BED 12 — Room 304, Mrs. Almeida, 67, F       │
  │  ────────────────────────────────────────────  │
  │  Diagnosis: post-operative recovery            │
  │  Medication: 8mg, every 6 hours, next 14:20    │
  │  Notes:  temperature 37.8 at 13:00, improving   │
  │  Vitals (last taken 14:05):                    │
  │     BP 128/80    HR 72    SpO2 97%             │
  │  Attending: Dr. Nakamura                      │
  │  Next review: 18:00                            │
  └──────────────────────────────────────────────┘
```

The patient is the **process**: doing something, or recovering.

The card is the **PCB**: everything needed to pick up care seamlessly, no
matter which staff member takes over next.

Walking from bed 12 to bed 19 is a **context switch**. The patient does not
move; only the doctor's attention moves.

---

# Intuition

```text
  ┌─────────────────────────────────────────────────────────────┐
  │                    PROCESSES (user memory)                  │
  │  ┌────────┐   ┌────────┐   ┌────────┐                      │
  │  │ Proc A │   │ Proc B │   │ Proc C │   ← code, data, heap  │
  │  └────────┘   └────────┘   └────────┘                      │
  └─────────────────────────────────────────────────────────────┘
            ▲                ▲                 ▲
            │                │                 │
       ┌────┴────┐     ┌─────┴─────┐    ┌──────┴──────┐
       │ PCB (A) │     │ PCB (B)   │    │ PCB (C)     │
       └─────────┘     └───────────┘    └─────────────┘
            ▲                ▲                 ▲
            └────────────────┴─────────────────┘
                             │
                  ┌──────────┴──────────┐
                  │   THE SCHEDULER     │
                  │  decides who runs   │
                  │  next, using only    │
                  │  the PCBs            │
                  └─────────────────────┘
```

The scheduler never reads the process's own memory. It reads only PCBs.

---

# Formal Definition

> **Process Control Block (PCB):** A data structure maintained by the operating
> system containing all the information the kernel needs to manage a process
> across time — including its current state, saved CPU context, memory
> mappings, open resources, and identity.

| Property | Value |
|---|---|
| Location | Kernel memory, never directly accessible from user space |
| Created | At process creation (`fork`, `clone`, `exec`) |
| Destroyed | When the process is reaped by its parent |
| Size (Linux) | `task_struct` ≈ 9–10 kB on a modern x86-64 kernel |
| User-space view | Exposed as `/proc/PID/` |
| Used by | Scheduler, signal delivery, memory manager, I/O subsystem |

---

# What a PCB Stores

## 1. Process Identity

```text
  PID             unique process identifier
  PPID            parent process identifier
  PGID            process group id  (job control)
  SID             session id        (login sessions)
  UID, GID        owner credentials
  TTY             controlling terminal
  starttime       when it was created (for accounting and anti-spoofing)
```

## 2. Processor State — the part that makes context switching possible

```text
  Program counter      RIP — where to resume
  CPU registers        RAX RBX RCX RDX RSI RDI RBP RSP R8–R15
  Status register      RFLAGS — carry, zero, sign, interrupt flag
  Stack pointer        RSP
  Floating point       x87 stack, XMM0–15, YMM, ZMM
  Kernel stack         RSP within the kernel
  Scheduling info      the saved thread_info struct
```

## 3. CPU Scheduling Information

```text
  State              R, S, D, Z, T...
  Priority / nice    static priority, nice value (-20 … 19)
  Scheduling policy  SCHED_OTHER, SCHED_FIFO, SCHED_RR, SCHED_BATCH, SCHED_IDLE
  Scheduling params   priority, policy, flags, runtime, deadline
  vruntime            the CFS "virtual" runtime — its fair-share score
  Timeslices used     for round-robin accounting
  CPU affinity        which CPUs this task may run on
  NUMA node, core     topology placement
```

## 4. Memory Management Information

```text
  Address-space map   all VMA (virtual memory area) structures
  Page table root     mm_struct → pgd
  RSS and VSZ         resident and virtual sizes
  Signal handlers     sigaction table, sigstack
  Pending signals     queue of signals not yet delivered
  Memory limits       rlim_t array, from setrlimit()
```

## 5. I/O and Resource Information

```text
  Open files          file descriptor table, one pointer per fd
  File pointers       f_pos, mode, flags per fd
  Root directory      cwd
  Root filesystem     root
  Namespaces          pid, mnt, uts, ipc, net, user, cgroup
  I/O context         ioac, rq, aio
  I/O priority        ioprio
```

## 6. Miscellaneous

```text
  Exit code           the value passed to exit()
  Signal to send      on termination
  Personality         execution domain
  Thread group        which processes share this thread group
  Security            LSM labels, capabilities, seccomp
  Audit               audit context
```

## The class-diagram view

```text
  ┌────────────────────────────────────┐
  │            PCB                     │
  ├────────────────────────────────────┤
  │ Process State                      │
  │ Process ID, Program Counter        │
  │ CPU Registers, CPU Scheduling Info │
  │ Memory Management Info             │
  │ Accounting Information             │
  │ I/O Status                         │
  │ Open Files                         │
│ Signal Handling                     │
  │ CPU Affinity, NUMA node            │
  └────────────────────────────────────┘
```

---

# Process Control Queues

The OS keeps PCBs in organised collections, mirroring the process states.

```text
  ┌──────────────────────────────────────────────────────────────┐
  │                    PROCESS QUEUES                            │
  ├──────────────────────────────────────────────────────────────┤
  │                                                              │
  │  ┌──────────────┐                                            │
  │  │ Job Queue    │  → new processes waiting to be admitted     │
  │  └──────┬───────┘                                            │
  │         │ admit                                              │
  │         ▼                                                    │
  │  ┌──────────────┐                                            │
  │  │ Ready Queue  │  → Runnable. The scheduler picks from here  │
  │  └──────┬───────┘                                            │
  │         │ dispatch                                           │
  │         ▼                                                    │
  │  ┌──────────────┐                                            │
  │  │ Running      │  → at most one per CPU core                │
  │  └──────┬───────┘                                            │
  │         │ I/O request / wait                                 │
  │         ▼                                                    │
  │  ┌──────────────┐                                            │
  │  │ Waiting Queue│  → blocked on I/O, a signal, or a lock      │
  │  └──────┬───────┘                                            │
  │         │ event completes                                     │
  │         └──────────────► back to Ready Queue                 │
  │                                                              │
  │  ┌──────────────┐                                            │
  │  │ Terminated   │  → exited, awaiting reap by the parent     │
  │  │ (zombie)     │                                            │
  │  └──────────────┘                                            │
  └──────────────────────────────────────────────────────────────┘
```

| Queue | Membership criterion | Chosen by |
|---|---|---|
| Job queue | Created but not yet admitted | The admission controller |
| Ready queue | Wants a CPU | The scheduler |
| Waiting queue | Blocked on an event | The event that unblocks it |
| Terminated | Exited, not reaped | The parent's `wait()` |

> [!TIP]
> In Linux these are not literal FIFO queues. The ready "queue" is a
> red-black tree keyed on `vruntime` (CFS), and blocked tasks sit on per-event
> wait queues (`wait_queue_head_t`). The conceptual model is the queue; the
> implementation is a priority structure.

---

# The Linux Implementation: task_struct

## Where it lives

```text
  <linux/sched.h>

  struct task_struct {
      /* ---- processor state ---- */
      unsigned long          state;          /* R, S, D, Z, T ... */
      void                  *stack;          /* kernel stack base */

      /* ---- scheduling ---- */
      unsigned int           prio;           /* static priority, 120 = normal */
      int                    nice;           /* -20 … 19 */
      unsigned long          vruntime;       /* CFS fair-share score */
      struct sched_entity    se;             /* the sched_entity for CFS */

      /* ---- identity ---- */
      pid_t                  pid;
      pid_t                  tgid;           /* thread group leader's pid */
      struct task_struct    *parent;
      struct list_head       children;       /* our child PCBs */
      struct list_head       sibling;        /* link in parent's child list */

      /* ---- mm ---- */
      struct mm_struct      *mm;             /* address space (NULL for kernel threads) */
      struct mm_struct      *active_mm;
      struct vm_area_struct *mmap;           /* VMA list */

      /* ---- files ---- */
      struct files_struct   *files;          /* fd table */

      /* ---- signals ---- */
      struct signal_struct  *signal;
      sigset_t               blocked;
      sigset_t               real_blocked;
      sigset_t               pending;
      struct sighand_struct *sighand;       /* one per signal number */

      /* ---- credentials ---- */
      uid_t                  euid, uid;
      gid_t                  egid, gid;

      /* ---- topology ---- */
      int                    cpu;            /* last CPU it ran on */
      int                    on_rq;          /* is it on a run queue? */
      struct list_head      tasks_node;      /* link in runqueue/other lists */

      /* ---- exit ---- */
      int                    exit_state;
      int                    exit_code;
      struct list_head      sibling;
      /* ... ~700 more lines in the real header ... */
  };
```

## Tracing the links

```c
/* Walk the process tree from any PID, using only the documented fields. */
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <dirent.h>
#include <ctype.h>

static int read_int_field(const char *dir, const char *file,
                          const char *key, long *out)
{
    char path[512];
    snprintf(path, sizeof path, "%s/%s", dir, file);

    FILE *f = fopen(path, "r");
    if (!f) return -1;

    char line[512];
    int rc = -1;
    while (fgets(line, sizeof line, f)) {
        if (strncmp(line, key, strlen(key)) == 0) {
            *out = strtol(line + strlen(key), NULL, 10);
            rc = 0;
            break;
        }
    }
    fclose(f);
    return rc;
}

int main(void)
{
    long target = 1;
    int depth = 0;

    printf("process tree from PID 1\n");
    for (int d = 0; d < 40; d++) {
        DIR *pdir = opendir("/proc");
        if (!pdir) return 1;
        struct dirent *de;
        int found = 0;

        while ((de = readdir(pdir))) {
            if (!isdigit((unsigned char)de->d_name[0])) continue;

            char dir[512];
            snprintf(dir, sizeof dir, "/proc/%s", de->d_name);

            long ppid = -1;
            if (read_int_field(dir, "status", "PPid:", &ppid) != 0)
                continue;

            if (ppid == target) {
                char comm[256] = "?";
                char cp[512];
                snprintf(cp, sizeof cp, "%s/comm", dir);
                FILE *f = fopen(cp, "r");
                if (f) { if (fgets(comm, sizeof comm, f)) {} fclose(f); }
                comm[strcspn(comm, "\n")] = '\0';

                long state = 0;
                read_int_field(dir, "stat", "", &state);
                (void)state;

                printf("%*s%s (%s)\n", depth * 2, "", de->d_name, comm);
                target = atol(de->d_name);
                found = 1;
                break;
            }
        }
        closedir(pdir);

        if (!found) break;
        depth++;
    }
    return 0;
}
```

```bash
gcc -Wall -Wextra -o tree tree.c
./tree
```

```text
process tree from PID 1
1 (systemd)
  891 (systemd-journal)
  1031 (dbus-daemon)
  2044 (gnome-shell)
    3391 (firefox)
      3412 (Web Content)
      3555 (Web Content)
```

> [!NOTE]
> The pure-C version of this is easy: read `/proc/PID/stat` and scan for the
> fields. The version above uses `PPid` from `status` for clarity, since the
> second field of `stat` (`comm`) can contain spaces and parentheses.

---

# Reading the PCB from User Space: /proc

`/proc` is a filesystem whose files are **generated by reading the PCB**. This
is the practical payoff of the concept.

```bash
# The single most useful file: /proc/PID/stat
cat /proc/$$/stat
```

```text
4821 (bash) S 4790 4821 4821 34816 4831 4194304 1123 0 0 0 5 12 0 0 20 0 1 0
12345 1700000 912 18446744073709551615 94000000000000 94000000000000 ...
```

## Field by field

| # | Field | Meaning |
|---|---|---|
| 1 | `pid` | process ID |
| 2 | `comm` | executable name, in parentheses |
| 3 | `state` | `R S D Z T t X x I` |
| 4 | `ppid` | parent PID |
| 5 | `pgrp` | process group |
| 6 | `session` | session ID |
| 7 | `tty_nr` | controlling terminal |
| 8 | `tpgid` | foreground process group of the tty |
| 9 | `flags` | flags (see below) |
| 10 | `minflt` | minor faults (lazy paging) |
| 11 | `cminflt` | minor faults, with children |
| 12 | `majflt` | major faults (actual disk reads) |
| 13 | `cmajflt` | major faults, with children |
| 14 | `utime` | user-mode CPU ticks |
| 15 | `stime` | kernel-mode CPU ticks |
| 16 | `cutime` | children's user ticks |
| 17 | `cstime` | children's kernel ticks |
| 18 | `priority` | static priority (100–130, lower = better) |
| 19 | `nice` | nice value, -20 … 19 |
| 20 | `num_threads` | thread count |
| 21 | `itrealvalue` | obsolete, was for `setuid` timers |
| 22 | `starttime` | start time in clock ticks since boot |
| 23 | `vsize` | virtual size in bytes |
| 24 | `rss` | resident set size in pages |
| 25 | `rsslim` | RSS soft limit |
| 26 | `startcode` | start of `.text` |
| 27 | `endcode` | end of `.text` |
| 28 | `startstack` | bottom of stack |
| 30 | `esp` | current stack pointer |
| 31 | `eip` | **the saved instruction pointer** |
| 32 | `eflags` | **the saved flags register** |
| 33–41 | `sig*` | pending and blocked signals |
| 42 | `sptr` | saved user SP (deprecated) |
| 43 | `kstkesp` | **kernel stack pointer** |
| 44 | `kstkeip` | **kernel instruction pointer** |
| 45 | `signal` | pending bitmap |
| 45–64 | `blocked`, `sigignore`, `sigcatch` | signal masks |
| 65–77 | thread CPU times, `processor`, `rt_priority`, policy, delayacct |
| 78–... | guest time, cpuset, scheduling info, `start_data`, `end_data`, `start_brk`, `arg_start`, `arg_end`, `env_start`, `env_end`, `exit_code` | |

Fields 31, 32 and 43, 44 are the **processor state** portion of the PCB: exactly
the fields a context switch saves and restores.

## /proc/PID/status — human readable

```bash
cat /proc/$$/status
```

```text
Name:	bash
Umask:	0022
State:	S (sleeping)
Tgid:	4821
Ngid:	0
Pid:	4821
PPid:	4790
TracerPid:	0
Uid:	1000	1000	1000	1000
Gid:	1000	1000	1000	1000
FDSize:	64
Groups:	27 100
VmPeak:	  123456 kB
VmSize:	  122000 kB
VmLck:	       0 kB
VmPin:	       0 kB
VmHWM:	    6100 kB
VmRSS:	    4000 kB
RssAnon:	   2200 kB
RssFile:	   1800 kB
RssShmem:	      0 kB
VmData:	    1024 kB
VmStk:	     132 kB
VmExe:	     124 kB
VmLib:	    2100 kB
VmPTE:	     536 kB
VmSwap:	       0 kB
Threads:	1
SigQ:		31263/307786
SigPnd:	0000000000000000
ShdPnd:	0000000000000000
SigBlk:	0000000000010000
SigIgn:	0000000000000001
SigCgt:	00000000000043fe
CapInh:	0000000000000000
CapPrm:	0000003fffffffff
CapEff:	0000003fffffffff
CapBnd:	0000003fffffffff
NoNewPrivs:	0
Seccomp:	0
Cpus_allowed:	ff
Cpus_allowed_list:	0-7
voluntary_ctxt_switches:	38214
nonvoluntary_ctxt_switches:	1042
```

> [!TIP]
> `VmHWM` (peak RSS) is invaluable: it is the **high water mark**. If a
> process peaked at 8 GB but now shows 200 MB, it had a large transient
> allocation — usually a large `realloc` or an oversized buffer.

## /proc/PID/stat flags field

```c
#define PF_KTHREAD        (1 << 0)   /* kernel thread */
#define PF_VM             (1 << 1)   /* VM (memory pressure) bookkeeping */
#define PF_NOFreeze       (1 << 2)   /* frozen for suspend */
#define PF_FORKNOEXIT     (1 << 3)   /* parent died, child must exit */
#define PF_SUPERVISOR     (1 << 4)
#define PF_MCE_EARLY      (1 << 5)   /* early MCE kill */
#define PF_LINUXTHREAD    (1 << 6)   /* is a thread of a process */
#define PF_KTHREAD        (1 << 0)
#define PF_IO_WORKER      (1 << 7)
#define PF_MEMALLOC       (1 << 8)   /* includes vmalloc'ed memory */
#define PF_NOFreeze       (1 << 2)
#define PF_SUPERVISOR     (1 << 4)
#define PF_RANDOM_CORE    (1 << 9)
#define PF_NO_SETUID_HASH (1 << 10)
```

---

# The PCB and the Context Switch

The relationship is direct and mechanical.

```text
  ┌─────────────────────────────────────────────────────────────┐
  │                    PCB (task_struct)                        │
  │                                                             │
  │   state  ─────────────────────────────────┐                │
  │   stack  ────────────────────┐            │                │
  │   vruntime ────────┐         │            │                │
  │   prio/nice ───────┤         │            │                │
  │   mm / files /     │         │            │                │
  │   signals          │         │            │                │
  │                    ▼         ▼            ▼                │
  │   ┌────────────────────────────────────────────┐            │
  │   │  KERNEL STACK                              │            │
  │   │  ┌──────────────────────────────────────┐  │            │
  │   │  │  saved RIP    ← eip in /proc/stat    │  │            │
  │   │  │  saved RFLAGS ← eflags               │  │            │
  │   │  │  saved RSP    ← esp                  │  │            │
  │   │  │  saved RBP, RBX, R12–R15            │  │            │
  │   │  │  (the syscall/interrupt frame)       │  │            │
  │   │  └──────────────────────────────────────┘  │            │
  │   └────────────────────────────────────────────┘            │
  │        ▲                        ▲                           │
  │        │                        │                           │
  │   scheduler reads          scheduler pushes here            │
  │   stack, prio, vruntime    before switching away            │
  └─────────────────────────────────────────────────────────────┘
```

## The two directions

```text
  SWITCHING OUT (saving into the PCB)
  ─────────────────────────────────────
    1. Push callee-saved registers onto the CURRENT kernel stack.
    2. Save the kernel stack pointer into current->stack.
    3. state = context_switch();          // the PCB's `state` field
    4. Save the full user frame (RIP, CS, RFLAGS, RSP, SS) onto the kernel
       stack — this happens on the way IN, at syscall/interrupt entry.
    5. If preempted, record vruntime into the PCB so the scheduler can
       account for the CPU time consumed.


  SWITCHING IN (restoring from the PCB)
  ─────────────────────────────────────
    1. next->stack holds the kernel stack pointer. Load RSP from it.
    2. Pop the callee-saved registers off that stack.
    3. state = TASK_RUNNING;              // ready
    4. Eventually iretq pops the user frame, restoring RIP/RSP/RFLAGS.
```

```bash
# Watch the `state` field of a specific PCB change in real time
watch -n0.2 "awk '{print \$1, \$3, \$14, \$15}' /proc/$(pgrep -n yourprog)/stat"
```

```text
  PID S  utime stime      ← the processor-state + accounting fields
 4821 S      12     3
 4821 R      13     3
 4821 S      13     3
 4821 R      13     4      ← utime/stime climbing = it is running
```

---

# Where the PCB Lives in the Kernel's Data Structures

```text
  All tasks
     │
     ├── Running / runnable
     │      └── per-CPU runqueue  (cfs_rq)
     │             └── rbtree of sched_entity, keyed on vruntime
     │                  └── each sched_entity embeds a pointer to the task
     │
     ├── Blocked on I/O
     │      └── wait_queue_head_t for that device
     │             └── wait_queue_entry per waiter
     │
     ├── Blocked on a mutex
     │      └── mutex->wait_list
     │
     ├── Sleepy (interruptible)
     │      └── the task is on a timer list or a futex wait queue
     │
     ├── Exited, awaiting reap
     │      └── on its parent's children list, state = Z
     │
     └── Suspended
            └── a hash table keyed by pid, plus a PID bitmap
```

## Finding the runqueue

```bash
# Which CPU is this task's vruntime recorded on?
cat /proc/$$/stat | awk '{print "last CPU:", $39}'

# How many runnable tasks?
cat /proc/loadavg
# 0.52 0.58 0.59  2/1387 20114
#          ^ 2 runnable, 1387 total tasks
```

```bash
# Per-CPU runqueue state, if CONFIG_SCHED_DEBUG is on
grep . /sys/kernel/debug/sched/debug 2>/dev/null | head -20

# Or the tracepoint, which always works
perf list | grep sched
perf record -e sched:sched_switch -a -- sleep 5
perf script | head -20
```

```text
sched_switch:  prev_comm=firefox prev_pid=3391 prev_state=S
               ==> next_comm=Web Content next_pid=3412 next_prio=120
```

---

# The PCB and Process Queues: Putting It Together

```text
  Process state            Where the PCB is
  ─────────────            ────────────────
  New (admission)          on the CPU's init task list, or in a cgroup
  Ready                     on a per-CPU runqueue, inside a sched_entity
  Running                   on a runqueue, marked on_cpu == 1
  Waiting (I/O)             on the device's wait_queue_head_t
  Waiting (mutex)           on the mutex's wait_list
  Waiting (futex)           in a futex bucket's hash table
  Sleeping (timed)          on a timer list, or in the hrtimer queue
  Stopped                   in a hash table keyed by pid
  Zombie                    on the parent's children list
```

> [!IMPORTANT]
> There is no single "PCB array" in Linux. A process may be on a runqueue
> *and* a wait queue *and* a timer list simultaneously. Membership is tracked
> by list pointers inside the `task_struct`, not by a state field alone.

---

# Process States and Their Effect on the PCB

| State | `state` value | On a runqueue? | Counts as running? |
|---|---|---|---|
| Running (on CPU) | `TASK_RUNNING` (0) | yes, `on_rq == 1` | yes |
| Ready | `TASK_RUNNING` (0) | yes, `on_rq == 1` | no |
| Interruptible sleep | `TASK_INTERRUPTIBLE` (1) | no | no |
| Uninterruptible sleep | `TASK_UNINTERRUPTIBLE` (2) | no | no |
| Stopped (signal) | `TASK_STOPPED` (4) | no | no |
| Traced | `TASK_TRACED` (8) | no | no |
| Zombie | `TASK_ZOMBIE` (9) | no | no |
| Dead (being reaped) | `TASK_DEAD` (14) | no | no |
| Parked | `TASK_PARKED` (4) | no | no |

> [!NOTE]
> This explains the `ps` output. `R` covers **both** running and ready,
> because both are `TASK_RUNNING` in the kernel. `ps` adds the `+` or `s`
> flags from the terminal fields to disambiguate.

---

# Common Misconceptions

### ❌ "The PCB lives in the process's own memory."

Incorrect.

The PCB is in **kernel** memory. If it were in user space, a malicious process
could overwrite its own PID, its state, or its parent pointer. `/proc/PID/` is
only a *view* generated by the kernel on read.

---

### ❌ "The PCB stores the process's whole memory contents."

Incorrect.

The PCB stores *pointers* to the address space (`mm_struct`), not the contents.
Swapping out a process does not touch the PCB beyond a few fields.

---

### ❌ "A context switch copies the PCB."

Incorrect.

Nothing is copied between processes. The kernel saves the *current* CPU context
onto the outgoing process's kernel stack, and later restores it. The PCB is not
the destination — the stack is.

---

### ❌ "There is one queue containing all ready processes."

Incorrect, on a multicore system.

There is one run queue **per CPU**. Linux load-balances between them. The
conceptual "Ready Queue" is a teaching abstraction.

---

### ❌ "A zombie still has its PCB with memory mappings."

Incorrect.

A zombie's `task_struct` remains, but `mm` is NULL, `files` is empty, and all
user memory is released. It is a few kilobytes, not the original footprint.

---

# Interview Questions

### Basic

- What is a PCB? What information does it contain?
- Why does the OS need a PCB?
- What is the relationship between the PCB and a context switch?

### Intermediate

- Explain the process control queues and how a process moves between them.
- What is `task_struct` and what are its main members?
- How is the PCB exposed to user space in Linux?
- What is the difference between a runqueue and a wait queue?

### Advanced

- Where is the saved register state actually stored, and why is it not in `task_struct` directly?
- How does the kernel find a process by PID?
- What would break if `task_struct::stack` were corrupted?
- How does the scheduler use `vruntime` and `on_rq`?
- Why does `ps` show `R` for both running and ready processes, and how does it distinguish them?

---

# University Exam Notes

### Definitions

- **PCB:** A kernel data structure holding all information required to manage a
  process across time, including its state, CPU context, memory mappings and
  open resources.
- **Process Control Queues:** Organised collections of PCBs corresponding to
  process states — job, ready, waiting and terminated queues.
- **Job Queue:** The queue of newly created processes awaiting admission to the
  ready queue.
- **Ready Queue:** The collection of processes that are ready to run and are
  awaiting CPU dispatch.
- **Waiting Queue:** The collection of processes blocked on an event.
- **task_struct:** The Linux kernel structure representing a process or thread.

### Frequently Asked Questions

- What is a PCB? Explain its contents with a diagram.
- What is the need for process control blocks?
- Explain the process control queues and the process state transition diagram.
- What are the fields of `/proc/PID/stat`?
- How does the PCB help the scheduler?
- What is `task_struct`? Explain the Linux process representation.

---

# Key Takeaways

- The PCB is the kernel's private record of a process. The scheduler uses
  *only* PCBs, never the process's own memory.
- It stores identity, **processor state**, scheduling info, memory info,
  resources and signal state.
- The processor-state portion is what makes a context switch possible.
- Linux implements it as `task_struct` in `<linux/sched.h>`.
- `/proc/PID/stat`, `/proc/PID/status` and `/proc/PID/task/` are the user-space
  view of the PCB.
- PCBs are linked into per-CPU **runqueues**, per-device **wait queues**, and
  parent **children lists** — several lists at once, not one queue.
- Fields 31, 32, 43, 44 of `/proc/PID/stat` (`eip`, `eflags`, `kstkesp`,
  `kstkeip`) are the saved CPU context.
- A zombie retains its `task_struct` but has `mm == NULL` — no user memory.

---

# References

- Operating System Concepts — Silberschatz
- Modern Operating Systems — Tanenbaum and Bos
- The Linux Programming Interface — Kerrisk
- Linux Kernel Development — Robert Love (see `task_struct`, `schedule()`)
- `/proc/PID/stat` — the definitive field reference
- `man 5 proc`, `man 3 proc_pid_stat`
- `ps(1)`, `top(1)`, `pstree(1)`
