# Threads

> [!NOTE]
> **Module:** Module II – Processes
>
> **Difficulty:** ⭐⭐⭐☆☆
>
> **Prerequisites:**
> - Processes
> - Process Control Block
> - Context Switching
>
> **Scope:** Linux and C only. No Windows or Java.

---

# Learning Objective

After reading this note, you should be able to:

- Define a thread and explain how it differs from a process.
- Describe what is shared and what is private between threads.
- Explain how Linux creates threads using `clone()`.
- Create, join and detach threads with pthreads.
- Explain thread-local storage and why `errno` is per-thread.
- Explain the thread lifecycle and thread states.
- Inspect threads from user space.
- Explain when threads are the right tool and when they are not.

---

# Why Do We Need Threads?

## The Problem

Consider a web server that serves 1000 clients.

Each client wants a page, which requires:

1. Read the request (blocking)
2. Read a file from disk (blocking)
3. Query a database (blocking)
4. Send the response

If the server handles one client at a time, it spends 99.9% of its time
**waiting**. Meanwhile 999 other clients are queued behind a socket that is
idle.

If you fork 1000 processes instead, each gets its own address space. The same
connection table, the same configuration, the same 200 MB of shared read-only
data now exist 1000 times over.

Threads are the middle ground: many independent streams of execution, sharing
one address space.

---

# Real World Analogy

A **process** is an office with its own letterhead, its own stationery, its own
copies of the filing system, and its own door key. Nothing is shared, so
nothing can be corrupted by a colleague. But photocopying the whole office 1000
times is expensive, and a note passed to a colleague requires walking down the
corridor.

A **thread** is 20 people working in the same office. They share the filing
system, the stationery and the printer, so they can pass notes instantly and use
one copy of everything. But now they can also tread on each other's toes: two
people updating the same spreadsheet at the same time produce garbage.

The choice is: **isolation (processes)** or **efficiency (threads)**.

---

# Intuition

```text
  PROCESSES:  isolation

     Process A            Process B
   ┌───────────┐        ┌───────────┐
   │  code     │        │  code     │     ← duplicated
   │  data     │        │  data     │
   │  heap     │        │  heap     │
   │  stack    │        │  stack    │
   │  fd table │        │  fd table │
   │  ───────  │        │  ───────  │
   │  kernel   │        │  kernel   │
   └───────────┘        └───────────┘
        │ IPC
        └──────────────►


  THREADS:  one process, many execution streams

                    ┌─────────────────────────────────┐
   Process          │  code     data     heap   (shared)│
                    │                                 │
     ┌────────┐     │  ┌────┐ ┌────┐ ┌────┐ ┌────┐     │
     │ Thread │     │  │stk1│ │stk2│ │stk3│ │stk4│     │
     │  (PC)  │     │  └────┘ └────┘ └────┘ └────┘     │
     └────────┘     │   PC1  PC2  PC3  PC4   (private) │
     ┌────────┐     │   TLS1 TLS2 TLS3 TLS4  (private) │
     │ Thread │     │   fd table (shared)               │
     │  (PC)  │     │   PCB per thread                  │
     └────────┘     └─────────────────────────────────┘
```

---

# Formal Definition

> **Thread:** A basic unit of CPU execution within a process, with its own
> program counter, register set and stack, sharing the process's address space,
> open files and other resources.

| | Process | Thread |
|---|---|---|
| Also called | task, program in execution | lightweight process, flow of control |
| Has its own PCB? | yes | yes, one per thread |
| Address space | private | **shared** with siblings |
| Has its own stack | yes | yes |
| Has its own heap | yes (separate) | no — shared |
| Open files | own fd table | **shared** fd table |
| Signal handlers | own | shared (delivered to one thread) |
| Context switch cost | 1–10 µs (TLB flush) | 0.5–2 µs |
| Creation cost | ~0.5–1 ms | ~20–50 µs |
| Communication | IPC mechanisms | **plain shared variables** |
| Isolation | strong | none |

---

# What Is Shared and What Is Private

## Shared between threads of one process

```text
  ✔ Code (text segment)         read-only, shared
  ✔ Global and static variables  .data and .bss
  ✔ Heap                         malloc'd memory
  ✔ Open file descriptors       one table, shared f_pos!
  ✔ Current working directory
  ✔ Signal dispositions          the sigaction table
  ✔ Address space                the whole VM
  ✔ mm_struct                    one per process
  ✔ File table (struct files)    one per process
```

## Private per thread

```text
  ✘ Program counter and registers     each thread's own
  ✘ Stack                            each thread's own
  ✘ Thread-local storage (TLS)        __thread / pthread_key_t
  ✘ errno                             per-thread by design
  ✘ Signal mask                       sigprocmask is per-thread
  ✘ Scheduling parameters             chrt/nice can be per-thread
  ✘ Robust mutex list                 per-thread
  ✘ Its own task_struct               one PCB-like structure each
```

## The dangerous shared state

```c
#include <pthread.h>
#include <stdio.h>
#include <unistd.h>

static FILE *logfile;                  /* SHARED */

static void log_msg(const char *who, const char *what)
{
    /*
     * This looks atomic. It is three steps, and stdio has an internal lock
     * that is held per-call, not per-call-chain. Two threads can interleave
     * here and produce:
     *      [thread-A]hello wor
     *      [thread-B]hello world
     *      [thread-A]ld
     */
    fprintf(logfile, "[%s] %s\n", who, what);
    fflush(logfile);
}
```

And the worse case, shared file position:

```c
static int fd;

/* Thread A writes 10 bytes, thread B writes 10 bytes. */
write(fd, bufA, 10);
write(fd, bufB, 10);
```

```text
  f_pos is ONE value in ONE shared file table.

  Thread A: write() → the kernel appends bufA, f_pos += 10
  Thread B: write() → the kernel appends bufB, f_pos += 10

  Result: the file contains A then B. Fine.

  But with O_APPEND and multiple writers, and with buffered stdio, or with
  lseek() from two threads, the interleaving is not guaranteed and the
  data can be scrambled.

  Fix: use O_APPEND and a single write() call, or add a lock around the
       seek-then-write pair.
```

---

# Creating Threads: clone()

Linux has no `pthread_create` system call. It has `clone()`.

`pthread_create()` in glibc is a wrapper that calls `clone()` with specific
flags.

```c
/* kernel/sched/fork.c — conceptually */

struct task_struct *kernel_clone(struct kernel_clone_args *args)
{
    /*
     * 1. Copy the task_struct (the thread's PCB)
     * 2. Duplicate or share the mm_struct, depending on CLONE_VM
     * 3. Duplicate or share the files_struct, depending on CLONE_FILES
     * 4. Duplicate or share the fs_struct, depending on CLONE_FS
     * 5. Duplicate or share the sighand_struct, depending on CLONE_SIGHAND
     * 6. Create a fresh kernel stack for the new thread
     * 7. Put the child on a runqueue
     */
}
```

## The clone() flag matrix

| Flag | Set | Effect |
|---|---|---|
| `CLONE_VM` | share the address space | threads share heap and globals |
| `CLONE_FILES` | share the fd table | threads see the same fds |
| `CLONE_FS` | share cwd and root | threads see the same working dir |
| `CLONE_SIGHAND` | share signal handlers | same dispositions |
| `CLONE_THREAD` | same thread group | `getpid()` returns the same value |
| `CLONE_SETTLS` | give the child a TLS block | makes it a real thread |
| `CLONE_PARENT_SETTID` | write the child's TID to the parent's memory | so the parent can join it |
| `CLONE_CHILD_CLEARTID` | clear the child's TID on exit | so the parent can detect exit |
| `CLONE_SYSVSEM` | share the System V undo list | |

| Flag | Clear | Effect |
|---|---|---|
| `CLONE_VM` absent | new address space | a forked **process** |
| `CLONE_THREAD` absent | different thread group | a **process** |

```text
  clone() with all CLONE_VM etc. set       → a THREAD
  clone() with none of them set            → a PROCESS (this is fork())
  vfork() / clone(CLONE_VM|CLONE_VFORK)    → shares the address space,
                                               parent BLOCKS until exec/exit
```

## fork() is clone() in disguise

```c
/* glibc's fork(), simplified */
#include <sched.h>
#include <sys/syscall.h>
#include <unistd.h>

pid_t fork(void)
{
    /* No CLONE_VM, no CLONE_FILES, no CLONE_THREAD: a brand new process. */
    return syscall(SYS_clone, SIGCHLD, 0, 0, 0, 0);
}
```

## Direct clone() from user space

```c
#define _GNU_SOURCE
#include <sched.h>
#include <sys/syscall.h>
#include <sys/wait.h>
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>

static int shared_counter = 0;

static int thread_fn(void *arg)
{
    (void)arg;
    for (int i = 0; i < 1000; i++)
        shared_counter++;         /* racing: this is a bug, deliberately */
    return 0;
}

int main(void)
{
    const size_t stack_size = 1 << 20;      /* 1 MiB */
    void *stack = malloc(stack_size);
    if (!stack) { perror("malloc"); return 1; }

    pid_t tid = syscall(SYS_clone,
                        CLONE_VM | CLONE_FS | CLONE_FILES | CLONE_SIGHAND |
                        CLONE_THREAD | CLONE_SETTLS | CLONE_CHILD_CLEARTID,
                        stack + stack_size,      /* top of the stack (grows down) */
                        thread_fn, NULL,          /* fn and arg */
                        0,                        /* tls */
                        0);                       /* ctid */

    if (tid < 0) { perror("clone"); return 1; }

    printf("spawned thread with tid %d (my pid is %d)\n", tid, getpid());

    int dummy;
    wait4(tid, &dummy, __WALL, NULL);        /* __WALL: it is a thread */
    printf("thread finished, counter = %d (expected 1000)\n", shared_counter);
    return 0;
}
```

```bash
gcc -Wall -Wextra -o rawclone rawclone.c
./rawclone
./rawclone
./rawclone
```

```text
spawned thread with tid 4822 (my pid is 4821)
thread finished, counter = 1000 (expected 1000)
thread finished, counter = 731 (expected 1000)
thread finished, counter = 588 (expected 1000)
```

> [!NOTE]
> Writing portable code with raw `clone()` is genuinely hard: TLS setup,
> `set_tid_address`, robust mutexes, and the child TID all have to be handled
> by hand. This example is for understanding the mechanism. Production code
> uses pthreads.

---

# Creating Threads with pthreads

## The complete lifecycle

```c
#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

/* Data shared by both threads. */
static int              shared_total;
static pthread_mutex_t  total_lock = PTHREAD_MUTEX_INITIALIZER;
static pthread_barrier_t start_line;

static void *worker(void *arg)
{
    long id = (long)arg;

    pthread_barrier_wait(&start_line);      /* make both start together */

    for (int i = 0; i < 100000; i++) {
        pthread_mutex_lock(&total_lock);
        shared_total++;
        pthread_mutex_unlock(&total_lock);
    }

    printf("thread %ld finished\n", id);
    return (void *)(id * 100);
}

int main(void)
{
    pthread_t t1, t2;

    pthread_barrier_init(&start_line, NULL, 2);   /* wait for 2 threads */

    pthread_create(&t1, NULL, worker, (void *)1L);
    pthread_create(&t2, NULL, worker, (void *)2L);

    void *r1, *r2;
    pthread_join(t1, &r1);
    pthread_join(t2, &r2);

    printf("total = %d (expected 200000)\n", shared_total);
    printf("returns: %ld and %ld\n", (long)r1, (long)r2);

    pthread_barrier_destroy(&start_line);
    return 0;
}
```

```bash
gcc -Wall -Wextra -pthread -o threaddemo threaddemo.c
./threaddemo
```

```text
thread 1 finished
thread 2 finished
total = 200000 (expected 200000)
returns: 100 and 200
```

## pthread_create

```c
int pthread_create(pthread_t *thread,
                   const pthread_attr_t *attr,
                   void *(*start_routine)(void *),
                   void *arg);
```

| Parameter | Meaning |
|---|---|
| `thread` | out: the thread ID |
| `attr` | stack size, scheduling policy, detached, affinity. `NULL` = defaults |
| `start_routine` | the entry point, returning `void *` |
| `arg` | a single `void *` passed to it |

```c
/* Passing more than one argument: pass a pointer to a struct. */
struct args {
    int          id;
    const char  *name;
    double       weight;
};

static void *scorer(void *p)
{
    struct args *a = p;
    printf("scoring %s (id %d, weight %.2f)\n", a->name, a->id, a->weight);
    return a;
}
```

## pthread_join

```c
int pthread_join(pthread_t thread, void **retval);
```

Blocks until the thread terminates, and optionally collects its return value.

| Situation | Correct call |
|---|---|
| You need the return value | `pthread_join(t, &ret)` |
| You only want to wait | `pthread_join(t, NULL)` |
| The thread is detached | **never join it** — that is `ESRCH` or undefined |
| The thread was never created | joining is a bug |

> [!IMPORTANT]
> Failing to `pthread_join()` a joinable thread is a resource leak. On Linux,
> its stack stays mapped and its `task_struct` stays allocated until it is
> joined or the whole process exits. A server that creates a thread per
> request and never joins leaks everything.

## Detached threads

```c
pthread_attr_t attr;
pthread_attr_init(&attr);
pthread_attr_setdetachstate(&attr, PTHREAD_CREATE_DETACHED);

pthread_t t;
pthread_create(&t, &attr, worker, arg);
pthread_detach(t);            /* never joined; resources freed on exit */
```

```text
  Joinable    → the creator must eventually pthread_join().
  Detached    → resources are reclaimed automatically at exit.
                The return value is discarded.
                pthread_join on it is an error.
```

## Setting thread attributes

```c
#include <pthread.h>
#include <sched.h>
#include <stdio.h>
#include <unistd.h>

static int shared_counter;

static void *worker(void *arg)
{
    for (int i = 0; i < 1000000; i++)
        shared_counter++;         /* deliberately racy */
    return NULL;
}

int main(void)
{
    pthread_attr_t attr;
    pthread_attr_init(&attr);

    /* A smaller stack: 256 KiB instead of the 8 MiB default. */
    pthread_attr_setstacksize(&attr, 256 * 1024);

    pthread_t t[4];
    for (int i = 0; i < 4; i++)
        if (pthread_create(&t[i], &attr, worker, NULL) != 0)
            perror("pthread_create");

    for (int i = 0; i < 4; i++)
        pthread_join(t[i], NULL);

    printf("counter = %d, expected 4000000\n", shared_counter);

    /* Pin this thread to CPU 0 */
    cpu_set_t set;
    CPU_ZERO(&set);
    CPU_SET(0, &set);
    if (pthread_setaffinity_np(pthread_self(), sizeof set, &set) != 0)
        perror("pthread_setaffinity_np");

    pthread_attr_destroy(&attr);
    return 0;
}
```

```bash
gcc -Wall -Wextra -pthread -D_GNU_SOURCE -o attrs attrs.c
./attrs
```

> [!NOTE]
> The default thread stack on Linux is **8 MiB**, the same as the main
> thread. 4,000 threads therefore reserve 32 GiB of address space. It is
> usually untouched (virtual, not resident), but `ulimit -s` and
> `vm.max_map_count` will object long before the RAM is used.

```bash
ulimit -s                       # 8192 (KiB)
cat /proc/sys/vm/max_map_count  # 65530 mappings
```

---

# Thread Identity: getpid() vs gettid()

This confuses everyone exactly once.

```c
#include <pthread.h>
#include <stdio.h>
#include <sys/syscall.h>
#include <unistd.h>

static void *worker(void *arg)
{
    (void)arg;
    printf("  thread: getpid=%d  gettid=%d  pthread_self=%lu\n",
           (int)getpid(),                       /* the process  */
           (int)syscall(SYS_gettid),            /* the thread   */
           (unsigned long)pthread_self());
    return NULL;
}

int main(void)
{
    pthread_t t;

    printf("main:   getpid=%d  gettid=%d  pthread_self=%lu\n",
           (int)getpid(), (int)syscall(SYS_gettid),
           (unsigned long)pthread_self());

    pthread_create(&t, NULL, worker, NULL);
    pthread_join(t, NULL);
    return 0;
}
```

```bash
gcc -Wall -Wextra -pthread -D_GNU_SOURCE -o ids ids.c
./ids
```

```text
main:   getpid=4821  gettid=4821  pthread_self=140234836314624
  thread: getpid=4821  gettid=4822  pthread_self=140234836314624
```

```text
  getpid()  = 4821 for BOTH   → they share a thread group
  gettid()  = 4821, 4822      → each thread is a distinct task
  pthread_self() is the same for both → it is a per-process value in glibc,
                                 NOT a unique thread identifier
```

> [!IMPORTANT]
> `pthread_self()` is **not** a unique thread ID. It returns the same value in
> every thread of the process. If you need a unique thread identifier, use
> `gettid()` via `syscall(SYS_gettid)`, or `pthread_getname_np` plus the tid.

## Signal delivery: which thread?

```text
  kill(pid, SIGUSR1)    → the kernel picks ONE arbitrary thread
                          that has not blocked SIGUSR1

  pthread_kill(t, sig)  → a specific thread

  The sigaction disposition is SHARED (one table for the process)
  The signal MASK is PER-THREAD
  The ALTERNATE signal stack is PER-THREAD
```

---

# Thread-Local Storage

## `__thread` — the fast, static case

```c
#include <pthread.h>
#include <stdio.h>

static __thread int    per_thread_counter;
static __thread double per_thread_buffer[64];
static __thread struct { int a, b; } record;

static void *worker(void *arg)
{
    (void)arg;
    per_thread_counter = 42;         /* only this thread sees this */
    printf("  thread: counter=%d buffer[0]=%.1f record={%d,%d}\n",
           per_thread_counter, per_thread_buffer[0], record.a, record.b);
    return NULL;
}

int main(void)
{
    per_thread_counter = 7;
    per_thread_buffer[0] = 1.5;
    record.a = 1; record.b = 2;

    printf("main:   counter=%d buffer[0]=%.1f record={%d,%d}\n",
           per_thread_counter, per_thread_buffer[0], record.a, record.b);

    pthread_t t;
    pthread_create(&t, NULL, worker, NULL);
    pthread_join(t, NULL);

    printf("main:   counter=%d (unchanged)\n", per_thread_counter);
    return 0;
}
```

```bash
gcc -Wall -Wextra -pthread -o tls tls.c
./tls
```

```text
main:   counter=7 buffer[0]=1.5 record={1,2}
  thread: counter=42 buffer[0]=0.0 record={0,0}
main:   counter=7 (unchanged)
```

```text
  On x86-64, __thread variables are addressed via the FS segment register.
  The offset is in %fs:base or a fixed offset. One MOV, no function call.
  Cost: essentially zero.
```

## `pthread_key_t` — the dynamic case

```c
#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>

static pthread_key_t key;

static void destructor(void *value)
{
    printf("  destructor called for %p\n", value);
    free(value);
}

static void *worker(void *arg)
{
    (void)arg;
    void *v = malloc(32);
    pthread_setspecific(key, v);          /* store in this thread only */
    printf("  thread: specific=%p\n", pthread_getspecific(key));
    return NULL;                          /* destructor runs at exit */
}

int main(void)
{
    pthread_key_create(&key, destructor);

    pthread_t t;
    pthread_create(&t, NULL, worker, NULL);
    pthread_join(t, NULL);

    printf("main:   specific=%p (null: never set here)\n",
           pthread_getspecific(key));
    pthread_key_delete(key);
    return 0;
}
```

```bash
gcc -Wall -Wextra -pthread -o tlsk tlsk.c
./tlsk
```

## Why errno is per-thread

```c
#include <errno.h>
#include <pthread.h>
#include <stdio.h>
#include <string.h>
#include <unistd.h>

static void *worker(void *arg)
{
    int fd = *(int *)arg;
    FILE *f = fdopen(fd, "r");
    (void)f;
    usleep(1000);
    printf("  thread: errno=%d (%s)\n", errno, strerror(errno));
    return NULL;
}

int main(void)
{
    errno = ENOENT;                       /* set it in main */
    printf("main:   errno=%d (%s)\n", errno, strerror(errno));

    int fds[2];
    pipe(fds);
    pthread_t t;
    pthread_create(&t, NULL, worker, &fds[1]);
    pthread_join(t, NULL);
    return 0;
}
```

```text
main:   errno=2 (No such file or directory)
  thread: errno=0 (Success)
```

If `errno` were a plain global, the two threads would corrupt each other.

---

# The Thread Lifecycle

```text
  ┌───────────┐
  │ Created   │  pthread_create() returns a pthread_t
  └─────┬─────┘
        │
        ▼
  ┌───────────┐    returns, or pthread_exit(), or return from start_routine
  │ Runnable  │◄──────────────────────────┐
  └─────┬─────┘                            │
        │  scheduled                        │ unblocked, or a higher-priority
        ▼                                   │ thread is waiting
  ┌───────────┐                            │
  │  Running  │────────────────────────────┘
  └─────┬─────┘
        │
        │  mutex, condvar, join, sleep, I/O
        ▼
  ┌───────────┐
  │ Blocked   │
  └─────┬─────┘
        │
        │  lock released, I/O complete
        └──────────────► Runnable
```

## States as seen in `ps`

```bash
ps -eLo pid,tid,psr,stat,comm | head -20
```

```text
  PID   TID PSR STAT COMMAND
 4821  4821   0 Ss   bash
 5100  5100   1 S    worker        ← R or S depending on the moment
 5100  5101   2 S    worker
 5100  5102   3 R    worker
 5100  5103   0 Sl   worker
 5100  5104   1 S    worker
```

```bash
# Threads of one process, with their state and CPU
ps -T -p 5100 -o tid,psr,stat,pcpu,comm

# Or directly from proc
for t in /proc/5100/task/*; do
    printf "%s %s\n" "$(basename $t)" "$(awk '/^State/{print $2}' $t/status)"
done

# Live view, per-thread
top -H -p 5100
htop -t
```

---

# Threads vs Processes: Choosing

## Use processes when

```text
  ✔ You need ISOLATION — a plugin must not be able to crash the host
  ✔ You need to run UNTRUSTED code
  ✔ The work is dominated by kernel time (I/O, syscalls)
  ✔ You want a hard resource limit per unit (ulimit, cgroup)
  ✔ Fault isolation matters more than memory efficiency
```

## Use threads when

```text
  ✔ You need to share a lot of DATA efficiently
  ✔ The work is CPU-bound in user space
  ✔ You need lower latency (a thread switch is cheaper)
  ✔ You need to keep shared caches warm
  ✔ The communication cost dominates the computation
```

## The memory argument

```text
  1,000 processes, 200 MB of shared read-only data each:

      naive:   1,000 × 200 MB = 200 GB of address space
      CoW:     shared physically until written
               → maybe 200 MB + a few private pages
      so:      process is fine, IF the data is genuinely read-only

  1,000 threads, 200 MB of shared data:

      shared by definition
               → exactly 200 MB, plus 1,000 stacks
               → 1,000 × 8 MiB = 8 GB of stack address space
                 (virtual; reduce with pthread_attr_setstacksize)
```

```bash
# Compare the memory footprint
/usr/bin/time -v ./process_server   2>&1 | grep -E 'Maximum resident'
/usr/bin/time -v ./thread_server    2>&1 | grep -E 'Maximum resident'
```

---

# Common Misconceptions

### ❌ "Threads are lighter than processes, so always use threads."

Incorrect.

Threads are cheaper to create and switch, but they share an address space, so
one buffer overrun corrupts everything. Threads trade isolation for speed.
Choose based on the trust boundary, not the benchmark.

---

### ❌ "`pthread_self()` uniquely identifies a thread."

Incorrect.

In glibc it returns the same value in every thread of the process. Use
`gettid()` for a unique thread identifier.

---

### ❌ "Threads get their own copy of globals."

Incorrect.

Globals are shared. Only `__thread` variables and `pthread_getspecific` values
are per-thread.

---

### ❌ "A thread has its own heap."

Incorrect.

The heap belongs to the process. `malloc` is thread-safe (it takes an internal
lock), but both threads allocate from the same heap.

---

### ❌ "Thread stacks are small because threads are light."

Incorrect.

The default is 8 MiB per thread, the same as the main stack. It is virtual
address space, so it is usually not resident — but it counts against
`vm.max_map_count` and `ulimit`.

---

### ❌ "You must call `pthread_join()` or the program crashes."

Incorrect.

You must call it or the thread's resources leak. Skipping it does not crash
the program; it wastes a stack and a `task_struct` until the process exits.

---

# Interview Questions

### Basic

- What is a thread? How is it different from a process?
- What is shared and what is private between threads?
- What are the three requirements a thread must satisfy to be considered a
  thread of a process?
- What is `pthread_join()` and why is it needed?

### Intermediate

- Explain `clone()` and its flags. Which flags make a clone a thread?
- What is thread-local storage, and how is `errno` implemented?
- How are signals delivered to a process with multiple threads?
- How do you inspect threads on a running Linux process?
- How would you set a thread's stack size and CPU affinity?

### Advanced

- Why is a thread context switch cheaper than a process context switch?
- What is the danger of sharing a file descriptor between threads, given that `f_pos` is shared?
- When is `fork()` in a multithreaded program dangerous, and what happens?
- How do you detect and debug a data race in a C program?
- What is the cost of 10,000 threads on Linux, and how would you reduce it?

---

# University Exam Notes

### Definitions

- **Thread:** A basic unit of CPU execution within a process, having its own
  program counter, registers and stack, while sharing the process's address
  space and resources.
- **Thread Control Block (TCB):** The kernel structure holding a thread's
  execution context — equivalent to the PCB for a thread.
- **Single-threaded / Multi-threaded:** A process with one / more than one
  thread of control.
- **Thread-local storage:** Storage that has a distinct instance for each
  thread, accessed via `__thread` or `pthread_getspecific`.
- **Detached thread:** A thread whose resources are reclaimed automatically
  when it exits, and which cannot be joined.

### Frequently Asked Questions

- What is a thread? Differentiate between a process and a thread.
- What is the difference between a process and a thread in terms of memory?
- Explain user-level and kernel-level threads.
- What is `clone()`? Explain the flags used to create threads.
- What is thread-local storage? Why is `errno` per-thread?
- Explain the states of a thread.
- When should you use processes instead of threads?

---

# Key Takeaways

- A thread is an independent execution stream **inside** one process, sharing
  the address space, heap and file descriptors.
- Private per thread: registers, stack, TLS, `errno`, signal mask.
- Linux creates threads with `clone()` + `CLONE_VM`; pthreads is a wrapper.
  `fork()` is `clone()` with none of those flags.
- `getpid()` is shared; `gettid()` is unique. **`pthread_self()` is not
  unique.**
- `__thread` is a single instruction (FS-relative). `pthread_key_t` is
  dynamic and supports destructors.
- The default stack is **8 MiB per thread**; set it explicitly with
  `pthread_attr_setstacksize`.
- Always `pthread_join()` joinable threads, or `pthread_detach()` them.
- Signal dispositions are shared; signal masks are per-thread.
- Choose processes for isolation, threads for shared data and low latency.

---

# References

- Operating System Concepts — Silberschatz
- Modern Operating Systems — Tanenbaum and Bos
- The Linux Programming Interface — Kerrisk
- Pthreads Programming — Nichols, Buttfield, Buckley
- `man 3 pthread_create`, `man 3 pthread_join`, `man 7 pthread`
- `man 2 clone`, `man 2 gettid`, `man 2 futex`
- `ps(1)`, `top(1)`, `ulimit(1)`
