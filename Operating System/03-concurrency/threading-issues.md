# Threading Issues

> [!NOTE]
> **Module:** Module III – Concurrency
>
> **Difficulty:** ⭐⭐⭐⭐☆
>
> **Prerequisites:**
> - Threads
> - Critical Section
> - Semaphores
>
> **Scope:** Linux and C only. No Windows or Java.

---

# Learning Objective

After reading this note, you should be able to:

- List the practical problems that arise when writing threaded C programs.
- Explain what happens on `fork()` in a multithreaded program, and why it is a problem.
- Explain signal handling problems in threaded programs.
- Explain thread-safety of library functions and the `errno` hazard.
- Explain deadlock, livelock and starvation, and how to prevent each.
- Explain performance problems: false sharing, oversubscription, context switch cost.
- Use ThreadSanitizer and other tools to find real defects.
- Apply practical rules that prevent most threading bugs.

---

# Why Do We Need a Note About This?

## The Problem

Threading bugs are not like ordinary bugs.

| | Ordinary bug | Threading bug |
|---|---|---|
| Reproducibility | usually deterministic | often intermittent |
| When it appears | every run | 1 run in 10,000 |
| Diagnosis | usually obvious | requires a different class of tool |
| Test coverage | measurable | effectively unmeasurable |
| Worst case | wrong output | silent corruption, hang, or security hole |

The practical consequence: **most threading bugs found in production were
written by people who were confident the code was correct.**

This note is the catalogue of the ways that confidence is misplaced.

---

# Real World Analogy

A shared office with 8 people and one photocopier.

There is no manager, no lock, and no sign saying "one at a time". Each person
judges for themselves when the machine is free.

- Person A starts a 20-page copy, turns away to get coffee.
- Persons B through H each also decide the machine is "free" and start.
- The machine's job buffer overflows. Half the copies are wrong.

Nothing malicious happened. Everyone behaved reasonably. The design was wrong.

---

# Intuition

```text
  Every threaded program has these four ingredients:

      1. SHARED STATE        the data more than one thread touches
      2. INTERLEAVING        the order of operations is not fixed
      3. NON-ATOMIC OPS      read-modify-write, check-then-act
      4. HIDDEN DEPENDENCIES what you did not write, but still call

  A defect needs all four:

      shared state  AND  non-atomic operation
                    AND  an interleaving that exposes it
                    AND  a hidden dependency that makes the fix wrong

  Remove any one, and the bug goes away. That is the entire design space.
```

---

# Problem 1 — fork() in a Multithreaded Program

## What happens

```text
  Process P has 4 threads: T1 (calling fork), T2, T3, T4.

  fork() does ONE thing: it clones the calling thread's task_struct.

  ┌──────────────────────────────────────────────────────────┐
  │  Child process contains:                                 │
  │    ✔ T1 (the caller), with a COPY of T1's registers and   │
  │      kernel stack, mid-`fork()`                          │
  │    ✘ T2, T3, T4 — GONE                                    │
  │                                                           │
  │    ✘ No mutexes held by T2/T3/T4 are held.                │
  │      Any such mutex is now PERMANENTLY UNLOCKED.           │
  │                                                           │
  │    ✘ No other thread's signal handlers, TLS, or atexit     │
  │      handlers exist.                                      │
  └──────────────────────────────────────────────────────────┘
```

> [!IMPORTANT]
> In the child, **only the thread that called `fork()` survives.** The child's
> memory holds no evidence that T2, T3 or T4 ever existed. Any lock they held
> is now free, and any invariant they were maintaining is now broken.

## The deadly case: a malloc lock held by a dead thread

```c
#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/wait.h>

static pthread_mutex_t log_lock = PTHREAD_MUTEX_INITIALIZER;
static FILE *logfile;

static void *logger_thread(void *arg)
{
    (void)arg;
    for (int i = 0; i < 1000; i++) {
        pthread_mutex_lock(&log_lock);
        fprintf(logfile, "line %d\n", i);
        pthread_mutex_unlock(&log_lock);
    }
    return NULL;
}

static void other_allocating_thread(void)
{
    /* malloc internally takes a lock. If this thread dies holding it,
       the lock is still held in the child's view — and the child's
       surviving thread will deadlock on its next malloc. */
    for (int i = 0; i < 100000; i++) {
        void *p = malloc(64);
        free(p);
    }
}

int main(void)
{
    logfile = fopen("/tmp/threadlog.txt", "w");

    pthread_t t1, t2;
    pthread_create(&t1, NULL, logger_thread, NULL);
    pthread_create(&t2, NULL, (void *(*)(void *))other_allocating_thread, NULL);

    usleep(200000);          /* let them get going */

    pid_t pid = fork();
    if (pid == 0) {
        /* CHILD: only the forking thread exists here. */
        puts("child: calling malloc after fork with 2 threads in the parent");
        fflush(stdout);
        void *p = malloc(64);
        puts("child: malloc returned OK");
        fflush(stdout);
        free(p);
        _exit(0);
    }

    int status;
    waitpid(pid, &status, 0);
    if (WIFSIGNALED(status))
        printf("child died from signal %d\n", WTERMSIG(status));
    else
        printf("child exited with %d\n", WEXITSTATUS(status));

    return 0;
}
```

```bash
gcc -Wall -Wextra -pthread -o forkrace forkrace.c
for i in $(seq 1 20); do ./forkrace; done
```

```text
child: calling malloc after fork with 2 threads in the parent
child: malloc returned OK
child exited with 0
...
child died from signal 6      ← SIGABRT: "double free or corruption"
child died from signal 6
```

The child's `malloc` walks a heap whose allocator state was corrupted by the
disappearance of a thread that was mid-`malloc`.

## The rules

```c
/* RULE 1: in a forked child of a multithreaded program, call only
           async-signal-safe functions until exec(). */

/* Safe:   _exit(), execve(), write(), dup2(), close(), fork(), waitpid() */
/* Unsafe: malloc(), printf(), pthread_mutex_lock(), anything that locks */

/* RULE 2: if you must do real work, do it before the fork. */

/* RULE 3: the recommended pattern — fork then immediately exec. */
pid_t pid = fork();
if (pid == 0) {
    /* only async-signal-safe calls here, then exec */
    char *const argv[] = { "/bin/ls", NULL };
    char *const envp[] = { "PATH=/bin", NULL };
    execve("/bin/ls", argv, envp);
    _exit(127);
}
```

```c
/* RULE 4: for a genuine "fork a child to do work" server, serialise first. */
pthread_mutex_t fork_lock = PTHREAD_MUTEX_INITIALIZER;

pid_t spawn(void (*fn)(void))
{
    pthread_mutex_lock(&fork_lock);
    /* fork() runs with the whole process quiesced:
       every other thread is stopped at a known, lock-free point */
    pid_t pid = fork();
    if (pid == 0) {
        fn();
        _exit(0);
    }
    pthread_mutex_unlock(&fork_lock);
    return pid;
}
```

```bash
# Observe the state directly
strace -f -e trace=clone,clone3,execve ./forkrace
```

---

# Problem 2 — Signal Handling in Threads

## The rules the kernel imposes

```text
  SHARED across all threads (one table for the process):
      the signal DISPOSITION      sigaction(SIGINT, ...)
      the signal MASK for the whole process is NOT shared —
                                        it IS per-thread

  PER-THREAD:
      the signal MASK             sigprocmask(), pthread_sigmask()
      the pending-signal QUEUE    a signal is delivered to one thread
      the alternate signal STACK  sigaltstack()
```

## The three classic mistakes

### Mistake 1 — assuming the handler runs on the thread that caused the event

```c
/* Thread A writes to a pipe, which the SIGPIPE handler reads. */
static void handler(int sig) { (void)sig; got_signal = 1; }

void *reader(void *arg) { (void)arg; for (;;) { if (got_signal) break; } return NULL; }
```

The signal is delivered to an **arbitrary thread that has not blocked it**, not
to the one that caused the event.

### Mistake 2 — block signals in some threads and not others

```c
/* Fix for the "wrong thread" problem: block in workers, handle in main. */
#include <pthread.h>
#include <signal.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

static volatile sig_atomic_t got_signal = 0;

static void handler(int sig) { (void)sig; got_signal = 1; }

static void *worker(void *arg)
{
    (void)arg;
    sigset_t set;
    sigemptyset(&set);
    sigaddset(&set, SIGUSR1);
    pthread_sigmask(SIG_BLOCK, &set, NULL);    /* workers block it */
    /* ... worker body ... */
    return NULL;
}

int main(void)
{
    /* Only the main thread has it unblocked, so only it can be chosen. */
    struct sigaction sa = { .sa_handler = handler };
    sigemptyset(&sa.sa_mask);
    sigaction(SIGUSR1, &sa, NULL);

    pthread_t t[4];
    for (int i = 0; i < 4; i++) pthread_create(&t[i], NULL, worker, NULL);

    raise(SIGUSR1);                       /* delivered to main, not a worker */

    for (int i = 0; i < 4; i++) pthread_join(t[i], NULL);
    printf("handled in main: got_signal = %d\n", (int)got_signal);
    return 0;
}
```

### Mistake 3 — using a mutex inside a handler

```c
static pthread_mutex_t db_lock;
static void handler(int sig) { pthread_mutex_lock(&db_lock); /* WRONG */ }
```

The signal may have arrived while the thread already held `db_lock` in normal
code. The handler then self-deadlocks.

```bash
# The safe C alternative: a self-pipe, or signalfd()
```

```c
/* The best answer on Linux: signalfd(). No handlers at all. */
#define _GNU_SOURCE
#include <signal.h>
#include <sys/signalfd.h>
#include <unistd.h>
#include <poll.h>
#include <stdio.h>

int main(void)
{
    sigset_t mask;
    sigemptyset(&mask);
    sigaddset(&mask, SIGUSR1);

    /* Block it in EVERY thread before creating any. */
    if (sigprocmask(SIG_BLOCK, &mask, NULL) == -1)
        perror("sigprocmask");

    int sfd = signalfd(-1, &mask, SFD_CLOEXEC);
    if (sfd < 0) { perror("signalfd"); return 1; }

    struct pollfd pfd = { .fd = sfd, .events = POLLIN };

    for (;;) {
        if (poll(&pfd, 1, -1) < 0) continue;
        struct signalfd_siginfo si;
        if (read(sfd, &si, sizeof si) == sizeof si)
            printf("received signal %u from pid %d\n", si.ssi_signo, si.ssi_pid);
    }
}
```

```bash
gcc -Wall -Wextra -o sfd sfd.c
./sfd &
kill -USR1 %1
```

> [!TIP]
> `signalfd()` turns signals into an ordinary readable file descriptor. It
> integrates with `poll()` and `epoll()` and has no async-signal-safety
> constraints at all. It is the modern Linux answer, and it is specific to
> Linux.

---

# Problem 3 — Thread Safety of Library Functions

## Which libc functions are safe?

```text
  THREAD-SAFE (glibc guarantees this):
      malloc, free, calloc, realloc
      printf, fprintf, snprintf
      open, close, read, write
      pthread_* (all)
      strerror_r (the GNU version)
      qsort, bsearch

  NOT THREAD-SAFE (POSIX explicitly reserves the name *_r):
      gethostbyname   →  getaddrinfo_a, getnameinfo
      asctime, ctime, localtime, gmtime
        (they return a pointer to a static buffer)
      strtok          →  strtok_r
      readdir         →  readdir_r (deprecated; prefer a per-thread DIR*)
      crypt           →  crypt_r
      getpwnam        →  getpwnam_r
      ttyname_r vs ttyname

  SAFE BUT A SURPRISE:
      asctime_r and ctime_r may still share a buffer on glibc
```

## The errno hazard

```c
#include <errno.h>
#include <stdio.h>

int fd = open("f", O_RDONLY);
if (fd < 0) {
    int e = errno;            /* save FIRST — nothing else may run first */
    log_error(e);
    perror("open");           /* this may itself modify errno */
    report(e);                /* use the saved copy */
}
```

```c
/* WRONG: errno read twice, and a function called in between. */
if (fd < 0)
    fprintf(stderr, "open failed: %s / %d\n", strerror(errno), errno);
```

```bash
# errno is per-thread and is also exposed as a variable
man 3 errno
```

---

# Problem 4 — Deadlock

## Definition

Deadlock is a state in which every thread is blocked, waiting for a resource
that only another blocked thread can release.

```text
  Thread 1                          Thread 2
  ────────                          ────────
  lock(A)          ──► held
                                    lock(B)    ──► held
  lock(B)  ──► BLOCKED (A has B? no, 2 has it)
                                    lock(A)  ──► BLOCKED (1 has it)

  Neither can proceed. Neither will ever proceed.
  No output. No crash. No log. Just a hang.
```

## The four Coffman conditions — all must hold

| Condition | Meaning | How to break it |
|---|---|---|
| **Mutual exclusion** | the resource is non-shareable | use a sharable resource |
| **Hold and wait** | holding one resource while waiting for another | acquire all at once |
| **No preemption** | a held resource cannot be taken away | make it stealable, or always release |
| **Circular wait** | a cycle in the wait-for graph | impose a global order |

## Classic deadlock in C

```c
#include <pthread.h>
#include <stdio.h>
#include <unistd.h>

static pthread_mutex_t lock_a = PTHREAD_MUTEX_INITIALIZER;
static pthread_mutex_t lock_b = PTHREAD_MUTEX_INITIALIZER;

static void transfer_a_to_b(void) { pthread_mutex_lock(&lock_a); usleep(1000); pthread_mutex_lock(&lock_b); /* ... */ pthread_mutex_unlock(&lock_b); pthread_mutex_unlock(&lock_a); }

static void transfer_b_to_a(void) { pthread_mutex_lock(&lock_b); usleep(1000); pthread_mutex_lock(&lock_a); /* ... */ pthread_mutex_unlock(&lock_a); pthread_mutex_unlock(&lock_b); }

static void *t1(void *arg) { (void)arg; transfer_a_to_b(); return NULL; }
static void *t2(void *arg) { (void)arg; transfer_b_to_a(); return NULL; }

int main(void)
{
    pthread_t x, y;
    pthread_create(&x, NULL, t1, NULL);
    pthread_create(&y, NULL, t2, NULL);
    pthread_join(x, NULL);
    pthread_join(y, NULL);
    puts("never printed");
    return 0;
}
```

```bash
gcc -Wall -Wextra -pthread -o deadlock deadlock.c
./deadlock &
sleep 2
ps -T -p $! -o pid,tid,stat,wchan:30,comm
```

```text
  PID   TID STAT WCHAN                          COMM
 5001  5001 S    futex_wait_queue_me             deadlock
 5001  5002 S    futex_wait_queue_me             deadlock
```

```bash
# Where is it stuck?
gdb -p 5001 -batch -ex "thread apply all bt"
cat /proc/5001/task/5001/stack
```

## The three fixes

```text
  FIX 1: ORDER THE LOCKS GLOBALLY  ← best, simplest, always works
  FIX 2: TRYLOCK AND BACK OFF
  FIX 3: ACQUIRE ALL AT ONCE
```

```c
/* FIX 1: a strict global order. Lock A always before B, everywhere. */
static void transfer_a_to_b(void) {
    pthread_mutex_lock(&lock_a);
    pthread_mutex_lock(&lock_b);      /* A then B */
    /* ... */
    pthread_mutex_unlock(&lock_b);
    pthread_mutex_unlock(&lock_a);
}

static void transfer_b_to_a(void) {
    pthread_mutex_lock(&lock_a);      /* ALSO A then B — same order! */
    pthread_mutex_lock(&lock_b);
    /* ... */
    pthread_mutex_unlock(&lock_b);
    pthread_mutex_unlock(&lock_a);
}
```

```c
/* FIX 2: trylock with a backoff, then retry the whole operation. */
static int transfer_ab(void)
{
    for (int attempt = 0; attempt < 100; attempt++) {
        if (pthread_mutex_trylock(&lock_a) != 0) {
            backoff(attempt);
            continue;
        }
        if (pthread_mutex_trylock(&lock_b) != 0) {
            pthread_mutex_unlock(&lock_a);
            backoff(attempt);
            continue;
        }
        /* critical section */
        do_transfer();
        pthread_mutex_unlock(&lock_b);
        pthread_mutex_unlock(&lock_a);
        return 0;
    }
    return -1;
}

static void backoff(int attempt)
{
    struct timespec ts = {
        .tv_sec  = 0,
        .tv_nsec = (rand() % 1000) * 1000L * (1 + attempt % 8)
    };
    nanosleep(&ts, NULL);       /* random: breaks the lockstep livelock */
}
```

```c
/* FIX 3: a single lock, or an ordered structure. */
static pthread_mutex_t vault = PTHREAD_MUTEX_INITIALIZER;

static void transfer_a_to_b(void) { pthread_mutex_lock(&vault); /* ... */ pthread_mutex_unlock(&vault); }
static void transfer_b_to_a(void) { pthread_mutex_lock(&vault); /* ... */ pthread_mutex_unlock(&vault); }
```

---

# Problem 5 — Livelock and Starvation

```text
  DEADLOCK   both threads wait forever, doing nothing
  LIVELOCK   both threads keep reacting to each other, doing nothing useful
  STARVATION one thread runs fine; another never gets a turn
```

## Livelock

```c
/*
 * Two threads politely yielding to each other forever.
 */
static pthread_mutex_t a = PTHREAD_MUTEX_INITIALIZER;
static pthread_mutex_t b = PTHREAD_MUTEX_INITIALIZER;

static void *polite(void *arg)
{
    pthread_mutex_t *me  = arg;
    pthread_mutex_t *you = (me == &a) ? &b : &a;

    for (int i = 0; i < 1000000; i++) {
        pthread_mutex_lock(me);
        if (pthread_mutex_trylock(you) != 0) {
            pthread_mutex_unlock(me);     /* "sorry, you go first" */
            sched_yield();
            continue;
        }
        do_work();
        pthread_mutex_unlock(you);
        pthread_mutex_unlock(me);
    }
    return NULL;
}
```

```text
  Both threads are running. Both are making progress in the sense that
  they are executing. Neither ever does the work.

  Fix: break the symmetry. Randomise the backoff, or prefer a fixed
  priority, or use a fair queue.
```

## Starvation

```c
/*
 * A stream of short critical sections can starve a long one.
 */
static void short_critical_section(void) {
    pthread_mutex_lock(&lock);
    tiny_work();
    pthread_mutex_unlock(&lock);
}

static void long_critical_section(void) {
    pthread_mutex_lock(&lock);
    for (int i = 0; i < 1000000; i++) work(i);   /* holds it a long time */
    pthread_mutex_unlock(&lock);
}
```

```text
  glibc's default mutex is NOT fair. A thread that releases and immediately
  re-acquires often wins the race again.

  Fixes:
    • pthread_mutexattr_settype(PTHREAD_MUTEX_ERRORCHECK) — detects bugs,
      still not fair
    • sem_t, which IS fair (FIFO waiter queue)
    • An explicit ticket lock
    • Split the long critical section into chunks, releasing between them
```

```bash
# Which threads are starving?
ps -T -p 5001 -o tid,pcpu,stat,comm
# A thread with 0.0% CPU and state S is a candidate
```

---

# Problem 6 — False Sharing

See [critical-section.md](../01-cpu-execution/critical-section.md) for the full
treatment. In summary:

```c
struct counters {
    long a;    /* written by thread 0 */
    long b;    /* written by thread 1 */
    long c;    /* written by thread 2 */
};

static struct counters c;      /* 24 bytes: ONE 64-byte cache line */
```

```text
  CPU 0 writes a → invalidates the line on CPU 1
  CPU 1 writes b → invalidates the line on CPU 0
  CPU 2 writes c → invalidates the line on both

  The threads share no logical state, but they share a cache line.
  Every write costs a coherence round trip: ~100 ns each.
```

```c
struct counters {
    long a;
    char pad0[64 - sizeof(long)];
    long b;
    char pad1[64 - sizeof(long)];
    long c;
    char pad2[64 - sizeof(long)];
} __attribute__((aligned(64)));
```

```bash
perf c2c record -g ./program
perf c2c report
# "Shared Data Cache Line" table shows which lines bounce between cores
```

---

# Problem 7 — Oversubscription and Context Switch Cost

```text
  8 cores. 64 runnable threads.

  The scheduler time-slices 64 threads across 8 cores.
  Every ~4 ms, a context switch. 64 threads × 250 switches/s = 16,000
  switches/s, plus the cache damage for each.

  Throughput can be LOWER than with 8 threads, because the extra switches
  cost more than the parallelism gains.
```

```c
#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

static double work(long seed)
{
    double x = seed + 1.0;
    for (long i = 0; i < 20000000L; i++)
        x = x * 1.0000001 + (i % 7);
    if (x < 0) printf("%f", x);
    return x;
}

static void *run(void *arg) { work((long)arg); return NULL; }

int main(void)
{
    long cores = sysconf(_SC_NPROCESSORS_ONLN);
    printf("%ld cores online\n", cores);

    long counts[] = { 1, 2, cores, cores * 2, cores * 4, cores * 8 };
    for (unsigned k = 0; k < sizeof counts / sizeof *counts; k++) {
        long n = counts[k];
        pthread_t *t = malloc(n * sizeof *t);

        struct timespec a, b;
        clock_gettime(CLOCK_MONOTONIC, &a);
        for (long i = 0; i < n; i++) pthread_create(&t[i], NULL, run, (void *)i);
        for (long i = 0; i < n; i++) pthread_join(t[i], NULL);
        clock_gettime(CLOCK_MONOTONIC, &b);

        double s = (b.tv_sec - a.tv_sec) + (b.tv_nsec - a.tv_nsec) / 1e9;
        printf("%4ld threads : %6.3f s   %s\n", n, s,
               n > cores ? "<-- oversubscribed" : "");
        free(t);
    }
    return 0;
}
```

```bash
gcc -O2 -pthread -D_GNU_SOURCE -o oversub oversub.c
./oversub
vmstat 1
```

```text
    8 cores online
    1 threads :  0.201 s
    2 threads :  0.202 s
    8 threads :  0.204 s
   16 threads :  0.611 s   <-- oversubscribed
   32 threads :  1.204 s   <-- oversubscribed
   64 threads :  2.410 s   <-- oversubscribed: 3× SLOWER than 8 threads
```

```bash
# Confirm the cause
vmstat 1 5
#   r = 64 (runnable),  cs = 40000+/s,  us = 60,  id = 35
```

**Rule: CPU-bound work gets exactly one thread per core.** More is strictly
worse.

---

# Problem 8 — Zombie Threads and Thread Leaks

```c
/* A server that leaks a thread per request. */
for (;;) {
    handle_one_request();          /* spawns a thread */
}
```

```text
  10,000 requests per second × 8 MiB per stack
  = 80 GiB of address space, and 10,000 task_structs, per second.

  Eventually: pthread_create fails with EAGAIN, and the server silently
  stops accepting work.
```

```bash
# Detect it
ls /proc/$(pidof myserver)/task | wc -l
ps -T -p $(pidof myserver) | wc -l
watch -n1 "ls /proc/$(pidof myserver)/task | wc -l"
```

```text
  Fix: a thread pool with a bounded queue, OR detached threads, OR —
  if the work is I/O — a single-threaded event loop with epoll.
```

---

# Problem 9 — Order-of-Initialisation Problems

```c
static pthread_mutex_t lock = PTHREAD_MUTEX_INITIALIZER;   /* fine */
static int counter = 0;                                    /* fine */

static pthread_mutex_t runtime_lock;   /* zero-initialised */
static void init(void) { pthread_mutex_init(&runtime_lock, NULL); }
```

```text
  A thread created before init() runs may find the mutex in a state
  it was never initialised into.

  Worse: a global CONSTRUCTOR that spawns a thread, and a global object
  that the thread uses. The order of initialisers is undefined, so the
  thread can observe a partially constructed object.

  Fix: one explicit init function, called from main(), before any thread.
```

```c
static pthread_once_t init_once = PTHREAD_ONCE_INIT;

static void do_init(void) { pthread_mutex_init(&runtime_lock, NULL); }

static void ensure_ready(void) { pthread_once(&init_once, do_init); }
```

---

# Problem 10 — Compiler and CPU Reordering

```c
static int data = 0;
static int ready = 0;

void producer(void) {
    data = 42;
    ready = 1;              /* looks like it happens after */
}

int consumer(void) {
    while (!ready) { }
    return data;            /* expects 42 */
}
```

```text
  On x86 this often works: the store buffer preserves store order.

  On ARM, RISC-V, or with -O2 on any target, it does NOT. The compiler
  is permitted to reorder two stores to independent locations, and the
  CPU is permitted to make them visible out of order.

  There is no data race here in the C11 sense for the stores to
  `data`/`ready` individually, but the PROGRAM is still wrong, and the
  C standard says the behaviour is undefined because of the loop.
```

```c
/* The correct fix: express the ordering explicitly. */
#include <stdatomic.h>

static _Atomic int data;
static _Atomic int ready;

void producer(void) {
    atomic_store_explicit(&data, 42, memory_order_relaxed);
    atomic_store_explicit(&ready, 1, memory_order_release);   /* publishes */
}

int consumer(void) {
    while (atomic_load_explicit(&ready, memory_order_acquire) == 0) { }
    return atomic_load_explicit(&data, memory_order_relaxed);  /* 42, guaranteed */
}
```

See [race-condition.md](../05-synchronization/race-condition.md) for the full
treatment of memory ordering.

---

# Finding Threading Bugs with Tools

## ThreadSanitizer — finds data races

```bash
gcc -fsanitize=thread -g -O1 -pthread -o prog prog.c
./prog
```

```text
WARNING: ThreadSanitizer: data race
  Write of size 8 at 0x7b0400000000 by thread T1:
    #0 increment counter+0x28 at prog.c:14
  Previous read of size 8 at 0x7b0400000000 by thread T2:
    #0 report+0x1c at prog.c:29
  Location is global 'counter' at prog.c:9

SUMMARY: ThreadSanitizer: data race prog.c:14 in increment
```

```bash
# TSAN options
TSAN_OPTIONS="halt_on_error=1 second_deadlock_stack=1"
TSAN_OPTIONS="history_size=7"      # more history
```

## Helgrind — find it without recompiling

```bash
valgrind --tool=helgrind ./prog
```

```text
==1234== Possible data race during write of size 8
==1234==    at 0x4005A1: main (prog.c:14)
==1234==  Address 0x1183c08 is 0 bytes inside a block of size 8 free'd
```

## AddressSanitizer — memory errors

```bash
gcc -fsanitize=address -g -O1 -pthread -o prog prog.c
ASAN_OPTIONS="detect_leaks=1" ./prog
```

Catches use-after-free, buffer overflows, and leaks — the bugs that
concurrency makes far more likely.

## UndefinedBehaviorSanitizer — the cheap, always-on one

```bash
gcc -fsanitize=undefined,integer-divide-by-zero,shift,bounds -g -O1
```

## Debugging practical issues

```bash
# Which thread is where?
gdb -p $(pidof prog)
(gdb) info threads
(gdb) thread apply all bt

# What is each thread blocked on?
for t in /proc/$(pidof prog)/task/*; do
    echo "tid $(basename $t): $(awk '/wchan/{print $2}' $t/stat)"
done

# Lock-order check (helgrind reports it)
/usr/bin/valgrind --tool=helgrind ./prog

# Syscall-level tracing, per thread
strace -f -tt -e trace=futex ./prog

# Count context switches
ps -T -p $(pidof prog) -o tid,nvcsw,nivcsw,stat
```

```text
  nm -D ./prog | grep pthread_mutex_lock    ← confirm the symbol exists
  gdb ./prog -batch -ex 'set pagination off' -ex 'info address pthread_mutex_lock'
```

---

# Common Misconceptions

### ❌ "`volatile` makes a variable safe to share between threads."

Incorrect, and this is the most damaging misconception in threaded C.

`volatile` only stops the **compiler** from caching the value in a register. It
says nothing about atomicity, nothing about the CPU's store buffer, and
nothing about reordering. Use `_Atomic` with a memory order, or a mutex.

---

### ❌ "Thread bugs only happen on multicore machines."

Incorrect.

A data race is a defect in the program. It manifests on a single core when a
timer interrupt arrives between the read and the write of a read-modify-write,
and it manifests more often on multicore because the window is far wider.

---

### ❌ "More threads always means better performance."

Incorrect for CPU-bound work. One thread per core is the maximum. Beyond that
you pay for switches and cache damage and get negative returns.

---

### ❌ "A mutex protects the code, not the data."

Incorrect, and the confusion causes real bugs.

A mutex protects a **data invariant**. A lock is only correct if every thread
that touches the data uses the same lock, over the same region. One unlocked
access anywhere voids the whole scheme.

---

### ❌ "If the tests pass, the threading is correct."

Incorrect.

A race that appears once in 10,000 runs passes a test suite every time. Thread
correctness is a reasoning problem, and it needs a model of the memory model,
not a test run.

---

# Interview Questions

### Basic

- What problems arise when using threads in a program?
- What happens when you call `fork()` in a multithreaded program?
- What is a deadlock? State the four Coffman conditions.
- What is the difference between deadlock, livelock and starvation?

### Intermediate

- Why is calling `malloc()` after `fork()` in a multithreaded program dangerous?
- How do signals interact with threads in Linux?
- What is false sharing and how do you fix it?
- Why is `volatile` not a synchronisation primitive?

### Advanced

- How would you detect a deadlock in a running Linux process?
- Your thread pool is slower with 32 threads than with 8. Diagnose it and fix it.
- How do you safely fork a child from a multithreaded server?
- Explain how the C11 memory model differs from the POSIX threads model.
- Design a thread lifecycle policy for a server handling 10,000 requests/second.

---

# University Exam Notes

### Definitions

- **Deadlock:** A state in which every thread is blocked, waiting for a resource
  that can only be released by another blocked thread.
- **Livelock:** A state in which threads remain active and responsive but make
  no progress, typically by repeatedly reacting to each other.
- **Starvation:** A state in which a thread is indefinitely denied access to a
  resource it needs, while other threads continue to make progress.
- **False sharing:** Performance loss when independently-used variables share
  a cache line and cause invalidations between cores.
- **Race condition:** A defect where the result depends on the unpredictable
  interleaving of thread operations.
- **Deadlock prevention:** Breaking at least one of the four Coffman
  conditions, typically by imposing a global lock ordering.

### Frequently Asked Questions

- What are the common problems in multithreaded programming?
- Explain deadlock and its four necessary conditions.
- What is livelock? How is it different from deadlock?
- What is starvation? How do you prevent it?
- What is false sharing? How do you detect and fix it?
- Why is `fork()` in a multithreaded program dangerous?
- How do you debug a multithreaded program on Linux?

---

# Key Takeaways

- `fork()` in a multithreaded program copies **only the calling thread**; locks
  held by other threads vanish, silently. Use `fork` + immediate `exec`.
- Signal **dispositions** are shared; signal **masks** are per-thread. Use
  `signalfd()` to avoid handler-safety rules entirely.
- Prefer `_r` (reentrant) library functions, and save `errno` immediately.
- Deadlock needs all four Coffman conditions. **Impose a global lock order** —
  it is the only fix that always works.
- Livelock needs broken symmetry; starvation needs fairness (a semaphore's
  waiter queue is fair, a default mutex is not).
- False sharing is found with `perf c2c record` and fixed with padding.
- For CPU-bound work, one thread per core is the **maximum**. More is slower.
- `volatile` is not a synchronisation primitive. Use `_Atomic` or a mutex.
- **ThreadSanitizer** finds data races. It is the first tool to reach for.

---

# References

- Operating System Concepts — Silberschatz
- Modern Operating Systems — Tanenbaum and Bos
- The Little Book of Semaphores — Allen Downey
- C11 standard, §5.1.2.4 (memory_order) and §7.17 (`<stdatomic.h>`)
- POSIX Base Specifications, Rationale §3.2.3 (the `*_r` suffix)
- `man 2 fork`, `man 2 signalfd`, `man 3 pthread_sigmask`, `man 3 pthread_mutex_trylock`
- ThreadSanitizer: https://clang.llvm.org/docs/ThreadSanitizer.html
