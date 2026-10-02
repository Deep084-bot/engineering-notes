# System Calls in Detail

> [!NOTE]
> **Module:** Module I – Introduction
>
> **Difficulty:** ⭐⭐⭐⭐☆
>
> **Prerequisites:**
> - Modes of Operation (cpu-modes.md)
> - Interrupts and I/O (interrupts.md)
> - OS Services and Functions
>
> **Scope:** Linux and C only. No Windows or Java.

---

# Learning Objective

After reading this note, you should be able to:

- Define a system call precisely and explain why it exists.
- Explain the user mode → kernel mode transition at the hardware level.
- Explain the difference between `syscall`, `int 0x80` and `sysenter`.
- Explain how arguments and return values cross the boundary.
- Explain the error convention: negative return values and `errno`.
- Write C code that invokes a system call directly with inline assembly.
- Explain the vDSO and why `clock_gettime()` often avoids the kernel.
- Use `strace` to observe system calls in a real program.

---

# Why Do We Need System Calls?

## The Problem

A user program wants to do something privileged: read a file, allocate a
mapping, create a process, touch a device.

The CPU is running in ring 3 (user mode).

The kernel is in ring 0 (kernel mode).

The MMU is configured so that:

- User code **cannot** read or write kernel memory.
- User code **cannot** execute privileged instructions (`hlt`, `cli`, `lidt`,
  `in`/`out` on I/O ports, `mov cr3`, ...).
- User code **cannot** configure the page tables, the IDT, or the APIC.

If any of this were permitted, a single bad program — or a malicious one —
could read another user's password, disable interrupts, or rewrite the kernel.

There must be a **narrow, controlled, enforced door** in this wall.

That door is the system call.

---

# Real World Analogy

A bank vault has a wall nobody can pass through.

But you need to deposit money.

The wall has a **teller window**: a slot with a clerk on the other side.

- You can only do the specific things on the clerk's list.
- You hand over an item through the slot.
- The clerk does the privileged thing safely.
- The clerk hands back a result and a receipt.

The slot is a **system call interface**. The vault is **kernel mode**. The bank
lobby is **user mode**.

The critical design property: the list of permitted operations is defined by
the *kernel*, not by the caller. You cannot invent a new operation by asking
loudly.

---

# Intuition

A system call is a **trap**, not a function call.

```text
  FUNCTION CALL                  SYSTEM CALL
  ─────────────                  ──────────
  push return address             push SS, RSP, RFLAGS, CS, RIP
  push callee-saved regs          push argument registers
  call target                     execute `syscall`
  ...                            ── CPU switches to ring 0 ──
  ...                            kernel handler runs
  pop regs                        kernel writes return value
  pop return address              execute `sysret`
  ret                             ── CPU switches to ring 3 ──
                                  ... resumes in user code
```

The difference is what is saved. A function call saves *code* context. A system
call saves the *entire machine context*, because the CPU will resume the user
program exactly where it left off, with everything unchanged.

---

# What a System Call Is Not

| It is not | Because |
|---|---|
| A function call | It crosses a privilege boundary, not a stack frame |
| A message to another process | It is the *same* thread, just in a different mode |
| A library function | Many library functions never make a syscall at all |
| An interrupt from a device | No device is involved; the program asks deliberately |
| Always slow | The vDSO path avoids the kernel entirely for some calls |

---

# Formal Definition

A **system call** is a program-controlled, synchronous transition from user
mode to kernel mode, through a privileged instruction, to request a service
that the kernel has explicitly published.

| Property | Value |
|---|---|
| Trigger | The program itself (`syscall` instruction) |
| Timing | Synchronous, at a known point in the instruction stream |
| Privilege change | Yes, ring 3 → ring 0 |
| Context saved | Full CPU state (RIP, CS, RFLAGS, RSP, SS, GDT base) |
| Who decides validity | The kernel, at the entry point |
| Who performs the work | The kernel |
| Return mechanism | `sysret` (x86-64), values in RAX and RDX |

---

# The Mechanism on x86-64

## 1. The `syscall` instruction

```asm
; 64-bit Linux, System V AMD64 ABI
syscall          ; traps to the kernel, does NOT change privilege by itself
                 ; -- the MSRs decide what happens
```

Two Model-Specific Registers configure the entry point:

| MSR | Purpose |
|---|---|
| `IA32_SYSENTER_CS` / `STAR` | Selects the code segment and privilege behaviour |
| `IA32_LSTAR` | The kernel entry point address |
| `IA32_SFMASK` | Which RFLAGS bits are cleared on entry (clears IF) |
| `IA32_KERNEL_GS_BASE` | Used by the kernel to find per-CPU data |

The kernel installs these at boot. Your program never touches them.

## 2. The argument register convention

System V AMD64 ABI: the first six integer/pointer arguments go in registers.

| Argument | Register | Syscall return | Register |
|---|---|---|---|
| arg 1 | `RDI` | return value | `RAX` |
| arg 2 | `RSI` | error indication | `RDX` |
| arg 3 | `RDX` | saved RFLAGS | `R11` |
| arg 4 | `R10` | saved RIP | `R11` (RIP is in RCX) |
| arg 5 | `R8` | | |
| arg 6 | `R9` | | |

`syscall` also clobbers `RCX` (holds the return RIP) and `R11` (holds saved
RFLAGS). Per the ABI, a syscall wrapper must treat `RCX` and `R11` as
destroyed.

## 3. C example: making a system call directly

```c
#include <errno.h>
#include <stdio.h>
#include <string.h>
#include <unistd.h>

#ifndef SYS_getpid
#define SYS_getpid 39
#endif
#ifndef SYS_write
#define SYS_write 1
#endif
#ifndef SYS_getppid
#define SYS_getppid 110
#endif

/*
 * Invoke the kernel directly, bypassing glibc.
 *
 * long syscall(long number, ...);
 *
 * The C `...` cannot be typed, so this relies on the ABI guaranteeing that
 * the first six integer arguments land in RDI, RSI, RDX, R10, R8, R9.
 */
static inline long raw_syscall1(long n, long a)
{
    long ret;
    __asm__ volatile (
        "syscall"
        : "=a" (ret)                 /* RAX = return value */
        : "a"  (n),                  /* RAX = syscall number */
          "D"  (a)                   /* RDI = arg1 */
        : "rcx", "r11",              /* clobbered by syscall */
          "memory"                   /* compiler barrier */
    );
    return ret;
}

static inline long raw_syscall3(long n, long a, long b, long c)
{
    long ret;
    __asm__ volatile (
        "syscall"
        : "=a" (ret)
        : "a"  (n),
          "D"  (a),
          "S"  (b),
          "d"  (c)
        : "rcx", "r11", "memory"
    );
    return ret;
}

int main(void)
{
    /* 1. SYS_getpid — no arguments, result in RAX */
    long pid = raw_syscall1(SYS_getpid, 0);
    printf("getpid()  = %ld\n", pid);

    /* 2. SYS_getppid — parent process */
    long ppid = raw_syscall1(SYS_getppid, 0);
    printf("getppid() = %ld\n", ppid);

    /* 3. SYS_write — fd=1, buf, count */
    const char msg[] = "written without glibc's write()\n";
    long n = raw_syscall3(SYS_write, 1, (long)msg, (long)(sizeof msg - 1));
    printf("write()   = %ld bytes\n", n);

    /* 4. Errors come back as a NEGATIVE errno, not a set errno */
    long bad = raw_syscall1(SYS_getpid, 0);
    if (bad < 0) {
        printf("error: %s (errno %ld)\n", strerror((int)-bad), -bad);
    }

    return 0;
}
```

```bash
gcc -Wall -Wextra -O2 -o rawsys rawsys.c
./rawsys
strace ./rawsys
```

```text
getpid()  = 4821
getppid() = 4790
write()   = 34 bytes
```

```text
# strace sees exactly the calls made, glibc or not
getpid()                          = 4821
getppid()                         = 4790
write(1, "written without glibc's write()\n", 34) = 34
+++ exited with 0 +++
```

> [!TIP]
> `strace` works precisely because it intercepts the entry to the kernel. It
> is a debugger for system calls specifically, and it works equally well on
> programs you do not have source for.

## 4. Why glibc exists if `syscall()` works

You should almost never write this by hand. glibc adds real value:

```c
/* glibc's write(), simplified */
ssize_t write(int fd, const void *buf, size_t count)
{
    long ret = syscall(SYS_write, fd, buf, count);

    /* Kernel reports errors as -1 .. -4095. glibc normalises this. */
    if (ret < 0 && ret > -4096) {
        errno = -ret;      /* convert to positive errno */
        return -1;         /* hide the detail from callers */
    }
    return ret;
}
```

glibc adds:

- **Error normalisation**: negative returns become `-1` + a positive `errno`.
- **Buffering**: `printf` and `stdout` are buffered; raw syscalls are not.
- **Portability**: one source, many architectures.
- **Cancelling and TLS**: thread-local `errno`, cancellation point handling.

---

# The Error Convention

This is the single most important practical detail.

```text
  On success:   the return value is the useful result, and errno is untouched.
  On failure:   the return value is -1 (for most calls), and errno is set
                to a positive error number.
```

```c
#include <stdio.h>
#include <string.h>
#include <errno.h>
#include <fcntl.h>

int main(void)
{
    errno = 0;                            /* clear before the call */
    FILE *f = fopen("/definitely/not/here", "r");

    if (f == NULL) {
        printf("fopen failed: %s (errno=%d)\n", strerror(errno), errno);
        /* ENOENT = 2  → "No such file or directory" */
        printf("raw: ENOENT=%d EACCES=%d EAGAIN=%d\n",
               ENOENT, EACCES, EAGAIN);
    }
    return 0;
}
```

## The raw form differs

At the `syscall()` level, the kernel returns `-errno` directly:

```c
long r = syscall(SYS_open, "/nope", O_RDONLY);
/* r == -2   (i.e. -ENOENT) */
```

| Style | Failure return | Where the code lives |
|---|---|---|
| Raw kernel | `-ENOENT` (a negative number) | the kernel |
| `syscall()` | still `-ENOENT` | glibc's `syscall()` does not translate |
| glibc wrapper | `-1`, with `errno = ENOENT` | glibc |

## errno is not a function

```c
extern int *__errno_location(void);
#define errno (*__errno_location())
```

`errno` is a **per-thread** variable, not a global. This is why:

- Two threads can have different `errno` values at the same time.
- You must save `errno` immediately if you call anything else, because that
  call may overwrite it.

```c
int fd = open("f", O_RDONLY);
if (fd < 0) {
    int saved = errno;        /* save FIRST */
    log_error(saved);
    perror("open");           /* this may clobber errno */
    return saved;
}
```

## Common errno values on Linux

| errno | Macro | Meaning |
|---|---|---|
| 1 | `EPERM` | Operation not permitted |
| 2 | `ENOENT` | No such file or directory |
| 5 | `EIO` | I/O error |
| 9 | `EBADF` | Bad file descriptor |
| 11 | `EAGAIN` / `EWOULDBLOCK` | Resource temporarily unavailable |
| 12 | `ENOMEM` | Cannot allocate memory |
| 13 | `EACCES` | Permission denied |
| 16 | `EBUSY` | Device or resource busy |
| 22 | `EINVAL` | Invalid argument |
| 32 | `EPIPE` | Broken pipe (writer closed) |
| 110 | `ETIMEDOUT` | Operation timed out |

```bash
# The whole list
man 3 errno

# Numeric values
cat /usr/include/asm-generic/errno-base.h
cat /usr/include/asm-generic/errno.h
```

---

# The `int 0x80` and `sysenter` History

| Mechanism | Architecture | Era | Status on x86-64 |
|---|---|---|---|
| `int 0x80` | i386 | 1990s | Works, but the numbers are the 32-bit table |
| `sysenter` | IA-32 / IA-64 | early 2000s | Replaced by `syscall` |
| `syscall` | x86-64 | 2003+ | **The mechanism to use** |

```asm
; 32-bit legacy
mov eax, 4          ; SYS_write in the 32-bit table
mov ebx, 1          ; fd = stdout
mov ecx, msg
mov edx, len
int 0x80
```

```c
/* 32-bit legacy, in C */
long r;
__asm__ volatile ("int $0x80"
                  : "=a"(r)
                  : "a"(4), "b"(1), "c"(msg), "d"(len)
                  : "memory");
```

> [!NOTE]
> The syscall **number** differs between 32-bit and 64-bit. `SYS_write` is
> 4 on i386 and 1 on x86-64. Always use the macro, never the literal.

---

# The vDSO: When No System Call Happens At All

Not every call to `clock_gettime()` reaches the kernel.

Linux maps a small **virtual dynamic shared object** into every process at a
fixed address:

```bash
# The vDSO is mapped in your address space
grep vdso /proc/self/maps
```

```text
7f2c1a4b8000-7f2c1a4b9000 r-xp 00000000 00:00 0                  [vdso]
```

```text
  Why does this exist?

  clock_gettime() is called by anything that measures elapsed time:
  profilers, databases, network stacks, games, JIT runtimes.

  The vDSO reads the TSC (Time Stamp Counter) directly from the CPU.
  No trap, no context switch, roughly 20 nanoseconds instead of ~500.
```

```c
#define _GNU_SOURCE
#include <stdio.h>
#include <time.h>
#include <sys/syscall.h>
#include <unistd.h>

int main(void)
{
    struct timespec a, b;

    /* 1. libc version — may or may not use the vDSO */
    clock_gettime(CLOCK_MONOTONIC, &a);
    clock_gettime(CLOCK_MONOTONIC, &b);
    long v1 = (b.tv_sec - a.tv_sec) * 1000000000L + (b.tv_nsec - a.tv_nsec);

    /* 2. forced raw syscall — guaranteed to enter the kernel */
    syscall(SYS_clock_gettime, CLOCK_MONOTONIC, &a);
    syscall(SYS_clock_gettime, CLOCK_MONOTONIC, &b);
    long v2 = (b.tv_sec - a.tv_sec) * 1000000000L + (b.tv_nsec - a.tv_nsec);

    printf("libc  : %ld ns for 2 calls\n", v1);
    printf("raw   : %ld ns for 2 calls\n", v2);
    printf("vDSO  : %s\n",
           syscall(SYS_getauxval, AT_SYSINFO_EHDR) ? "present" : "absent");
    return 0;
}
```

```bash
gcc -O2 -o vdso vdso.c
./vdso
strace -e trace=clock_gettime ./vdso
```

```text
libc  : 46 ns for 2 calls
raw   : 612 ns for 2 calls
vDSO  : present
```

If `strace` shows **no** `clock_gettime` for the libc calls but does for the
`syscall()` calls, the vDSO is working exactly as intended.

## The general pattern

```
  glibc wrapper
       │
       ├── checks if the kernel supports a fast path
       │     └── if yes: read a register / memory-mapped page  (no trap)
       │
       └── if no: issue the syscall instruction                (trap)
```

This is why the cost of a system call varies so much: 20 ns for a vDSO hit,
roughly 50–500 ns for a real trap, and much more if the operation actually
has to touch a disk.

---

# A Complete Tour of System Calls in C

```c
#define _GNU_SOURCE
#include <fcntl.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/mman.h>
#include <sys/stat.h>
#include <sys/syscall.h>
#include <sys/wait.h>
#include <unistd.h>

int main(void)
{
    /* --- FILE SYSTEM: open / fstat / read / close --- */
    int fd = open("/etc/hostname", O_RDONLY);
    if (fd < 0) { perror("open"); return 1; }

    struct stat st;
    if (fstat(fd, &st) == 0) {
        printf("size=%ld inode=%lu\n", (long)st.st_size, (unsigned long)st.st_ino);
    }

    char buf[64];
    ssize_t n = read(fd, buf, sizeof buf - 1);
    if (n > 0) {
        buf[n] = '\0';
        printf("hostname=%s", buf);
    }
    close(fd);

    /* --- MEMORY: mmap --- */
    void *p = mmap(NULL, 4096, PROT_READ | PROT_WRITE,
                   MAP_PRIVATE | MAP_ANONYMOUS, -1, 0);
    if (p == MAP_FAILED) { perror("mmap"); return 1; }
    memset(p, 'A', 4096);
    printf("first byte = %c\n", ((char *)p)[0]);
    munmap(p, 4096);

    /* --- PROCESS: getpid / fork / execve / wait4 --- */
    printf("my pid = %d\n", (int)getpid());

    pid_t pid = fork();
    if (pid == 0) {
        char *const argv[] = { "/bin/echo", "child ran", NULL };
        char *const envp[] = { "PATH=/usr/bin:/bin", NULL };
        execve("/bin/echo", argv, envp);
        _exit(127);                    /* only reached if execve failed */
    }
    int status;
    waitpid(pid, &status, 0);
    printf("child exited with %d\n", WEXITSTATUS(status));

    /* --- THREADS: the same clone() underneath, different flags --- */
    long tid = syscall(SYS_gettid);
    printf("my tid = %ld (equals pid for the main thread)\n", tid);

    /* --- RAW: bypass glibc entirely --- */
    long ppid = syscall(SYS_getppid);
    printf("raw getppid = %ld\n", ppid);

    /* --- SIGNAL: install a handler --- */
    /* see interrupts.md for a full example */

    return 0;
}
```

```bash
gcc -Wall -Wextra -O2 -o tour tour.c
./tour
strace -f -c ./tour
```

---

# Observing System Calls

## strace

```bash
# Trace everything
strace ./program

# Trace only specific calls
strace -e trace=openat,read,write ./program

# Follow children
strace -f ./program

# Print a summary with timing
strace -c -f ./program

# Print call arguments and returns
strace -v ./program

# Attach to a running process
strace -p 12345

# Print the raw system call numbers
strace -X raw ./program

# Filter by result
strace -e trace=openat -e status=successful ./program
```

```text
# Output
openat(AT_FDCWD, "/etc/hostname", O_RDONLY|O_CLOEXEC) = 3
fstat(3, {st_dev=..., st_ino=1058577, st_mode=S_IFREG|0444, st_nlink=1,
          st_uid=0, st_gid=0, st_size=12, ...}) = 0
newfstatat(3, "", {st_dev=..., st_ino=..., st_mode=S_IFREG|0444, ...},
           AT_EMPTY_PATH) = 0
read(3, "my-laptop\n", 64)              = 10
close(3)                                = 0
mmap(NULL, 4096, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f...
munmap(0x7f..., 4096)                   = 0
exit_group(0)                           = 0
+++ exited with 0 +++
```

## ltrace — library calls

```bash
ltrace -e malloc ./program     # library function calls
ltrace -S ./program            # include system calls too
```

## perf — statistical, low overhead

```bash
perf stat -e syscalls:sys_enter_openat ./program
perf trace ./program
perf record -g ./program && perf report
```

`perf` samples rather than tracing, so it does not slow the program down. Use
`strace` when you need exact behaviour, `perf` when you need speed.

## gdb — stepping into the kernel

```bash
# Break on a syscall return
gdb --args ./program
(gdb) catch syscall openat
```

---

# Choosing the Right System Call

| Need | Use | Not |
|---|---|---|
| Read a file | `read()` on an fd | `mmap()` unless you need random access |
| Random access to a large file | `mmap()` | repeated `lseek()` + `read()` |
| Create a thread | `pthread_create()` | raw `clone()` |
| Full control of a process | `clone()` with flags | `fork()` + `execve()` |
| Start a program | `fork()` + `execve()`, or `posix_spawn()` | `system()` in a multithreaded program |
| Wall-clock time | `clock_gettime(CLOCK_REALTIME)` | `time()` |
| Elapsed time | `clock_gettime(CLOCK_MONOTONIC)` | `CLOCK_REALTIME` for intervals |
| A short wait | `nanosleep()` | `usleep()` (obsolete) |
| Check a lock without blocking | `futex` with `FUTEX_WAIT` private | `sem_trywait()` in a hot loop |

```c
/* Never measure an interval with CLOCK_REALTIME: NTP can step it backwards. */
struct timespec t0, t1;
clock_gettime(CLOCK_MONOTONIC, &t0);
/* ... work ... */
clock_gettime(CLOCK_MONOTONIC, &t1);
double secs = (t1.tv_sec - t0.tv_sec) + (t1.tv_nsec - t0.tv_nsec) / 1e9;
```

---

# Common Misconceptions

### ❌ "A system call is slow, so avoid them."

Incorrect for correctness-critical operations, and wrong for `errno`. But
`printf` doing a `write` per call *is* genuinely slow — because of buffering,
not because of the syscall. Use buffered I/O, not fewer calls, to fix it.

```c
/* Slow: one write syscall per number */
for (int i = 0; i < 10000; i++)
    printf("%d\n", i);

/* Fast: one write syscall total */
for (int i = 0; i < 10000; i++)
    printf("%d\n", i);   /* stdout is fully buffered when not a tty */

/* Fastest: one write syscall, explicitly */
char *big = malloc(1 << 20);
int len = snprintf(big, 1 << 20, ...);
write(1, big, len);
```

---

### ❌ "errno tells you why a call failed."

Almost certainly wrong.

`errno` is only meaningful if the call *actually* reported failure. It is a
sticky value that persists until your next call changes it. Check the return
value, not `errno`.

```c
/* Wrong */
close(fd);
if (errno == EBADF) { ... }   /* errno is left over from anything */

/* Right */
if (close(fd) == -1) {
    int e = errno;
}
```

---

### ❌ "`syscall()` and calling the libc function are the same thing."

Incorrect.

`syscall(SYS_read, ...)` is a raw trap. `read()` is a glibc wrapper that
translates `-errno` into `-1` + `errno`, and in some cases does extra work.
Different semantics, and only the wrapper version sets `errno`.

---

### ❌ "All system calls cost the same."

Incorrect.

Costs range from ~20 ns (vDSO hit) to ~200 ns (a simple trap) to milliseconds
(a blocking read on a cold disk). The spread is why blocking calls are
essential and why spinning is usually the wrong answer.

---

### ❌ "System calls can be made from user mode in C without any special mechanism."

Incorrect.

C has no way to express "trap into the kernel". You need either the libc
wrapper or inline assembly with the `syscall` instruction. The compiler
cannot synthesise a privilege transition.

---

# Interview Questions

### Basic

- What is a system call? Why is it needed?
- Explain the user mode and kernel mode transition during a system call.
- What is the role of `errno`?
- Differentiate between a system call and a library function.

### Intermediate

- Explain the x86-64 `syscall` instruction: which registers hold arguments, and which are clobbered?
- Why does the kernel return `-errno` instead of setting `errno` itself?
- What is the vDSO, and which system calls use it?
- Explain `int 0x80` vs `syscall`.
- What is the difference between `read()` and `mmap()` for accessing a file?

### Advanced

- Why is `errno` per-thread rather than per-process?
- How would you add a new system call to the Linux kernel, and what would you need to update?
- How does `strace` work, given that it observes system calls without modifying the target program?
- Why does `fork()` in a multithreaded program only copy the calling thread?
- When is a raw `syscall()` appropriate, and when is it a mistake?
- How would you measure the true cost of a system call in your program, and why might your measurement be wrong?

---

# University Exam Notes

### Definitions

- **System call:** A controlled, synchronous entry from user mode into kernel
  mode through a privileged instruction, used to request a kernel service.
- **Trap:** A deliberate software-generated entry into the kernel.
- **Trap gate / interrupt gate:** IDT entry types; the interrupt gate disables
  further interrupts on entry.
- **errno:** A per-thread integer holding the error number from the most recent
  failed library call.
- **vDSO:** A virtual shared object mapped into every process that lets some
  libc functions be served without entering the kernel.
- **Starvation of syscalls:** Spurious wakeups; `read()` may return `-1` with
  `EAGAIN` on a non-blocking fd, and may return `-1` with `EINTR` if a signal
  arrives.

### Frequently Asked Questions

- What is a system call? Explain with a diagram.
- Explain the steps of a system call on x86-64.
- What are interrupts, exceptions and traps? How do they differ?
- What is the difference between a library function and a system call?
- Explain the vDSO and why it exists.
- How does the kernel return an error to user space?

---

# Key Takeaways

- A system call is a **trap**, not a function call: it saves the full machine
  context and changes privilege.
- On x86-64, `syscall` passes arguments in `RDI, RSI, RDX, R10, R8, R9`,
  returns the value in `RAX`, and clobbers `RCX` and `R11`.
- The **kernel** returns `-errno`. **glibc** translates that to `-1` plus a
  positive `errno`. Always check the return value first.
- `errno` is per-thread and sticky. Save it immediately on failure.
- `int 0x80` is the legacy 32-bit mechanism; `syscall` is the x86-64 one.
- The **vDSO** lets `clock_gettime()` read the CPU timestamp counter directly,
  making it ~20 ns instead of ~500 ns. This is why "system call cost" is not
  one number.
- `strace -c` is the first tool to reach for when a program misbehaves.

---

# References

- Operating System Concepts — Silberschatz
- The Linux Programming Interface — Kerrisk
- Intel 64 and IA-32 Architectures Software Developer's Manual, Vol. 2B (`SYSCALL`)
- System V Application Binary Interface, AMD64 Architecture Processor Supplement
- `man 2 syscall`, `man 2 syscalls`, `man 3 errno`, `man 2 intro`
- `strace(1)` man page — `-e`, `-f`, `-c`, `-v`, `-X raw` options
