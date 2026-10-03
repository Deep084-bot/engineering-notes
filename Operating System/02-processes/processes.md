# Processes

> [!NOTE]
> **Module:** Module II – Processes
>
> **Difficulty:** ⭐⭐☆☆☆
>
> **Prerequisites:**
> - System Calls
> - Context Switching
>
> **Scope:** Linux and C only. No Windows or Java.

---

# Learning Objective

After reading this note, you should be able to:

- Define a process and distinguish it from a program.
- Describe the five classic states of a process and the transitions between them.
- Read Linux process states from `ps` and `/proc`.
- Describe the structure of a process's virtual address space.
- Create processes with `fork()` and replace their image with `execve()`.
- Explain why `fork()` returns twice.
- Explain zombie and orphan processes.
- Inspect a live process from user space.

---

# Why Do We Need Processes?

## The Problem

You want to run three things at once:

- A web server answering requests
- A text editor saving your file
- A backup job copying 10 GB to an external disk

Running them one after another means the editor freezes for 40 minutes while the
backup runs.

But running them "at the same time" on one CPU is not free either: each one
needs its own program counter, its own stack, its own open files, and its own
identity. Otherwise one program could clobber another's variables.

The operating system needs a unit of isolation that provides exactly this.

That unit is the **process**.

---

# Real World Analogy

A **program** is a blueprint.

A **process** is a construction crew actually following that blueprint.

```text
  Program   = the recipe on the wall, identical for every cook
  Process   = an actual cook, with their own hands, own pan,
              own oven timer, own mistakes
```

You can print the same recipe 500 times. You get 500 recipes, not 500 cooks.

Executing it 500 times gives 500 cooks, each with their own state. Each cook is
a process.

---

# Intuition

```text
  A process is three things at once:

  ┌───────────────────────────────────────────────┐
  │  1. An EXECUTION CONTEXT                      │
  │     where in the code am I? (program counter) │
  │     what is in my registers?                  │
  │     what is on my stack?                      │
  ├───────────────────────────────────────────────┤
  │  2. AN ADDRESS SPACE                          │
  │     a private view of memory: code, data,    │
  │     heap, stack                               │
  ├───────────────────────────────────────────────┤
  │  3. A SET OF RESOURCES                       │
  │     open files, sockets, signal handlers,    │
  │     the PCB, the process group, credentials  │
  └───────────────────────────────────────────────┘
```

Two processes running the *same* program are still different, because they
have different address spaces, different stacks, and different open files.

---

# Program vs Process

| | Program | Process |
|---|---|---|
| Definition | Passive set of instructions on disk | Active entity executing those instructions |
| Stored in | A file (`/bin/ls`) | RAM, in the kernel's task list |
| Identity | None — no PID | Has a unique PID |
| Lifetime | Exists as long as the file does | Has a start and an end |
| Many at once? | Yes, one file, many processes | Yes, each independent |
| Owns memory? | No | Yes, its own address space |
| Example | `/bin/ls` | four concurrent `ls` in four terminals |

```bash
# One program...
file /bin/ls

# ...four processes
ls -d /proc/$(pgrep -x ls)
```

---

# Formal Definition

> **Process:** An instance of a program in execution, consisting of its
> execution context, its private virtual address space, and the kernel-managed
> resources (open files, signals, credentials) associated with it.

On Linux, the concrete representation of a process is the **task_struct**, and
the process is reachable from user space as `/proc/<pid>/`.

---

# The Five Classic Process States

```text
                    ┌──────────┐
                    │   New    │
                    └────┬─────┘
                         │ admitted
                         ▼
    ┌───────────┐     ┌──────────┐
    │  Terminated│     │  Ready   │◄──────────────┐
    │  (exit)    │     └────┬─────┘               │
    └───────────┘          │ dispatched           │ preempted
                          ▼                       │ / yield
                    ┌──────────┐                  │
        ┌──────────►│ Running  │──────────────────┘
        │           └────┬─────┘
        │                │ waits for I/O or an event
        │                ▼
        │           ┌──────────┐
        └───────────┤  Waiting │
        event occurs └──────────┘
```

| State | Meaning | In the ready queue? | On a CPU? |
|---|---|---|---|
| **New** | Being created, resources allocated | no | no |
| **Ready** | Wants the CPU, waiting its turn | **yes** | no |
| **Running** | Currently executing | no | **yes** |
| **Waiting / Blocked** | Waiting for I/O, a signal, or a lock | no | no |
| **Terminated** | Finished, resources being reclaimed | no | no |

## The distinction that matters

```text
  Ready    = I could run right now, I just need a CPU
  Waiting  = I could NOT run even with a CPU, I need something else first
```

A process reading from a keyboard is *waiting*, not *ready*. Giving it a CPU
would achieve nothing.

## Reasons for each transition

| Transition | Triggered by |
|---|---|
| New → Ready | Process admitted; memory and PCB set up |
| Ready → Running | Scheduler picks it; a CPU becomes free |
| Running → Ready | Preempted by a higher-priority task, or `sched_yield()` |
| Running → Waiting | `read()` with no data, `wait()`, lock contention, `sleep()` |
| Waiting → Ready | I/O completes, a signal arrives, a lock is released |
| Running → Terminated | `exit()`, `return` from `main`, or a fatal signal |

---

# Linux Process States

Linux collapses the classic five into a richer set. Read them from `ps`:

```bash
ps -eo pid,ppid,stat,comm | head -20
```

```text
  PID  PPID STAT COMMAND
    1     0 Ss   systemd
    8     1 S    ksoftirqd/0
   11     2 S    kworker/0:1
  512     1 S    systemd-journal
 1031     1 S    dbus-daemon
 2044  1031 S    gnome-shell
 3391  2044 S    firefox
 3412  3391 R    Web Content
 4821  4790 Ss   bash
 5104  4821 S    sleep 60
 7712  5104 D    dd
```

## The state codes

| Code | Name | Meaning |
|---|---|---|
| `R` | Running | Running or runnable — on a CPU or in the run queue |
| `S` | Interruptible sleep | Waiting for an event; **can** be woken by a signal |
| `D` | Uninterruptible sleep | Waiting for I/O; **cannot** be woken or killed |
| `Z` | Zombie | Exited, but the parent has not reaped it |
| `T` | Stopped | Suspended by a job-control signal (`SIGSTOP`) |
| `t` | Tracing stop | Under `ptrace` |
| `X` / `x` | Dead | Exited, being reaped |
| `I` | Idle | Kernel idle thread |

## The two-letter form

```text
  Ss   S = interruptible sleep
            s = session leader

  S+   S = interruptible sleep
            + = in the foreground process group

  R+   R = running
            + = in the foreground process group

  Ss   S = interruptible sleep
            s = session leader (e.g. a login shell)
```

The second letter is not a state — it is a flag.

> [!TIP]
> A process stuck in `D` state is the signature of a broken device or a
> filesystem problem. It cannot be killed with `SIGKILL` because it is not
> executing at all; it is blocked in uninterruptible I/O.

```bash
ps -eo pid,stat,wchan:30,comm | awk '$2 ~ /D/'
```

---

# The Virtual Address Space of a Process

```text
  High addresses
  ┌──────────────────────────────┐  0x7ffffffff000
  │         STACK                │  ← grows DOWN
  │   (local variables,          │     automatic
  │    return addresses,         │     (alloca, VLA)
  │    function arguments)       │
  ├──────────────────────────────┤
  │            ↓                 │
  │                              │
  │         (free)               │
  │                              │
  │            ↑                 │
  ├──────────────────────────────┤
  │         HEAP                 │  ← grows UP
  │   (malloc, calloc, new)      │     dynamic
  ├──────────────────────────────┤
  │         BSS                 │  uninitialised globals
  ├──────────────────────────────┤
  │         DATA                │  initialised globals
  ├──────────────────────────────┤
  │         TEXT (code)         │  read-only, shared
  └──────────────────────────────┘  0x400000
  Low addresses

  Also mapped here (not shown):
    • shared libraries (libc.so, libpthread)
    • the vDSO
    • the stack guard page
    • memory-mapped files
```

## The gap is deliberate

The stack grows down and the heap grows up. The gap between them is unmapped.

If they ever meet, the program crashes. This is caught deliberately:

```c
/* Set a guard page by hand, to see what a stack overflow looks like. */
#include <signal.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

static void on_segv(int sig)
{
    (void)sig;
    write(2, "SIGSEGV: probably a stack overflow\n", 37);
    _exit(1);
}

int main(void)
{
    signal(SIGSEGV, on_segv);

    printf("stack top: %p\n", (void *)__builtin_frame_address(0));
    fflush(stdout);

    volatile char buf[1024];
    while (1)
        buf[0] = 1, buf[sizeof buf - 1] = 2;   /* force the growth */
}
```

```bash
gcc -O0 -o stackoverflow stackoverflow.c
./stackoverflow
stack top: 0x7ffd4c3b2a40
Segmentation fault      ← grew into the guard page
```

## Layout is not a guarantee

The kernel, ASLR, and large pages can move these regions around. Never write
code that depends on where the stack sits relative to the heap.

---

# Creating Processes: fork()

## The Definition

`fork()` creates a **child process** that is a near-copy of the parent.

```c
#include <unistd.h>
pid_t fork(void);
```

| Returns in | Means |
|---|---|
| `-1` | Failed; `errno` is set |
| `0` | You are the **child** |
| `> 0` | You are the **parent**; the value is the child's PID |

## Why fork() Returns Twice

```text
  ┌──────────────┐   fork()   ┌──────────────┐
  │    PARENT    │ ─────────► │    CHILD     │
  │              │            │              │
  │ returns 4822 │            │ returns 0    │
  │ (child's PID)│            │ (no PID yet │
  │              │            │  to return)  │
  └──────────────┘            └──────────────┘

  The kernel:
    1. duplicates the page tables
    2. gives the child a new PID
    3. creates a copy of the task_struct
    4. sets up a fresh kernel stack for the child
    5. sets the child's return value to 0
    6. sets the parent's return value to the child's PID
    7. schedules BOTH

  Same code, two executions, two different return values.
```

## The Classic Example

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/wait.h>

int main(void)
{
    printf("before fork: pid=%d ppid=%d\n", getpid(), getppid());
    fflush(stdout);              /* flush before fork: see the note below */

    pid_t pid = fork();

    if (pid < 0) {
        perror("fork");
        return 1;
    }

    if (pid == 0) {
        /* CHILD */
        printf("child : pid=%d ppid=%d\n", getpid(), getppid());
        fflush(stdout);
        _exit(0);                /* NOT exit(): see below */
    }

    /* PARENT */
    printf("parent: pid=%d ppid=%d, child=%d\n",
           getpid(), getppid(), pid);
    fflush(stdout);

    int status;
    waitpid(pid, &status, 0);
    printf("parent: child %d exited with %d\n", pid, WEXITSTATUS(status));
    return 0;
}
```

```bash
gcc -Wall -Wextra -o forkdemo forkdemo.c
./forkdemo
```

```text
before fork: pid=4821 ppid=4790
parent: pid=4821 ppid=4790, child=4822
child : pid=4822 ppid=4821
parent: child 4822 exited with 0
```

## Why `fflush` before `fork()`

```text
  stdout is FULLY BUFFERED when output is a pipe or a file, and
  LINE BUFFERED when it is a terminal.

  Without the flush, the buffered "before fork" line exists in the
  parent's stdio buffer. fork() copies the buffer into the child.

  Result: the line is printed TWICE.
```

This is one of the most famous surprises in systems programming.

## `_exit()` vs `exit()`

| | `exit()` | `_exit()` |
|---|---|---|
| Flushes stdio buffers | yes | **no** |
| Runs atexit handlers | yes | no |
| Runs destructors | yes | no |
| Ends the process | yes | yes |
| Use in a forked child | risky — **duplicates the parent's buffers** | **correct** |

Using `exit()` in a child after a `fork()` can print duplicated output and run
parent cleanup handlers twice.

---

# fork() + exec(): Starting a New Program

`fork()` alone gives you a copy of the *same* program. To run a different
program, you follow `fork()` with `execve()`, which **replaces** the current
image entirely.

```text
  ┌─────────────────────────────────────────────────────┐
  │  pid=4821  ./myprogram                              │
  │      │                                               │
  │      │  fork()                                       │
  │      ▼                                               │
  │  ┌──────────────┐        ┌──────────────────────┐    │
  │  │ pid=4821     │        │ pid=4822             │    │
  │  │ same program │        │ same program         │    │
  │  │ (parent)     │        │ (child copy)         │    │
  │  └──────────────┘        └──────────┬───────────┘    │
  │                                      │                │
  │                                      │ execve("/bin/ls")│
  │                                      ▼                │
  │                           ┌──────────────────────┐    │
  │                           │ pid=4822             │    │
  │                           │ /bin/ls              │    │
  │                           │ ◄── SAME PID!        │    │
  │                           └──────────────────────┘    │
  └─────────────────────────────────────────────────────┘
```

> [!IMPORTANT]
> `execve()` does **not** change the PID. The process identity, the open file
> descriptors, and the kernel stack all survive. Only the memory image, the
> code, the data, and the stack are replaced. This is why an `exec` failure can
> be reported back to the original caller.

## A Robust Exec

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>

int main(int argc, char **argv)
{
    if (argc != 2) {
        fprintf(stderr, "usage: %s <command>\n", argv[0]);
        return 2;
    }

    pid_t pid = fork();

    if (pid < 0) {
        perror("fork");
        return 1;
    }

    if (pid == 0) {
        /* CHILD */

        /* Reset anything the parent's atexit handlers would otherwise run. */
        atexit(NULL);

        char *const child_argv[] = { argv[1], NULL };
        execvp(argv[1], child_argv);

        /* Only reached if execvp failed. Tell the parent why. */
        perror("execvp");
        _exit(127);
    }

    /* PARENT */
    int status;
    if (waitpid(pid, &status, 0) < 0) {
        perror("waitpid");
        return 1;
    }

    if (WIFEXITED(status))
        return WEXITSTATUS(status);
    if (WIFSIGNALED(status)) {
        fprintf(stderr, "child killed by signal %d\n", WTERMSIG(status));
        return 128 + WTERMSIG(status);
    }
    return 1;
}
```

```bash
gcc -Wall -Wextra -o run run.c
./run /bin/ls
./run /no/such/program
./run /bin/sleep 5
```

## The exit status conventions

```c
WIFEXITED(status)     /* exited normally */
WEXITSTATUS(status)   /* its exit code, 0–255 */
WIFSIGNALED(status)   /* killed by a signal */
WTERMSIG(status)      /* which signal */
WCOREDUMP(status)     /* it dumped core */
WIFSTOPPED(status)    /* stopped (job control) */
WSTOPSIG(status)      /* which stop signal */
```

Shell convention: a process killed by signal *N* appears to the shell as exit
code `128 + N`. `kill -9` becomes 137.

---

# Zombie and Orphan Processes

## The Problem: Who Cleans Up?

When a process exits, the kernel must free its memory, close its files, and
release its PID.

But the kernel keeps the exit status around so the **parent can collect it**.

So the dead process cannot be fully reclaimed until the parent calls `wait()`.

That gap is the zombie.

## Definitions

| Term | Definition |
|---|---|
| **Zombie** | A process that has exited but whose parent has not yet called `wait()`. Its PCB remains; it holds no memory. |
| **Orphan** | A process whose parent has exited. It is immediately reparented. |
| **Reaper / adopter** | The new parent. On Linux this is `systemd` (PID 1) or a subreaper. |

```text
  ZOMBIE
  ───────
  parent          child
    │               │
    │  fork()       │
    │◄──────────────│
    │               │  exit(0)
    │               │──────────►  child is now a ZOMBIE
    │                  PCB kept: exit status, PID, small state
    │                  no memory, no CPU, cannot run
    │               │
    │  ...never calls wait()...
    │               │
    │               │  still a zombie forever
    │
    │  wait()  ──►  zombie is reaped, PCB freed


  ORPHAN
  ──────
  parent          child
    │               │
    │               │  parent exits FIRST
    │  exit(0)       │
    │──────────────► │
    │               │
    │               │  reparented to systemd (PID 1)
    │               │  keeps running normally
    │               │
    │               │  when IT exits, systemd waits() on it
```

## Observing zombies

```bash
ps -eo pid,ppid,stat,comm | awk '$3 ~ /^Z/'
```

```text
   PID  PPID STAT COMMAND
  6101  6001 Z    worker
  6102  6001 Z    worker
  6103  6001 Z    worker
```

```bash
# How many are there?
ps -eo stat | grep -c '^Z'

# Whose are they?
ps -eo ppid,stat | awk '$2 ~ /^Z/ {print $1}' | sort | uniq -c
```

## The C demonstration

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/wait.h>

int main(void)
{
    pid_t pid = fork();

    if (pid == 0)
        _exit(0);                     /* child dies immediately */

    /* Deliberately never call wait(). */
    sleep(30);
    printf("exiting; child %d was a zombie the whole time\n", pid);
    return 0;
}
```

```bash
gcc -Wall -Wextra -o zombie zombie.c
./zombie &
sleep 1
ps -o pid,ppid,stat,comm -p $(pgrep zombie)
cat /proc/$(pgrep zombie)/status | grep -E 'State|PPid'
```

```text
   PID  PPID STAT COMMAND
  6201  6180 Z    zombie
```

```text
State:	Z (zombie)
PPid:	6180
```

The zombie holds a PID slot and a `task_struct` — a few kilobytes. Ten thousand
of them is a real problem, which is why every well-written parent calls
`waitpid()` in a loop.

## The robust pattern: reap everything

```c
#include <errno.h>
#include <signal.h>
#include <stdio.h>
#include <sys/wait.h>
#include <unistd.h>

/* Ask the kernel to notify us about ANY child, asynchronously. */
static void on_sigchld(int sig)
{
    int saved = errno;
    (void)sig;

    for (;;) {
        int status;
        pid_t p = waitpid(-1, &status, WNOHANG);   /* non-blocking */

        if (p == -1) {
            if (errno == ECHILD) break;           /* no children left */
            break;                                 /* EINTR or a real error */
        }
        if (p == 0) break;                         /* children still running */

        /* p > 0: a child was reaped. Handle it. */
        if (WIFEXITED(status))
            fprintf(stderr, "child %d exited %d\n", p, WEXITSTATUS(status));
        else if (WIFSIGNALED(status))
            fprintf(stderr, "child %d killed by %d\n", p, WTERMSIG(status));
    }

    errno = saved;                                 /* handlers must restore it */
}

int main(void)
{
    struct sigaction sa = { .sa_handler = on_sigchld };
    sigemptyset(&sa.sa_mask);
    sa.sa_flags = SA_RESTART | SA_NOCLDSTOP;
    sigaction(SIGCHLD, &sa, NULL);

    for (int i = 0; i < 5; i++)
        if (fork() == 0)
            _exit(0);

    /* Go do real work. Children are reaped as they exit. */
    for (;;)
        sleep(1);
}
```

> [!TIP]
> `SIGCHLD` with `SA_NOCLDSTOP` plus a `WNOHANG` loop in the handler is the
> canonical fix for zombies. Alternatively, set `SIGCHLD` to `SIG_IGN`, which
> makes the kernel auto-reap children — but then you lose the exit status.

---

# Inspecting a Process

## From user space

```bash
ps -eo pid,ppid,stat,pcpu,pmem,rss,vsz,nlwp,comm
```

```text
  PID  PPID STAT  %CPU %MEM   RSS    VSZ NLWP COMMAND
    1     0 Ss    0.0  0.1  21200  172000   1 systemd
 3391  2044 S     4.2  3.8 380000 1400000  12 firefox
 3412  3391 R    18.7  1.2 120000  900000   1 Web Content
 4821  4790 Ss    0.0  0.0   4000    6000   1 bash
```

| Column | Meaning |
|---|---|
| `PID` | process ID |
| `PPID` | parent PID |
| `STAT` | state, with flags |
| `%CPU` | CPU used over the process lifetime |
| `RSS` | resident set size, in KiB — real physical memory |
| `VSZ` | virtual size — everything mapped, including unused |
| `NLWP` | number of threads |

## From /proc

```bash
ls /proc/$$/
```

```text
cmdline  comm  cgroup  environ  exe  fd  limits  maps  mem
net      oom    root    smaps    stack  stat  status  syscall
```

```bash
# The command line, as an argv array
tr '\0' ' ' < /proc/$$/cmdline

# The executable it is running
readlink /proc/$$/exe

# Its open file descriptors
ls -l /proc/$$/fd/

# Resource limits
cat /proc/$$/limits

# Its environment (same security caveats as /proc/PID/environ)
tr '\0' '\n' < /proc/$$/environ | head

# How much of the address space is actually resident
cat /proc/$$/status | grep -E 'VmRSS|VmSize|Threads'
```

## A memory-reading demo

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>

int main(void)
{
    size_t n = 256 * 1024;
    unsigned char *big = malloc(n);
    if (!big) { perror("malloc"); return 1; }
    memset(big, 1, n);

    printf("this process now has %zu KiB resident\n", n / 1024);
    printf("check with: grep VmRSS /proc/%d/status\n", (int)getpid());

    /* Touch one byte per page to make it all genuinely resident. */
    for (size_t i = 0; i < n; i += 4096)
        big[i] = 2;

    printf("touched every page; RSS should now be about %zu KiB\n", n / 1024);
    printf("sleep 30 to inspect it from another terminal\n");
    free(big);
    return 0;
}
```

---

# Common Misconceptions

### ❌ "A process is a program."

Incorrect.

A program is a file. A process is a running instance with its own PID, address
space, stack, and file descriptors. Four terminals running `ls` share one file
and have four processes.

---

### ❌ "fork() creates a copy of the program on disk."

Incorrect.

`fork()` copies the **page tables** and creates a new task_struct. Copy-on-write
means no memory is physically duplicated until one of the two writes to a
shared page.

---

### ❌ "The child starts executing at the line after fork()."

Incorrect.

The child resumes at the **same** instruction — the `fork()` call itself — with
a return value of 0. The instruction after `fork()` is executed by both, but
the branch is different.

---

### ❌ "A zombie process is still using memory and CPU."

Incorrect.

A zombie has exited. It holds no user memory, uses no CPU, and cannot run. It
occupies only a PID slot and a small `task_struct` until the parent reaps it.

---

### ❌ "Killing an orphan process does nothing."

Incorrect.

Orphans are reparented to `systemd` (PID 1), which reaps them. Killing one
works normally.

---

### ❌ "execve() creates a new process."

Incorrect.

`execve()` replaces the image of the *same* process. The PID is preserved.
This is exactly why it can fail and return to the caller.

---

# Interview Questions

### Basic

- What is a process? How is it different from a program?
- What are the five states of a process?
- Why does `fork()` return twice, and what does each return value mean?
- What is a zombie process? How do you prevent it?

### Intermediate

- Explain `fork()`, `execve()` and `waitpid()` and how they combine to run a program.
- What is copy-on-write, and how does `fork()` use it?
- Why should a child call `_exit()` rather than `exit()`?
- What are the Linux process states in `ps`, and what does `D` mean?
- What is the difference between RSS and VSZ?

### Advanced

- Why is `fork()` dangerous in a multithreaded program?
- How does PID reuse work, and how does it create a TOCTOU race in `kill()`?
- How would you detect a fork bomb, and how would you recover without rebooting?
- What happens in the kernel during `fork()`, step by step, on a system with a huge address space?
- Why is `vfork()` obsolete, and what replaced it?

---

# University Exam Notes

### Definitions

- **Process:** An instance of a program in execution, comprising its execution
  context, address space, and kernel-managed resources.
- **Process state:** The stage of a process in its lifecycle — new, ready,
  running, waiting, terminated.
- **Zombie process:** A terminated process whose parent has not yet called
  `wait()`, retaining only its PCB and exit status.
- **Orphan process:** A process whose parent has exited; it is reparented to
  init and continues running.
- **Copy-on-write:** A memory-sharing technique where parent and child share
  physical pages read-only, and the kernel duplicates a page only when one of
  them writes to it.

### Frequently Asked Questions

- What is a process? Differentiate between a program and a process.
- Explain the five states of a process with a state transition diagram.
- What is fork()? Why does it return twice?
- Differentiate between `fork()`, `exec()` and `wait()`.
- What are zombie and orphan processes? Explain with diagrams.
- Explain the memory layout of a process.
- Explain the Linux process states with their `ps` codes.

---

# Key Takeaways

- A **process** is a running program: execution context + address space +
  resources, with a unique PID.
- Ready means "wants a CPU"; waiting means "needs something else first".
- Linux states: `R` running, `S` interruptible sleep, `D` uninterruptible,
  `Z` zombie, `T` stopped, `I` idle.
- `fork()` duplicates the page tables under copy-on-write; the child resumes
  at the same instruction with a return value of 0.
- `execve()` **replaces** the image but keeps the PID, so a failure returns to
  the original caller.
- Always `waitpid()` to reap children, or set `SIGCHLD` to `SIG_IGN`.
- `D` state means uninterruptible I/O — the process cannot be killed, which
  points at the device or filesystem, not at the process.
- `_exit()`, not `exit()`, in a forked child.

---

# References

- Operating System Concepts — Silberschatz
- Modern Operating Systems — Tanenbaum and Bos
- The Linux Programming Interface — Kerrisk (`fork(2)`, `execve(2)`, `wait(2)`, `waitid(2)`)
- `man 3 fork`, `man 2 waitpid`, `man 2 kill`, `man 5 proc`
- `ps(1)`, `top(1)`, `pstree(1)`
- `/proc/PID/stat`, `/proc/PID/status`, `/proc/PID/maps`
