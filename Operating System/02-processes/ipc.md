# Inter-Process Communication (IPC)

> [!NOTE]
> **Module:** Module II – Processes
>
> **Difficulty:** ⭐⭐⭐⭐☆
>
> **Prerequisites:**
> - Processes
> - Critical Section
>
> **Scope:** Linux and C only. No Windows or Java.

---

# Learning Objective

After reading this note, you should be able to:

- Explain why processes need IPC and what makes it difficult.
- List the IPC mechanisms Linux provides and classify them.
- Write C programs using pipes, FIFOs, message queues and shared memory.
- Explain the difference between System V and POSIX IPC.
- Explain the producer-consumer problem and how shared memory requires
  synchronisation that shared memory does not provide.
- Choose the right IPC mechanism for a given problem.
- Debug IPC programs with the right tools.

---

# Why Do We Need IPC?

## The Problem

Isolation gives processes privacy: one process cannot read another's memory.

But real programs are not islands.

- A shell needs to run a program and read its output.
- A web server needs to talk to a database.
- A build system needs to know when 8 compiler processes are done.
- Two unrelated programs on a machine need to agree on a lock file.

Isolation is a feature for security, and an obstacle for cooperation.

IPC is the set of mechanisms the OS provides to let processes cooperate
**without** giving up their memory isolation.

---

# Real World Analogy

Two offices on different floors cannot enter each other's rooms. That is
**isolation**.

They still need to work together. So the building provides shared resources:

| Building facility | IPC equivalent |
|---|---|
| A pneumatic tube between floors | pipe, Unix domain socket |
| A shared noticeboard in the lobby | shared file, file-backed `mmap` |
| A numbered tray where you leave a note | message queue |
| A lamp outside the room (on = busy) | semaphore, `flock` |
| A receptionist who relays messages | socket, TCP connection |
| Everyone shouting in the atrium | signals, broadcast on a channel |

Each has different properties: cost, reliability, whether it blocks, whether
ordering is guaranteed, and whether it needs a helper process.

---

# Intuition

Every IPC mechanism is a **naming question** plus a **data question**.

```text
  1. NAMING: how do the two processes find each other?
       • an inherited file descriptor      (pipe)
       • a path in the filesystem          (FIFO, Unix socket, file)
       • a key                            (System V: ftok())
       • a well-known name                (POSIX: "/myqueue")
       • port number + address             (TCP/UDP)
       • PID                               (signal)
       • an address in shared memory       (shm_open + mmap)

  2. DATA: how do the bytes move?
       • the kernel copies them            (pipe, message queue, socket)
       • a shared page, no copy at all     (shared memory)
       • a signal with no payload          (signal)
```

Shared memory is unique: it answers question 1 for the *address*, and then
bypasses question 2 entirely.

---

# Classification of IPC Mechanisms

```text
                     Linux IPC
                         │
        ┌────────────────┴─────────────────┐
        │                                  │
  LOCAL (same host)                 NETWORK / REMOTE
        │                                  │
   ┌────┴──────────────┐            ┌──────┴────────┐
   │                   │            │               │
 Data passing      Coordination   BSD sockets    Netlink
   │                   │            │               │
 ┌─┴────┬────┬────┐   ├─────────┐   ├───────────┤
 │      │    │    │   │         │   │           │
pipe  FIFO  mmap  socket  sem  mutex  Unix domain  TCP/UDP
             (shm)                socket (AF_UNIX)
                       signal
```

| Mechanism | Data copies | Ordering | Blocking | Needs a file? | Best for |
|---|---|---|---|---|---|
| `pipe()` | 2 copies | byte stream | yes | no | Parent → child |
| FIFO | 2 copies | byte stream | yes | yes (a path) | Unrelated processes |
| Shared memory | **0** | none | no | yes | Bulk data transfer |
| Message queue | 2 copies | message order | yes | yes (POSIX) | Discrete messages |
| Unix socket | 2 copies | byte stream | yes | yes (usually) | Local client/server |
| TCP socket | 2 copies | byte stream | yes | no | Across machines |
| Signal | n/a | n/a | usually not | no | Events, not data |

> [!NOTE]
> "2 copies" means: one copy from the sender's buffer into the kernel, and one
> from the kernel into the receiver's buffer. That is the cost of crossing a
> privilege boundary. Shared memory is the only mechanism that avoids it.

---

# Pipes

## Definition

A **pipe** is a unidirectional, byte-oriented, kernel buffer with a read end
and a write end.

```text
   writer  ────▶  ┌──────────────┐  ────▶  reader
   (fd_out)        │ kernel buffer │         (fd_in)
                   │   64 KiB      │
                   └──────────────┘
```

| Property | Value |
|---|---|
| Direction | unidirectional (one read end, one write end) |
| Data | raw bytes, no message boundaries |
| Buffer | 64 KiB by default, in 4 KiB pages |
| Full | write blocks until the reader drains |
| Empty | read blocks until the writer writes |
| Closed | write end closed → read returns 0 (EOF) |
| Name | none — an inherited file descriptor |

## Writing to a Pipe

```c
#include <stdio.h>
#include <string.h>
#include <unistd.h>

int main(void)
{
    int fd[2];                          /* fd[0] = read, fd[1] = write */

    if (pipe(fd) == -1) {
        perror("pipe");
        return 1;
    }

    const char *msg = "hello through a pipe\n";
    ssize_t n = write(fd[1], msg, strlen(msg));
    if (n < 0)
        perror("write");

    close(fd[1]);                       /* closing signals EOF to the reader */

    char buf[128];
    ssize_t got = read(fd[0], buf, sizeof buf - 1);
    if (got > 0) {
        buf[got] = '\0';
        printf("read %zd bytes: %s", got, buf);
    }

    close(fd[0]);
    return 0;
}
```

```bash
gcc -Wall -Wextra -o pipetest pipetest.c
./pipetest
```

## The `SIGPIPE` Problem

```text
  reader closes fd[0]
            │
            ▼
  writer calls write(fd[1], ...)
            │
            ▼
  SIGPIPE is delivered → by default the writer is TERMINATED
```

This is why a well-behaved writer ignores `SIGPIPE` and checks for `EPIPE`:

```c
#include <signal.h>
#include <errno.h>

signal(SIGPIPE, SIG_IGN);

/* ... later ... */
ssize_t n = write(fd[1], buf, len);
if (n < 0 && errno == EPIPE) {
    /* the reader is gone. Handle it. Do not die. */
}
```

```bash
# Reproduce the default behaviour
kill -l | grep -i pipe
# 13) SIGPIPE
```

## Communicating Between Parent and Child

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>

int main(void)
{
    int fd[2];
    if (pipe(fd) == -1) { perror("pipe"); return 1; }

    pid_t pid = fork();
    if (pid < 0) { perror("fork"); return 1; }

    if (pid == 0) {
        /* CHILD: write to the pipe */
        close(fd[0]);                       /* the child does not read */

        char line[256];
        printf("child: what shall I send? ");
        fflush(stdout);
        if (fgets(line, sizeof line, stdin)) {
            ssize_t n = write(fd[1], line, strlen(line));
            if (n < 0) perror("write");
        }

        close(fd[1]);
        _exit(0);
    }

    /* PARENT: read from the pipe */
    close(fd[1]);                           /* the parent does not write */

    char buf[256];
    ssize_t n = read(fd[0], buf, sizeof buf - 1);
    if (n > 0) {
        buf[n] = '\0';
        printf("parent received: %s", buf);
    }

    close(fd[0]);
    wait(NULL);
    return 0;
}
```

```bash
gcc -Wall -Wextra -o parentchild parentchild.c
./parentchild
```

```text
child: what shall I send? hello from the child
parent received: hello from the child
```

## Shell Pipelines Are Pipes

```bash
ls -l | grep ".md" | wc -l
```

```text
  shell does this for you:

    pid1 = fork();  exec("ls", "-l")      → stdout → pipe A
    pid2 = fork();  exec("grep", ".md")   → stdin = pipe A, stdout → pipe B
    pid3 = fork();  exec("wc", "-l")      → stdin = pipe B

  Each | creates a new pipe(2) call.
```

```bash
# See them
strace -f -e trace=pipe2,clone,dup2 sh -c 'ls | grep md | wc -l'
```

---

# FIFOs (Named Pipes)

## Definition

A **FIFO** is a pipe with a name in the filesystem. Unrelated processes open
it by path.

```c
#include <fcntl.h>
#include <stdio.h>
#include <string.h>
#include <sys/stat.h>
#include <sys/wait.h>
#include <unistd.h>

int main(void)
{
    const char *path = "/tmp/demo_fifo";

    unlink(path);
    if (mkfifo(path, 0666) == -1) {       /* create it */
        perror("mkfifo");
        return 1;
    }

    if (fork() == 0) {
        /* WRITER */
        int fd = open(path, O_WRONLY);     /* blocks until a reader appears */
        if (fd < 0) { perror("open"); _exit(1); }
        write(fd, "written by the writer\n", 23);
        close(fd);
        _exit(0);
    }

    /* READER */
    int fd = open(path, O_RDONLY);         /* blocks until a writer appears */
    if (fd < 0) { perror("open"); return 1; }

    char buf[64];
    ssize_t n = read(fd, buf, sizeof buf - 1);
    if (n > 0) { buf[n] = '\0'; printf("read: %s", buf); }

    close(fd);
    wait(NULL);
    unlink(path);
    return 0;
}
```

```bash
gcc -Wall -Wextra -o fifo fifo.c
./fifo

# Or, entirely from the shell
mkfifo /tmp/f1
(echo "hello" > /tmp/f1) &      # blocks until a reader opens it
cat /tmp/f1
rm /tmp/f1
```

> [!IMPORTANT]
> `open()` on a FIFO blocks until the **other end** is also open. This is
> deliberate: it removes a race where the writer could write and exit before
> the reader even started. Use `O_NONBLOCK` if you need to avoid it.

## FIFOs vs Unix Sockets

| | FIFO | Unix domain socket |
|---|---|---|
| Bidirectional | no | yes |
| Message boundaries | no (byte stream) | yes (`SOCK_SEQPACKET`), or stream |
| Peer discovery | you must find the peer | `connect()` on an address |
| Data passed | by inherited fd or `SCM_RIGHTS` | in the socket address |
| Typical use | shell pipelines, simple producer/consumer | local client/server servers |

---

# Message Queues

A message queue preserves **message boundaries**: a write of 100 bytes is read
as one 100-byte message, not split arbitrarily.

## System V

```c
#include <stdio.h>
#include <string.h>
#include <sys/ipc.h>
#include <sys/msg.h>
#include <sys/types.h>
#include <unistd.h>

/* Every System V message must start with this. */
struct mymsg {
    long mtype;                            /* the "type"/channel */
    char text[128];
};

int main(void)
{
    /* ftok(key, id) → a unique key from a file and a project id */
    key_t key = ftok("/tmp/myapp", 'A');
    if (key == (key_t)-1) { perror("ftok"); return 1; }

    int qid = msgget(key, IPC_CREAT | 0600);
    if (qid < 0) { perror("msgget"); return 1; }

    pid_t pid = fork();
    if (pid == 0) {
        struct mymsg m;
        m.mtype = 42;                      /* receiver filters on this */
        strcpy(m.text, "a whole message, boundaries preserved");
        if (msgsnd(qid, &m, sizeof m, 0) < 0) perror("msgsnd");
        _exit(0);
    }

    struct mymsg m;
    ssize_t n = msgrcv(qid, &m, sizeof m, 42, 0);   /* type 42 only */
    if (n > 0)
        printf("received %zd bytes: %s\n", n, m.text);

    wait(NULL);
    msgctl(qid, IPC_RMID, NULL);           /* destroy */
    return 0;
}
```

```bash
gcc -Wall -Wextra -o svmsg svmsg.c
./svmsg
ipcs -q
ipcrm -q <id>
```

## POSIX

```c
#include <fcntl.h>
#include <mqueue.h>
#include <stdio.h>
#include <string.h>
#include <sys/stat.h>

int main(void)
{
    struct mq_attr attr = { .mq_maxmsg = 10, .mq_msgsize = 256 };

    mqd_t mq = mq_open("/my_queue", O_CREAT | O_RDWR, 0600, &attr);
    if (mq == (mqd_t)-1) { perror("mq_open"); return 1; }

    const char *msg = "a POSIX message";
    if (mq_send(mq, msg, strlen(msg), 0) != 0)
        perror("mq_send");

    char buf[256];
    ssize_t n = mq_receive(mq, buf, sizeof buf, NULL);
    if (n >= 0)
        printf("received %zd bytes: %.*s\n", n, (int)n, buf);

    mq_close(mq);
    mq_unlink("/my_queue");
    return 0;
}
```

```bash
gcc -Wall -Wextra -lrt -o posixmq posixmq.c
./posixmq
```

| | System V | POSIX |
|---|---|---|
| Name | numeric key from `ftok()` | a path in the filesystem, `/name` |
| Handles | integer id | `mqd_t` descriptor |
| Max message size | fixed, 1–8192 bytes | configurable per queue |
| Up to | 65,536 messages | `mq_maxmsg`, max 10 per POSIX |
| Cleanup | manual: `ipcrm` or `IPC_RMID` | manual: `mq_unlink` |
| Priority | yes, per message | yes, per message |

> [!NOTE]
> POSIX message queues are not persistent across reboots on most Linux
> filesystems, and (as of Linux 2.6.30) they are implemented on top of
> `mqueuefs`, not System V message queues.

---

# Shared Memory

## Definition

**Shared memory** maps the *same physical pages* into the address space of two
or more processes. After that, no kernel involvement is needed — both
processes read and write ordinary memory.

```text
  Process A                    Process B
  ┌──────────────┐            ┌──────────────┐
  │ address space│            │ address space│
  │              │            │              │
  │   0x7f0000───┼────────────┼──► shared    │
  │   (mapped)   │  same      │   page       │
  │              │  physical  │              │
  │   0x400000───┼──── page ──┼──── 0x900000 │
  │   (normal)   │   table    │   (mapped)   │
  └──────────────┘            └──────────────┘
```

## The critical insight

```text
  Shared memory gives you SHARED DATA.
  It gives you NO SYNCHRONISATION.

  You now have two processes writing the same bytes with no lock, no
  ordering, and no guarantee. This is the fastest way to create a race
  condition in a program.

  Shared memory is always paired with a synchronisation mechanism:
      a semaphore, a mutex in shared memory, or eventfd().
```

## A working example with a POSIX semaphore

```c
#include <fcntl.h>
#include <semaphore.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/mman.h>
#include <sys/stat.h>
#include <sys/wait.h>
#include <unistd.h>

#define REGION_SIZE 4096

struct region {
    int          value;              /* the shared data */
    int          ready;              /* the shared flag  */
    sem_t        lock;               /* the shared lock  */
};

int main(void)
{
    /* Create a memory-backed file object */
    int fd = shm_open("/myregion", O_CREAT | O_RDWR, 0600);
    if (fd < 0) { perror("shm_open"); return 1; }

    if (ftruncate(fd, sizeof(struct region)) < 0) {
        perror("ftruncate");
        return 1;
    }

    struct region *r = mmap(NULL, sizeof *r, PROT_READ | PROT_WRITE,
                             MAP_SHARED, fd, 0);
    if (r == MAP_FAILED) { perror("mmap"); return 1; }

    if (sem_init(&r->lock, 1, 1) < 0) { perror("sem_init"); return 1; }
    r->value = 0;
    r->ready = 0;

    pid_t pid = fork();
    if (pid == 0) {
        /* WRITER */
        for (int i = 1; i <= 10; i++) {
            sem_wait(&r->lock);
            r->value = i * i;
            sem_post(&r->lock);
            usleep(50000);
        }
        sem_wait(&r->lock);
        r->ready = 1;                    /* "I am finished" */
        sem_post(&r->lock);
        _exit(0);
    }

    /* READER */
    for (;;) {
        sem_wait(&r->lock);
        int v = r->value, done = r->ready;
        sem_post(&r->lock);

        if (v) printf("read value = %d\n", v);
        if (done) break;
        usleep(20000);
    }

    wait(NULL);
    sem_destroy(&r->lock);
    munmap(r, sizeof *r);
    shm_unlink("/myregion");
    close(fd);
    return 0;
}
```

```bash
gcc -Wall -Wextra -lrt -pthread -o shmdemo shmdemo.c
./shmdemo
```

## Without a lock: the lost update, live

```c
/* Remove the sem_wait/sem_post and run it 1000 times. */
#include <fcntl.h>
#include <stdio.h>
#include <sys/mman.h>
#include <sys/wait.h>
#include <unistd.h>

int main(void)
{
    int fd = shm_open("/counter", O_CREAT | O_RDWR, 0600);
    ftruncate(fd, 4096);
    long *counter = mmap(NULL, 4096, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
    *counter = 0;

    for (int i = 0; i < 20; i++) {
        pid_t p = fork();
        if (p == 0) {
            for (int j = 0; j < 10000; j++)
                (*counter)++;             /*  ← not atomic, not protected */
            _exit(0);
        }
    }
    wait(NULL);

    printf("expected 200000, got %ld\n", *counter);
    shm_unlink("/counter");
    return 0;
}
```

```bash
gcc -Wall -Wextra -lrt -o lost lost.c
for i in 1 2 3; do ./lost; done
```

```text
expected 200000, got 142883
expected 200000, got 189204
expected 200000, got 200000      ← the one that got lucky
```

> [!TIP]
> If the count is sometimes exactly right, that is not good news. The race is
> still there; you just got a lucky interleaving. Run it a thousand times.

## System V shared memory

```c
#include <sys/ipc.h>
#include <sys/shm.h>

int shmid = shmget(IPC_PRIVATE, 4096, IPC_CREAT | 0600);
long *counter = shmat(shmid, NULL, 0);     /* attach */
shmctl(shmid, IPC_RMID, NULL);            /* mark for destruction NOW */
                                         /* it lives until the last detaches */
shmdt(counter);                          /* detach */
```

`IPC_RMID` immediately after `shmat` is a well-known idiom: the segment is
marked for destruction, but the mapping stays valid until every process
detaches. There is no leak, even on a crash.

---

# Signals as IPC

Signals carry almost no data. They carry an *event*.

```text
  ┌───────────────┐         ┌─────────────────┐
  │ Sender        │  kill() │ Kernel          │
  │               │ ──────► │                 │
  │               │         │ pending bitmask │
  │               │ ◄────── │ per destination │
  └───────────────┘  signal │                 │
                          └─────────────────┘
```

| Property | Value |
|---|---|
| Payload | none (real-time signals carry up to 32 bytes) |
| Queueing | standard signals coalesce; RT signals queue |
| Ordering | not guaranteed for standard signals |
| Blocking | no |
| Sync possible | only with a signalfd or self-pipe |

## C example

```c
#include <signal.h>
#include <stdio.h>
#include <unistd.h>

static volatile sig_atomic_t got = 0;

static void handler(int sig) { (void)sig; got = 1; }

int main(void)
{
    struct sigaction sa = {0};
    sa.sa_handler = handler;
    sigemptyset(&sa.sa_mask);
    sa.sa_flags = SA_RESTART;
    sigaction(SIGUSR1, &sa, NULL);

    printf("pid=%d, waiting for SIGUSR1\n", (int)getpid());
    fflush(stdout);

    while (!got)
        pause();                 /* sleep until a signal arrives */

    puts("got it");
    return 0;
}
```

```bash
gcc -Wall -Wextra -o sigipc sigipc.c
./sigipc &
kill -USR1 %1
```

## Self-pipe: the correct way to combine signals with a selector

```c
#include <fcntl.h>
#include <poll.h>
#include <signal.h>
#include <unistd.h>
#include <string.h>
#include <errno.h>
#include <stdio.h>

static int self_pipe[2] = { -1, -1 };

/* async-signal-safe: only write(2) */
static void handler(int sig)
{
    int saved = errno;
    char c = (char)sig;
    ssize_t r = write(self_pipe[1], &c, 1);
    (void)r;
    errno = saved;
}

int main(void)
{
    if (pipe(self_pipe) == -1) return 1;
    fcntl(self_pipe[0], F_SETFL, O_NONBLOCK);
    fcntl(self_pipe[1], F_SETFL, O_NONBLOCK);

    struct sigaction sa = {0};
    sa.sa_handler = handler;
    sigemptyset(&sa.sa_mask);
    sa.sa_flags = SA_RESTART;
    sigaction(SIGINT,  &sa, NULL);
    sigaction(SIGTERM, &sa, NULL);

    for (;;) {
        struct pollfd pfd = { .fd = self_pipe[0], .events = POLLIN };
        int n = poll(&pfd, 1, -1);
        if (n < 0 && errno == EINTR) continue;

        char c;
        while (read(self_pipe[0], &c, 1) == 1)
            printf("handled signal %d (signal %d)\n", (int)c, SIGINT);

        break;
    }
    return 0;
}
```

```bash
gcc -Wall -Wextra -o selfpipe selfpipe.c
./selfpipe
^Chandled signal 2 (signal 2)
```

> [!IMPORTANT]
> A signal handler that does anything beyond setting a flag, or writing to a
> self-pipe, is a bug. `printf`, `malloc`, `pthread_mutex_lock` and friends are
> not async-signal-safe.

---

# Unix Domain Sockets

A socket that speaks the socket API but connects over a **file**, not the
network.

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/socket.h>
#include <sys/un.h>
#include <unistd.h>

#define SOCK_PATH "/tmp/demo.sock"

int main(void)
{
    unlink(SOCK_PATH);

    int sfd = socket(AF_UNIX, SOCK_STREAM, 0);
    if (sfd < 0) { perror("socket"); return 1; }

    struct sockaddr_un addr = { .sun_family = AF_UNIX };
    strncpy(addr.sun_path, SOCK_PATH, sizeof addr.sun_path - 1);

    if (bind(sfd, (struct sockaddr *)&addr, sizeof addr) < 0) {
        perror("bind");
        return 1;
    }
    if (listen(sfd, 5) < 0) { perror("listen"); return 1; }

    if (fork() == 0) {
        /* CLIENT */
        int cfd = socket(AF_UNIX, SOCK_STREAM, 0);
        if (connect(cfd, (struct sockaddr *)&addr, sizeof addr) < 0) {
            perror("connect");
            _exit(1);
        }
        write(cfd, "ping\n", 5);
        char buf[16];
        ssize_t n = read(cfd, buf, sizeof buf - 1);
        if (n > 0) { buf[n] = '\0'; printf("client got: %s", buf); }
        close(cfd);
        _exit(0);
    }

    /* SERVER */
    int cfd = accept(sfd, NULL, NULL);
    char buf[64];
    ssize_t n = read(cfd, buf, sizeof buf - 1);
    if (n > 0) {
        buf[n] = '\0';
        printf("server got: %s", buf);
        write(cfd, "pong\n", 5);
    }
    close(cfd);
    close(sfd);
    unlink(SOCK_PATH);
    wait(NULL);
    return 0;
}
```

```bash
gcc -Wall -Wextra -o usock usock.c
./usock
ss -xl | grep demo.sock
```

```bash
# Inspect it while running
ss -xlp
lsof -U | grep demo.sock
```

---

# Choosing an IPC Mechanism

| Requirement | Use |
|---|---|
| Parent → child, one direction, simple | `pipe()` |
| Two unrelated processes, byte stream | FIFO or Unix domain socket |
| Preserve message boundaries | message queue, or `SOCK_SEQPACKET` |
| Move large amounts of data quickly | shared memory **+** a lock |
| Count events / mutual exclusion | semaphore |
| Lock a data file | `flock()` or `fcntl(F_SETLK)` |
| Tell a process an event happened | signal, or `signalfd` |
| Combine IPC with `select`/`poll`/`epoll` | socket, `signalfd`, `eventfd`, `timerfd` |
| Work across machines | TCP or UDP |

## The selection heuristic

```text
  Do you need to move more than ~64 KiB?
        │
        ├── NO  → a pipe or socket is simpler and safer
        │
        └── YES → shared memory
                    │
                    └── you MUST now add synchronisation yourself
```

The crossover point where shared memory wins is workload-dependent. Below a few
hundred kilobytes per message, the copy is cheap and the complexity is not
worth it.

---

# Debugging IPC

```bash
# Which shared memory segments, queues, semaphores exist?
ipcs -a

# Read a System V message queue's state
ipcs -q -i <id>
cat /proc/sysvipc/msg

# Which processes hold a lock?
fuser -v /var/lock/mylock
lsof /var/lock/mylock

# What shared memory is mapped?
ls -l /proc/*/maps 2>/dev/null | grep -i shm
cat /proc/sysvipc/shm

# Trace the syscalls a program makes
strace -f -e trace=pipe2,read,write,msgsnd,msgrcv,shmget,shmat ./program

# See the byte flow
strace -s 200 -e trace=read,write -f ./program
```

```bash
# /dev/shm holds POSIX shared memory and message queues
ls -l /dev/shm/
```

> [!TIP]
> IPC bugs are usually **ordering** bugs, not data bugs. A shared memory
> program that works on your machine and fails on a colleague's usually has a
> missing memory barrier. Check your `memory_order` arguments, not your
> arithmetic.

---

# Common Misconceptions

### ❌ "Shared memory is faster because it skips the kernel."

Half correct.

It skips the **data copy**, but it does not skip the kernel — the first
mapping needs `shm_open()` and `mmap()`, which are system calls. Shared memory
wins when the transfer is large and frequent, not because the kernel is
bypassed.

---

### ❌ "Shared memory handles synchronisation for you."

Incorrect, and this is the most dangerous misconception in IPC.

Shared memory provides no ordering, no mutual exclusion, and no visibility
guarantee. You must add a semaphore, a shared mutex, or explicit atomics. This
is what the POSIX shared memory example with a `sem_t` inside the region is
demonstrating.

---

### ❌ "A pipe is bidirectional."

Incorrect.

A pipe is unidirectional. To get two directions, create two pipes. A Unix
domain socket is bidirectional.

---

### ❌ "Pipes preserve message boundaries."

Incorrect.

A pipe is a byte stream. Write 5 bytes then 5 bytes, and the reader may see 10
bytes at once, or 3 then 7, or 1 ten times. Use a length prefix, a delimiter,
or `SOCK_SEQPACKET` if boundaries matter.

---

### ❌ "System V IPC is a different system from POSIX IPC."

Incorrect.

Both are implemented by the same kernel, on Linux. The System V interfaces
(`ftok`, numeric ids, `ipcs`/`ipcrm`) predate the POSIX ones
(`shm_open`, names, `mq_open`). They are two APIs to similar facilities.

---

### ❌ "Signals can carry data."

Incorrect.

Standard signals carry no payload. Real-time signals (`SIGRTMIN` upward) carry
a 32-bit value via `sigqueue()`/`sigwaitinfo()`, and nothing more. For data,
use a pipe or a socket.

---

# Interview Questions

### Basic

- What is IPC? Why is it needed?
- List the IPC mechanisms provided by Linux.
- What is a pipe? What are its properties?
- What is the difference between a pipe and a FIFO?

### Intermediate

- Explain shared memory. Why does it need separate synchronisation?
- Differentiate between System V and POSIX message queues.
- When would you use a Unix domain socket instead of a FIFO?
- What is `SIGPIPE`, and how should a program handle it?
- Compare message queues and pipes in terms of message boundaries.

### Advanced

- Explain the self-pipe pattern. Why is a handler that calls `printf()` a bug?
- How do you pass a file descriptor to another process?
- Why is shared memory faster, and at what workload does that stop being true?
- How would you debug a program that works with one process but hangs with two?
- What is a thundering herd, and how do you prevent it with IPC?

---

# University Exam Notes

### Definitions

- **IPC:** Operating system mechanisms that allow processes to exchange data
  and coordinate actions while retaining memory isolation.
- **Pipe:** A unidirectional, byte-oriented kernel buffer with a read end and
  a write end, created by `pipe()`.
- **FIFO (named pipe):** A pipe identified by a path in the filesystem, used
  by processes that are not parent and child.
- **Message queue:** An IPC structure that preserves message boundaries and
  optionally delivers messages in priority order.
- **Shared memory:** Memory mapped into the address spaces of multiple
  processes, allowing them to read and write the same physical pages.
- **Semaphore (as IPC):** A synchronisation object shared between processes to
  control access to shared resources.

### Frequently Asked Questions

- What is IPC? Explain its need.
- List and explain the different IPC mechanisms in Linux.
- What is a pipe? What are its characteristics?
- Differentiate between named pipes and anonymous pipes.
- Explain shared memory and why it requires separate synchronisation.
- Differentiate between System V and POSIX IPC.
- What is a signal? How is it used for IPC?
- What is `SIGPIPE` and how do you handle it?

---

# Key Takeaways

- IPC solves the tension between **isolation** and **cooperation**.
- Linux mechanisms: pipes, FIFOs, message queues, shared memory, signals,
  Unix domain sockets, semaphores, `eventfd`/`signalfd`/`timerfd`.
- Pipes and FIFOs are unidirectional byte streams with no message boundaries.
- Message queues preserve boundaries; shared memory preserves nothing.
- **Shared memory gives you data but no synchronisation** — you must add a
  lock, and forgetting this is the classic bug.
- System V uses `ftok()` keys and integer ids; POSIX uses filesystem names.
- Signals carry events, not data. Keep handlers to a flag set or a self-pipe
  write.
- Inspect state with `ipcs -a`, `/dev/shm`, `/proc/sysvipc/`, and `strace`.

---

# References

- Operating System Concepts — Silberschatz
- The Linux Programming Interface — Kerrisk
- The Art of Unix Programming — Stevens and Rago
- `man 2 pipe`, `man 2 pipe2`, `man 2 mkfifo`, `man 3 mq_*`
- `man 2 shm_open`, `man 2 msgsnd`, `man 2 msgsnd`, `man 2 kill`
- `man 7 unix`, `man 7 signal`, `ipcs(1)`, `lsipc(1)`
- Stevens & Rago, *UNIX Network Programming, Vol. 2*
