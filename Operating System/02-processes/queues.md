# Queues

> [!NOTE]
> **Module:** Module II – Processes
>
> **Difficulty:** ⭐⭐⭐☆☆
>
> **Prerequisites:**
> - Processes
> - Context Switching
> - Process Control Block
>
> **Scope:** Linux and C only. No Windows or Java.

---

# Learning Objective

After reading this note, you should be able to:

- Define a queue and explain why operating systems need them everywhere.
- Explain the classic queue data structures and their trade-offs.
- Describe the Linux kernel's run queues, wait queues and work queues.
- Explain how the scheduler picks a task from a run queue.
- Explain how blocked tasks are woken from a wait queue.
- Implement a bounded work queue in C with a mutex and a condition variable.
- Choose the right queue implementation for a given requirement.

---

# Why Do We Need Queues?

## The Problem

Almost every subsystem in an operating system has the same shape:

> Something produces events. Something else consumes them, later, at its own
> pace, possibly on a different CPU, possibly after a delay.

The scheduler has runnable tasks waiting for a CPU.

A disk has I/O requests waiting for the controller.

The network stack has packets waiting for processing.

A keyboard driver has keystrokes waiting for a terminal.

A thread pool has jobs waiting for a worker.

The common structure is a **queue**: an ordered collection where items are
added at one end and removed at the other, with a policy for *which* item comes
out next.

> [!NOTE]
> When an operating systems course says "queues", it usually means the classic
> **process ready queue** — the list of runnable processes. But the idea
> generalises, and the generalisation is what you actually need in practice.

---

# Real World Analogy

A queue at a bank counter.

- People **arrive** (enqueue) at the back.
- People are **served** (dequeue) from the front.
- It is **FIFO** — first in, first out.

Now a hospital emergency department:

- The receptionist does not simply serve the next patient.
- Patients are **triaged** and assigned a severity.
- A more severe arriving patient may be seen **before** a less severe one who
  has waited longer.

That is a **priority queue**. Same structure, different removal policy.

And an airport with three check-in desks:

- Incoming passengers are **distributed** among the desks, so that the desk
  with the shortest queue gets the next person.

That is a **work-stealing / work-distributing** structure.

---

# Intuition

```text
  ┌─────────┐    enqueue    ┌───────────────────────┐   dequeue   ┌─────────┐
  │         │ ────────────► │                       │ ──────────► │         │
  │         │               │   Q U E U E            │             │         │
  │producer │               │  ┌─┐┌─┐┌─┐┌─┐┌─┐┌─┐   │             │consumer │
  │         │               │  └─┘└─┘└─┘└─┘└─┘└─┘   │             │         │
  └─────────┘               │    ▲         ▲         │             └─────────┘
                            │    │         │         │
                            │  front     back        │
                            └───────────────────────┘

  The queue's job: BUFFER the rate difference.

  Producer fast, consumer slow  → queue absorbs the burst
  Producer slow, consumer fast  → queue is usually empty (and you have
                                  wasted a data structure)
```

The critical design questions are always the same:

1. **What is the removal policy?** FIFO, priority, LIFO, random?
2. **What is the bound?** Unbounded (memory risk) or bounded (backpressure)?
3. **What happens when it is full?** Block the producer? Drop? Overwrite?
4. **Is it thread safe?** Almost always yes.

---

# Classic Queue Data Structures

| Structure | Removal order | Operations | Use when |
|---|---|---|---|
| Array / ring buffer | FIFO | O(1) amortised | Fixed size, high throughput |
| Linked list | FIFO | O(1) | Size unknown |
| Deque | FIFO or LIFO | O(1) | You need both ends |
| Priority queue (heap) | Highest priority first | O(log n) insert, O(1) remove-min | Order matters, not arrival |
| Sorted array | Priority | O(n) insert | Tiny n, or already sorted |
| Unordered hash "queue" | Arbitrary | O(1) | Order does not matter |
| Multi-producer / multi-consumer queue | FIFO | O(1) with a lock | Concurrent access |

## The ring buffer

```text
  head = 3                          (next slot to read)
  tail = 1                          (next slot to write)
  count = 4                         (elements present)

  index:  0     1     2     3     4     5     6     7
        ┌─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┐
        │ 'a' │ 'b' │  ·  │  ·  │  ·  │  ·  │  ·  │  ·  │
        └─────┴─▲───┴─────┴─▲───┴─────┴─────┴─────┴─────┘
                  │           │
                tail         head

  Writing goes to tail, reading from head, and both wrap around.
  An empty queue is head == tail. A full queue is head == tail too,
  so you need a separate count (or a wasted slot) to tell them apart.
```

## A bounded MPMC queue in C

```c
#include <pthread.h>
#include <stdatomic.h>
#include <stdio.h>
#include <string.h>

#define CAPACITY 1024

struct mpmc_queue {
    void              *buf[CAPACITY];
    atomic_size_t      head;        /* next to pop  */
    atomic_size_t      tail;        /* next to push */
    pthread_mutex_t    lock;
    pthread_cond_t     not_empty;
    pthread_cond_t     not_full;
    size_t             count;
};

static void q_init(struct mpmc_queue *q)
{
    atomic_init(&q->head, 0);
    atomic_init(&q->tail, 0);
    q->count = 0;
    pthread_mutex_init(&q->lock, NULL);
    pthread_cond_init(&q->not_empty, NULL);
    pthread_cond_init(&q->not_full, NULL);
}

static void q_push(struct mpmc_queue *q, void *item)
{
    pthread_mutex_lock(&q->lock);

    while (q->count == CAPACITY)
        pthread_cond_wait(&q->not_full, &q->lock);   /* backpressure */

    q->buf[atomic_fetch_add(&q->tail, 1) % CAPACITY] = item;
    q->count++;

    pthread_cond_signal(&q->not_empty);
    pthread_mutex_unlock(&q->lock);
}

static void *q_pop(struct mpmc_queue *q)
{
    pthread_mutex_lock(&q->lock);

    while (q->count == 0)
        pthread_cond_wait(&q->not_empty, &q->lock);  /* block until work */

    void *item = q->buf[atomic_fetch_add(&q->head, 1) % CAPACITY];
    q->count--;

    pthread_cond_signal(&q->not_full);
    pthread_mutex_unlock(&q->lock);
    return item;
}

static int q_empty(struct mpmc_queue *q)
{
    pthread_mutex_lock(&q->lock);
    int e = (q->count == 0);
    pthread_mutex_unlock(&q->lock);
    return e;
}
```

> [!NOTE]
> The `while` loop, not `if`, in the condition-variable wait is mandatory.
> A condition variable can wake spuriously, and `pthread_cond_wait` can return
> `EINTR`. Always re-check the predicate in a loop.

## A priority queue in C

```c
#include <stdlib.h>
#include <stdio.h>

/* A binary max-heap: the largest priority comes out first. */
struct heap {
    void  **data;
    long   *prio;
    size_t  size;
    size_t  cap;
};

static void heap_push(struct heap *h, void *item, long prio)
{
    if (h->size == h->cap) {
        h->cap = h->cap ? h->cap * 2 : 8;
        h->data = realloc(h->data, h->cap * sizeof *h->data);
        h->prio = realloc(h->prio, h->cap * sizeof *h->prio);
        if (!h->data || !h->prio) { perror("realloc"); exit(1); }
    }

    size_t i = h->size++;
    h->data[i] = item;
    h->prio[i] = prio;

    while (i > 0) {                          /* sift up */
        size_t p = (i - 1) / 2;
        if (h->prio[p] >= h->prio[i]) break;
        long t = h->prio[p]; h->prio[p] = h->prio[i]; h->prio[i] = t;
        void *d = h->data[p]; h->data[p] = h->data[i]; h->data[i] = d;
        i = p;
    }
}

static void *heap_pop(struct heap *h)
{
    if (h->size == 0) return NULL;

    void *top = h->data[0];
    h->data[0] = h->data[--h->size];         /* move last to root */
    h->prio[0] = h->prio[h->size];

    size_t i = 0;                            /* sift down */
    for (;;) {
        size_t l = 2 * i + 1, r = l + 1, m = i;
        if (l < h->size && h->prio[l] > h->prio[m]) m = l;
        if (r < h->size && h->prio[r] > h->prio[m]) m = r;
        if (m == i) break;
        long t = h->prio[m]; h->prio[m] = h->prio[i]; h->prio[i] = t;
        void *d = h->data[m]; h->data[m] = h->data[i]; h->data[i] = d;
        i = m;
    }
    return top;
}
```

| Insert | Remove-max |
|---|---|
| O(log n) | O(log n) |

Compare with a FIFO's O(1) / O(1). The priority queue buys ordering with
logarithmic cost. Use it only if ordering is worth it.

---

# Kernel Queues in Linux

The kernel maintains several structurally different kinds of queue.

## 1. Run Queues — the Ready Queue

Each CPU has its own run queue (`struct cfs_rq`). It holds the tasks that are
runnable on that CPU.

```text
  CPU 0                                         CPU 1
  ┌──────────────────────────┐                 ┌──────────────────────────┐
  │      cfs_rq (runqueue)   │                 │      cfs_rq (runqueue)   │
  │  ┌────────────────────┐  │                 │  ┌────────────────────┐  │
  │  │  red-black tree    │  │                 │  │  red-black tree    │  │
  │  │  keyed on vruntime │  │                 │  │  keyed on vruntime │  │
  │  │                    │  │                 │  │                    │  │
  │  │   task A  vruntime  │  │   ◄── load ──► │  │   task F  vruntime  │  │
  │  │   task C  vruntime  │  │   balancer     │  │   task G  vruntime  │  │
  │  │   task D  vruntime  │  │                 │  │   task H  vruntime  │  │
  │  └────────────────────┘  │                 │  └────────────────────┘  │
  │                          │                 │                          │
  │  +leftmost = next to run │                 │  +leftmost = next to run │
  └──────────────────────────┘                 └──────────────────────────┘
```

| Property | Value |
|---|---|
| What it holds | Runnable tasks |
| Data structure | red-black tree keyed on `vruntime` (CFS) |
| One per | CPU |
| Min element | the leftmost node — the task that has run least |
| Why a tree, not a list | O(log n) pick, and O(log n) arbitrary removal |
| Also holds | a `min_vruntime` marker for fairness accounting |

```c
/* Structurally: what the kernel uses. */
struct cfs_rq {
    struct rb_root    tasks_timeline;   /* the red-black tree */
    struct sched_entity *leftmost;      /* cached: next task to run */
    unsigned long     nr_running;       /* how many are on it */
    ...
};

/* The sched_entity is embedded in task_struct, and also in group structs. */
struct sched_entity {
    struct list_node   group_node;      /* if this is a task group */
    unsigned int       on_rq;           /* am I enqueued? */
    unsigned long      vruntime;       /* my fair-share score */
    u64                sum_exec_runtime;   /* actual CPU ns used */
    u64                prev_sum_exec_runtime;
    ...
};
```

```bash
# How many runnable tasks, and the load average
cat /proc/loadavg
# 2.31 1.87 1.62  2/1387 20114
#            ^ 2 runnable/uninterruptible, 1387 total

# See the switches the scheduler makes
perf record -e sched:sched_switch -a -- sleep 5
perf script | head
```

```text
sched_switch:
  prev_comm=kworker/2:1 prev_pid=1245 prev_prio=120 prev_state=S (runnable)
  ==> next_comm=nginx next_pid=2301 next_prio=120
```

### How CFS picks the next task

```text
  1. Every task on the run queue has a vruntime.
  2. vruntime increases by roughly:
         actual_execution_time  ×  (nice_weight / NICE_0_LOAD)
  3. A nice -20 task's vruntime grows ~10× slower than a nice 0 task.
  4. The scheduler runs the task with the SMALLEST vruntime.
  5. Result: a small-nice task's vruntime grows slowly, so it keeps being the
     smallest, so it runs proportionally more. That is the whole fairness
     mechanism.
```

```c
/* kernel/sched/fair.c — the core of the idea, simplified */
static void update_curr(struct cfs_rq *cfs_rq, struct sched_entity *se)
{
    u64 delta_exec = exec_runtime(se);
    se->vruntime += calc_delta_fair(delta_exec, se);
    se->sum_exec_runtime += delta_exec;
    sfb->last_update = sfb->curr_time;
}

/* the pick */
se = rb_first_cfs_rq(cfs_rq);     /* leftmost = smallest vruntime */
```

## 2. Wait Queues — the Blocking Queue

When a task blocks, it does not sit on the run queue. It goes onto a **wait
queue** belonging to whatever it is waiting for.

```text
  blocked on: read() from /dev/sda
              │
              ▼
  ┌──────────────────────────────────────┐
  │  wait_queue_head_t                   │
  │  (one per device, in the driver's     │
  │   request struct)                     │
  │                                      │
  │   ┌────────┐  ┌────────┐  ┌────────┐  │
  │   │ task A │  │ task B │  │ task C │  │
  │   │ wants  │  │ wants  │  │ wants  │  │
  │   │ data   │  │ data   │  │ EOF    │  │
  │   └────────┘  └────────┘  └────────┘  │
  └──────────────┬───────────────────────┘
                 │
                 │ the disk completes a request
                 ▼
         wake_up_interruptible(&wq)
                 │
                 ├── task A moved to a run queue
                 ├── task B moved to a run queue
                 └── task C is woken but re-checks its condition
                     and goes back to sleep  ← spurious wakeup
```

| Property | Value |
|---|---|
| What it holds | Blocked tasks |
| Data structure | Linked list of `wait_queue_entry` |
| One per | Whatever can block — device, mutex, condition |
| Waking | `wake_up()`, `wake_up_interruptible()`, `wake_up_sync()` |
| Ordering | FIFO on the list, but the driver decides which to wake |

```c
/* kernel/wait.c — conceptually */
#define add_wait_queue(q, e)                   \
    list_add_tail(&(e)->entry, &(q)->head)

#define wake_up_interruptible(x)               \
    ({ __wake_up(x, TASK_INTERRUPTIBLE, 1); })

struct wait_queue_head {
    spinlock_t      lock;
    struct list_head head;
};

struct wait_queue_entry {
    unsigned int    flags;      /* WQ_FLAG_EXCLUSIVE etc. */
    void           *private;     /* the task_struct, usually */
    wait_queue_entry_t *next;
    wait_queue_entry_t *prev;
};
```

### Spurious wakeups — why the `while` loop exists

```c
/* CORRECT */
while (count == 0)
    cond_wait(&cv, &mutex);
do_work();

/* WRONG */
if (count == 0)
    cond_wait(&cv, &mutex);
do_work();      /* may run with count still 0 */
```

Spurious wakeups are not rare. A `pthread_cond_signal` for one waiter, a
`SIGCHLD` handler, or a kernel re-check can cause a wake with the predicate
still false. The condition-variable `while` loop is not defensive coding; it is
the specification.

## 3. Work Queues — Deferred Work in Kernel Threads

A work queue runs functions in the context of a **kernel thread**, so the work
may sleep and may allocate memory.

```text
  HARD IRQ  (cannot sleep)
     │
     │ schedule_work(&my_work)
     ▼
  ┌──────────────────────────────────────────────┐
  │  workqueue "events"                          │
  │  ┌────────────────────────────────────────┐  │
  │  │ pending: my_work, disk_write, ...      │  │  ◄── the queue
  │  └────────────────────────────────────────┘  │
  │         │                                    │
  │         ▼                                    │
  │  kworker/3:2   ← a real kernel thread        │
  │  runs my_work() in process context           │
  │  sleeping and allocating: ALLOWED            │
  └──────────────────────────────────────────────┘
```

```c
#include <linux/workqueue.h>

static void my_worker(struct work_struct *work);

/* An ordered workqueue: items run in submission order, one at a time */
static struct workqueue_struct *my_wq;

/* A memory-reclaim-ordered workqueue: for shrinker callbacks */
static struct workqueue_struct *my_mem_wq;

static int __init my_init(void)
{
    my_wq = alloc_ordered_workqueue("my_driver", WQ_MEM_RECLAIM);
    my_mem_wq = alloc_workqueue("my_driver_mem", WQ_MEM_RECLAIM);

    struct work_struct *w = kmalloc(sizeof *w, GFP_KERNEL);
    INIT_WORK(w, my_worker);
    queue_work(my_wq, w);          /* becomes "pending" until it runs */
    return 0;
}

static void my_worker(struct work_struct *work)
{
    /* Process context. Sleeping, allocating, and locking are all fine here. */
    kfree(work);
}
```

| Type | Concurrency | Order | Use for |
|---|---|---|---|
| `create_workqueue()` | parallel | none | general work |
| `alloc_ordered_workqueue()` | single | FIFO | dependent steps |
| `alloc_workqueue(... WQ_MEM_RECLAIM)` | any | none | shrinker / memory reclaim |
| `queue_delayed_work()` | any | by timer | timeouts |
| `schedule_work()` | any | none | one-shot |

## 4. The complete kernel picture

```text
  ┌─────────────────────────────────────────────────────────────────┐
  │ RUN QUEUES (one per CPU)                                        │
  │   cfs_rq  →  red-black tree of sched_entity, keyed on vruntime    │
  │   holds: runnable tasks, including the one currently on the CPU  │
  └─────────────────────────────────────────────────────────────────┘
        ▲                            │
        │ task becomes runnable       │ task blocks
        │ (I/O done, lock released,   │ (read with no data, mutex held,
        │  timer fired, yield)        │  wait4, futex)
        │                            ▼
  ┌─────┴──────────────────┐  ┌────────────────────────────────────┐
  │  timers / hrtimers     │  │ WAIT QUEUES                        │
  │  timeout, itimer,      │─►│   wait_queue_head_t per resource   │
  │  POSIX timer_create    │  │   list of wait_queue_entry         │
  └────────────────────────┘  └────────────────────────────────────┘

  ┌────────────────────────────────────────────────────────────────┐
  │ WORK QUEUES — run in kworker threads, may sleep                 │
  │   ordinary work, ordered work, memory-reclaim work             │
  └────────────────────────────────────────────────────────────────┘

  ┌────────────────────────────────────────────────────────────────┐
  │ LOCK QUEUES — waiters on a contended mutex/spinlock            │
  │   mutex->wait_list, PI waiters, RT priority waiters            │
  └────────────────────────────────────────────────────────────────┘

  ┌────────────────────────────────────────────────────────────────┐
  │ NETWORK — per-CPU / per-NUMA backlog queues                    │
  │   qdisc netdev queues, socket accept backlog, NIC ring buffers  │
  └────────────────────────────────────────────────────────────────┘
```

```bash
# How many runnable tasks
cat /proc/loadavg

# Where runnable tasks live right now
for t in /proc/[0-9]*; do
    printf "%s %s\n" "$(basename "$t")" "$(cat $t/stat 2>/dev/null | awk '{print $3}')"
done | awk '$2=="R"{c++} END{print c+0, "tasks in state R"}'

# Which runqueue
ps -eo pid,psr,stat,comm | head

# Work queues
ps -eo comm | grep kworker | head
ps -eo pid,comm | grep kworker
```

---

# Userspace Queues: POSIX Message Queues and Socket Backlogs

## POSIX message queue

A queue of fixed-size messages in the kernel, addressed by name, with
priorities.

```text
  /my_queue

  ┌────┬────┬────┬────┬────────┐
  │ p0 │ p1 │ p2 │ p3 │  ...   │     mq_maxmsg = 10
  │pri │pri │pri │pri │        │     mq_msgsize = 256
  └────┴────┴────┴────┴────────┘

  mq_send()  → highest priority first out
  mq_receive() with NULL priority → head of the queue (FIFO within a priority)
  mq_receive() with a priority    → the first message of that priority
```

```c
#include <mqueue.h>
#include <fcntl.h>
#include <stdio.h>
#include <string.h>
#include <errno.h>

int main(void)
{
    struct mq_attr a = { .mq_maxmsg = 4, .mq_msgsize = 64, .mq_flags = 0 };
    mqd_t q = mq_open("/demo", O_CREAT | O_RDWR, 0600, &a);
    if (q == (mqd_t)-1) { perror("mq_open"); return 1; }

    if (mq_send(q, "low",  3, 1) != 0) perror("mq_send");
    if (mq_send(q, "high", 4, 9) != 0) perror("mq_send");
    if (mq_send(q, "mid",  3, 5) != 0) perror("mq_send");

    /* full: the next send must either block or fail */
    if (mq_send(q, "x", 1, 0) != 0)
        printf("queue full: %s\n", strerror(errno));

    char buf[64];
    unsigned prio;
    for (int i = 0; i < 3; i++) {
        ssize_t n = mq_receive(q, buf, sizeof buf, &prio);
        if (n > 0) printf("prio %u: %.*s\n", prio, (int)n, buf);
    }

    mq_close(q);
    mq_unlink("/demo");
    return 0;
}
```

```bash
gcc -Wall -Wextra -lrt -o mqdemo mqdemo.c
./mqdemo
```

```text
prio 9: high
prio 5: mid
prio 1: low
queue full: Resource temporarily unavailable
```

> [!TIP]
> `mq_send` on a full queue **blocks** by default, or returns `EAGAIN` if you
> pass `O_NONBLOCK`. That is backpressure, and it is a feature: it stops a
> fast producer from exhausting memory.

## Socket accept backlog

```c
int listen(int sockfd, int backlog);
```

`backlog` is the length of the **accept queue** of completed connections
waiting to be accepted. It is not a rate limit, and it does not bound the
number of connections in `SYN_RECV`.

```bash
ss -ltn            # Send-Q / Recv-Q per listening socket
ss -ltn 'sport = :80'
netstat -ltn

# The historical maximum, before 4.1
sysctl net.core.somaxconn
```

```text
LISTEN  Recv-Q  Send-Q   Local Address:Port  Peer Address:Port
0       0       4096     0.0.0.0:80          0.0.0.0:*
             ^        ^
             backlog  somaxconn (capped)
```

> [!NOTE]
> Before Linux 2.6.45, the effective backlog was `backlog + 1`. Since 2.6.45
> (and 4.1 for `SOMAXCONN` accounting) it is `min(backlog, somaxconn)`.
> This is why an old book and a modern system can disagree.

## NIC ring buffers

The network driver allocates descriptor rings: the producer (the receiving
side) fills a ring, the consumer (the NAPI poll loop) drains it.

```bash
ethtool -g eth0          # ring sizes
ethtool -c eth0          # interrupt coalescing
ethtool -k eth0          # offload features
```

A small ring plus interrupt coalescing is the classic tuning for high-rate
traffic: fewer interrupts, more batching, slightly more latency.

---

# Implementing a Thread Pool: A Real Work Queue

A work queue where consumers are threads, not processes.

```c
#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>

#define NWORKERS  4
#define QUEUE_CAP  64

typedef void (*task_fn)(void *arg);

struct task {
    task_fn fn;
    void   *arg;
};

struct pool {
    struct task        queue[QUEUE_CAP];
    int                head, tail, count;

    pthread_mutex_t    mutex;
    pthread_cond_t     not_empty;    /* workers wait here */
    pthread_cond_t     not_full;     /* submitters wait here */
    int                shutdown;
};

static struct pool g;

static void pool_init(void)
{
    memset(&g, 0, sizeof g);
    pthread_mutex_init(&g.mutex, NULL);
    pthread_cond_init(&g.not_empty, NULL);
    pthread_cond_init(&g.not_full, NULL);
}

static int pool_submit(task_fn fn, void *arg)
{
    pthread_mutex_lock(&g.mutex);

    while (g.count == QUEUE_CAP && !g.shutdown)
        pthread_cond_wait(&g.not_full, &g.mutex);      /* backpressure */

    if (g.shutdown) {
        pthread_mutex_unlock(&g.mutex);
        return -1;
    }

    g.queue[g.tail].fn  = fn;
    g.queue[g.tail].arg = arg;
    g.tail = (g.tail + 1) % QUEUE_CAP;
    g.count++;

    pthread_cond_signal(&g.not_empty);
    pthread_mutex_unlock(&g.mutex);
    return 0;
}

static void *worker(void *unused)
{
    (void)unused;

    for (;;) {
        pthread_mutex_lock(&g.mutex);

        while (g.count == 0 && !g.shutdown)
            pthread_cond_wait(&g.not_empty, &g.mutex);

        if (g.count == 0 && g.shutdown) {
            pthread_mutex_unlock(&g.mutex);
            return NULL;
        }

        struct task t = g.queue[g.head];
        g.head = (g.head + 1) % QUEUE_CAP;
        g.count--;

        pthread_cond_signal(&g.not_full);
        pthread_mutex_unlock(&g.mutex);

        t.fn(t.arg);                  /* run OUTSIDE the lock */
    }
}

static void square(void *arg)
{
    long n = (long)arg;
    printf("sqrt(%ld) = %.4f\n", n, (double)__builtin_sqrt((double)n));
}

int main(void)
{
    pool_init();

    pthread_t tids[NWORKERS];
    for (int i = 0; i < NWORKERS; i++)
        pthread_create(&tids[i], NULL, worker, NULL);

    for (long i = 1; i <= 20; i++)
        pool_submit(square, (void *)i);

    pthread_mutex_lock(&g.mutex);
    g.shutdown = 1;
    pthread_cond_broadcast(&g.not_empty);
    pthread_mutex_unlock(&g.mutex);

    for (int i = 0; i < NWORKERS; i++)
        pthread_join(tids[i], NULL);

    return 0;
}
```

```bash
gcc -Wall -Wextra -pthread -o pool pool.c
./pool
```

## Design points that matter

| Decision | Consequence |
|---|---|
| Run `t.fn()` outside the lock | Workers run in parallel. Holding the lock would serialise the whole pool. |
| `while` in every `cond_wait` | Handles spurious wakeups correctly. |
| `broadcast` on shutdown | Wakes **all** workers. `signal` would leave most asleep forever. |
| Bounded queue | Gives you backpressure and a bound on memory. |
| Unbounded queue | Never blocks the producer, but an unbounded memory leak waiting to happen. |
| `pthread_join` before exit | Reaps threads; avoids a "thread leak" on every loop of a server. |

---

# Choosing a Queue for a Job

```text
  Do tasks need to run in submission order?
        │
        ├── YES → ordered queue / ordered workqueue / FIFO
        │
        └── NO  → do tasks have different urgencies?
                    │
                    ├── YES → priority queue (heap, or run queue with vruntime)
                    │
                    └── NO  → FIFO ring buffer or linked list
                              use MPMC queue if threads produce and consume
```

| Requirement | Structure |
|---|---|
| Fixed capacity, no allocation | Ring buffer |
| Unknown capacity, single producer/consumer | Lock-free ring buffer (SPSC) |
| Unknown capacity, many threads | Mutex + condvar queue, or lock-free MPMC |
| Ordered by priority | Binary heap |
| Task must not wait | Bounded queue + reject or block |
| Producer must never block | Unbounded queue, with a size limit you alert on |
| Runs in the kernel, may sleep | Workqueue |
| Runs in the kernel, must not sleep | Softirq or tasklet |

---

# Common Misconceptions

### ❌ "The ready queue is a FIFO."

Incorrect, and this is the most important correction in this note.

Linux's run queue is a **red-black tree keyed on `vruntime`**, and the task
that runs next is the one with the *smallest* vruntime, not the oldest arrival.
Fairness is based on accumulated CPU time, not on arrival order.

---

### ❌ "A queue always has bounded capacity."

Incorrect.

Unbounded queues exist and are common in naive designs. The trouble surfaces
under load, as unbounded memory growth and then an OOM kill.

---

### ❌ "A condition variable can be used with `if`."

Incorrect.

`pthread_cond_wait` may return even when the predicate is false (spurious
wakeup) or when interrupted by a signal. It must be wrapped in a `while` loop
re-testing the actual condition.

---

### ❌ "Work queues and run queues are the same thing."

Incorrect.

A run queue holds *tasks* and is about CPU time. A workqueue holds *pending
work functions* and is about deferring work into a kernel thread context.

---

### ❌ "A full socket backlog means the server is overloaded and dropping."

Incorrect, most of the time.

A full accept backlog means connections complete the handshake and wait. They
are refused only once both the accept queue and the SYN queue are full, or
`tcp_abort_on_overflow` is set.

---

# Interview Questions

### Basic

- What is a queue? Why does an OS need queues?
- What are the different queue data structures and their complexities?
- What is a ready queue?
- What is a priority queue?

### Intermediate

- How does the Linux scheduler pick the next task from a run queue?
- What is a wait queue, and what is a spurious wakeup?
- What is a workqueue, and when would you use one instead of a softirq?
- Implement a bounded producer-consumer queue in C.

### Advanced

- Why is the Linux run queue a red-black tree rather than a linked list?
- How do you shut down a thread pool correctly, and what happens if you get it wrong?
- What is thundering herd, and how do you avoid it in a work queue?
- How would you choose between a bounded and an unbounded queue for a network server?
- Explain how `nice` values affect `vruntime` and the resulting scheduling fairness.

---

# University Exam Notes

### Definitions

- **Queue:** A linear data structure in which insertion and deletion occur at
  opposite ends, following a specified order.
- **Process ready queue:** The collection of processes that are in the ready
  state, awaiting CPU dispatch.
- **Priority queue:** A queue from which elements are removed in order of
  priority rather than arrival.
- **Circular queue (ring buffer):** A fixed-capacity queue in which the
  indices wrap around to the beginning.
- **Wait queue:** A kernel structure holding processes blocked on a resource.
- **Workqueue:** A kernel mechanism for running functions in a kernel thread
  context, where they may sleep.
- **Spurious wakeup:** A wakeup delivered when the waiting condition is not
  satisfied.

### Frequently Asked Questions

- What is a queue? Explain the different types of queue data structures.
- What is a ready queue? How does the scheduler select from it?
- Explain circular queues with an example.
- What is a priority queue and how is it implemented?
- What are wait queues? What is a spurious wakeup?
- Differentiate between a run queue and a workqueue.
- Write a producer-consumer program using a bounded queue.

---

# Key Takeaways

- A queue buffers a rate mismatch between a producer and a consumer.
- Structures: ring buffer (O(1) FIFO), deque, priority queue/heap
  (O(log n)), and MPMC queues for concurrent access.
- **Every `cond_wait` must be wrapped in a `while` loop** — spurious wakeups
  are part of the contract, not a bug.
- Linux has structurally different queues: run queues (per CPU, red-black tree
  on `vruntime`), wait queues (per resource, FIFO list of blocked tasks),
  workqueues (per pool, run in `kworker` threads), and lock queues.
- The Linux run queue is **not a FIFO**: CFS picks the smallest `vruntime`,
  which is how `nice` becomes proportional fairness.
- A full message queue or socket backlog is **backpressure**, and it is a
  feature you should use deliberately.
- Run worker functions **outside** the lock, or you have built a serialiser.

---

# References

- Operating System Concepts — Silberschatz
- Modern Operating Systems — Tanenbaum and Bos
- The Linux Programming Interface — Kerrisk
- Linux Kernel Development — Robert Love
- Documentation/scheduler/sched-design-CFS.rst
- `man 3 pthread_cond_wait`, `man 3 mq_send`, `man 2 listen`
- `ss(8)`, `ipcs(1)`, `ethtool(8)`, `vmstat(1)`
- The Little Book of Semaphores — Allen Downey
