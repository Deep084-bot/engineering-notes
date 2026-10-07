# Operating System Services and Functions

> [!NOTE]
> **Module:** Module I – Introduction
>
> **Difficulty:** ⭐⭐☆☆☆
>
> **Prerequisites:**
> - What is an Operating System
> - Kernel

---

# Learning Objective

After reading this note, you should be able to:

- Describe the major services every operating system provides.
- Explain how system programs differ from user programs.
- Understand why system calls are the only way to request kernel services.
- Explain the role of the system call interface.
- Describe what utility programs such as the shell do.
- List the core Linux interfaces that provide each service.

---

# Why Do We Need OS Services?

## The Problem

Consider a program that reads a file.

At the hardware level, reading a file means sending a command to a disk controller, waiting for the platter to spin, transferring bytes over PCIe, and receiving them.

But the program author does not want to:

- Know which disk the file is on
- Know the sector layout of the filesystem
- Wait in a spin loop burning CPU
- Share the disk safely with other programs

Every program would need to reimplement the same low-level logic.

That is obviously wrong.

The operating system should provide these capabilities **once, correctly, and safely**, and let programs request them.

---

# Real World Analogy

A hotel does not expect every guest to generate their own electricity.

The hotel provides:

| Service | What the guest gets |
|---|---|
| Electricity | A socket in the room |
| Water | Taps and hot water |
| Internet | A network port |
| Security | Locked doors, staff |
| Housekeeping | Cleaned rooms |

A guest plugs in and uses the service without knowing the generator model or the water treatment plant.

The operating system is that hotel, and system calls are the wall sockets.

---

# Intuition

An operating system is fundamentally a **provider of abstractions**.

The raw hardware is powerful but unusable directly.

The OS wraps it in services:

```
   User program wants to read a file
                │
                ▼
        read(fd, buf, n)            ← system call
                │
                ▼
   ┌─────────────────────────────────────────┐
   │              KERNEL                      │
   │  VFS → filesystem → page cache → block   │
   │       layer → I/O scheduler → driver     │
   │       → disk controller → hardware        │
   └─────────────────────────────────────────┘
                │
                ▼
        Bytes appear in buf[]
```

The program never touches the disk controller.

---

# The Two Categories of OS Code

## User Programs

Programs that run in **user mode**.

- Cannot access kernel memory
- Cannot execute privileged instructions
- Can only request kernel services via system calls

Examples: your C program, `vim`, `firefox`, `bash`.

## System Programs

Programs that are part of the OS, shipped by the OS vendor.

- **System utilities**: `ls`, `cp`, `ps`, `top`
- **Program development tools**: `gcc`, `make`, `gdb`
- **System administration**: `systemctl`, `ip`, `mount`

The OS is conventionally defined as:

> Kernel + system libraries + system utilities + system programs

---

# System Calls: The Fundamental Service Interface

## Definition

A **system call** is a controlled entry point from user mode into kernel mode.

It is the mechanism by which a user program asks the kernel to perform a privileged action on its behalf.

## Why a System Call Is Necessary

The CPU has two privilege levels:

- **User mode**: cannot touch kernel memory, cannot run privileged instructions
- **Kernel mode**: full access to hardware and memory

Without hardware protection, any program could read another program's memory, disable interrupts, or write directly to disk.

The system call is the hardware-enforced door between the two.

```
   User mode                          Kernel mode
   ─────────                          ───────────
   your code
       │  syscall instruction  ──────▶  kernel entry
       │                                    │
       │  ◀──────  return + errno  ─────────│
   your code continues
```

The C library (`glibc`) wraps most system calls:

```c
#include <unistd.h>
ssize_t read(int fd, void *buf, size_t count);
```

When you call `read()`, `glibc` eventually executes a `syscall` instruction.

> [!NOTE]
> The system call interface is the API of the operating system.
> `read`, `write`, `open`, `fork`, `execve`, `mmap` are the API.
> The hardware mechanism is a trap instruction, not a function call.

---

# The Major OS Services

## 1. Process Management / Process Control

Creating, terminating, and controlling processes.

| Service | Linux system calls |
|---|---|
| Create a process | `fork()`, `clone()`, `vfork()`, `posix_spawn()` |
| Replace program image | `execve()`, `execv()`, `execl()` |
| Wait for child | `wait()`, `waitpid()`, `waitid()` |
| Terminate | `exit()`, `_exit()`, `kill()`, `raise()` |
| Get process ID | `getpid()`, `getppid()`, `gettid()` |
| Change scheduling priority | `setpriority()`, `nice()`, `sched_yield()` |

Example:

```c
#include <stdio.h>
#include <unistd.h>
#include <sys/wait.h>

int main(void)
{
    pid_t pid = fork();

    if (pid == 0) {
        /* Child process */
        puts("child");
        _exit(0);
    }

    waitpid(pid, NULL, 0);
    printf("parent reaped child %d\n", pid);
    return 0;
}
```

---

## 2. File System Management

Managing files, directories, and their metadata.

| Service | Linux system calls |
|---|---|
| Open / close | `open()`, `openat()`, `close()` |
| Read / write | `read()`, `write()`, `pread()`, `pwrite()` |
| Metadata | `stat()`, `fstat()`, `lstat()` |
| Change permissions | `chmod()`, `chown()` |
| Link / unlink | `link()`, `unlink()`, `rename()` |
| List directory | `getdents64()` (glibc wraps as `readdir()`) |
| Mount filesystems | `mount()`, `umount()` |

> [!TIP]
> On Linux, the VFS (Virtual File System) gives every filesystem — ext4,
> XFS, btrfs, tmpfs, procfs, overlayfs — the same `open`/`read`/`write`
> interface. The kernel can switch filesystems without the program noticing.

```c
#include <fcntl.h>
#include <unistd.h>

int fd = open("config.txt", O_RDONLY);
char buf[256];
ssize_t n = read(fd, buf, sizeof buf);
close(fd);
```

---

## 3. Memory Management

Allocating, protecting, and mapping memory.

| Service | Linux system calls |
|---|---|
| Map anonymous memory | `mmap()`, `munmap()`, `brk()`, `sbrk()` |
| Shared memory mapping | `shmget()` (System V), `shm_open()` (POSIX) |
| Change protection | `mprotect()` |
| Advice (prefetch, huge pages) | `madvise()` |
| Query memory info | `sysinfo()`, `/proc/meminfo` |

```c
#include <sys/mman.h>
#include <stdlib.h>

void *p = mmap(NULL, 4096,
               PROT_READ | PROT_WRITE,
               MAP_PRIVATE | MAP_ANONYMOUS,
               -1, 0);

if (p == MAP_FAILED) {
    perror("mmap");
    return 1;
}
/* p now points at a writable page that was never backed by a file */
munmap(p, 4096);
```

---

## 4. Device Management

Interacting with hardware devices.

| Service | Linux system calls / interfaces |
|---|---|
| Device I/O | `read()`, `write()`, `ioctl()` |
| Direct memory access | `mmap()` on device files, `/dev/mem` (privileged) |
| Device discovery | `mmap()`, `/sys/class/*`, `/dev` |
| Interrupt registration | `request_irq()` (kernel-side) |

Every device on Linux is exposed as a **device file** under `/dev`.

```bash
ls -l /dev/sda
crw-rw---- 1 root disk 8, 0 Sep 29 10:00 /dev/sda
```

- `c` in the mode means **character device**
- `8, 0` is the major/minor pair used to route to the correct driver

---

## 5. Communication and Synchronisation

Exchanging data between processes and threads.

| Service | Linux mechanisms |
|---|---|
| Pipes | `pipe()`, `pipe2()` |
| Named pipes (FIFOs) | `mkfifo()` |
| Message queues | `msgget()`, `msgsnd()` (System V), `mq_open()` (POSIX) |
| Shared memory | `shmget()`, `shm_open()` + `mmap()` |
| Signals | `sigaction()`, `kill()`, `sigprocmask()` |
| Sockets | `socket()`, `bind()`, `connect()`, `send()`, `recv()` |
| Thread synchronisation | `pthread_mutex_*`, `sem_*`, `pthread_cond_*` |

---

## 6. Protection and Security

Restricting what each process can do.

| Service | Linux mechanisms |
|---|---|
| User and group identity | `getuid()`, `setuid()`, `getgid()` |
| File permissions | `chmod()`, `chown()`, `setuid` bit, ACLs |
| Process capabilities | `capabilities(7)`, `cap_set_proc()` |
| Namespaces (isolation) | `unshare()`, `CLONE_NEWNS`, `CLONE_NEWPID` |
| Mandatory access control | SELinux, AppArmor |
| Secure boot | UEFI Secure Boot, IMA |

```c
#include <unistd.h>

if (getuid() == 0) {
    /* running as root: no protection boundary */
}
```

---

# The System Call Interface

```
                 ┌───────────────────────────────┐
   User space    │   User application (C)        │
                 │   printf(), open(), malloc()   │
                 └──────────────┬────────────────┘
                                │
                 ┌──────────────▼────────────────┐
   Libraries     │   glibc / musl                │
                 │   read() → syscall stub       │
                 └──────────────┬────────────────┘
                                │  syscall instruction
                 ┌──────────────▼────────────────┐
   Kernel space  │   Kernel entry point          │
                 │   ──► VFS ──► fs ──► driver  │
                 └───────────────────────────────┘
```

Three layers matter:

| Layer | Runs in | Example |
|---|---|---|
| Application | User mode | `printf("hi\n")` |
| C library wrapper | User mode | `write(1, buf, 3)` |
| System call | Kernel mode | kernel's `sys_write` |

> [!IMPORTANT]
> Not everything is a system call. `printf` is a library function.
> It buffers output and eventually calls `write`.
> A "system call" that never crosses into the kernel is a myth.

---

# Linux System Call Organisation

## Numbered by Category

On x86-64, Linux assigns each system call a number.

| # | Call | # | Call |
|---|---|---|---|
| 0 | `read` | 1 | `write` |
| 2 | `open` | 3 | `close` |
| 9 | `mmap` | 10 | `mprotect` |
| 56 | `clone` | 57 | `fork` |
| 59 | `execve` | 61 | `wait4` |
| 202 | `futex` | 230 | `clock_nanosleep` |
| 293 | `pipe2` | 302 | `prlimit64` |
| 318 | `getrandom` | 334 | `rseq` |

You can look them up:

```bash
# See the full table
man 2 syscalls
ausyscall dump    # from auditd userspace
grep __NR_ /usr/include/x86_64-linux-gnu/asm/unistd_64.h
```

## Grouped Man Sections

```
man 1   user commands              ls, ps, gcc
man 2   system calls              read, open, fork
man 3   library functions         printf, malloc
man 4   devices and interfaces    /dev/sda, terminal
man 5   file formats              /etc/passwd
man 7   misc (syscalls, regex)    syscalls(2), signal(7)
man 8   system administration     systemd, mount
```

> [!TIP]
> `man 2 <name>` is the fastest way to look up the exact Linux system call
> signature, errno values, and return semantics. Slightly different from
> man 3, and easy to forget.

---

# System Libraries

## What a Library Is

A library is pre-compiled code reused by many programs.

Linux system libraries:

| Library | Path | Role |
|---|---|---|
| glibc | `/lib/x86_64-linux-gnu/libc.so.6` | The standard C library, wraps system calls |
| libpthread | (merged into glibc 2.34+) | POSIX threads |
| libm | `/lib/.../libm.so.6` | Math functions |
| libdl | (merged into glibc) | `dlopen()` dynamic loading |

Since glibc 2.34, libpthread, libdl and librt are **merged into libc**.

## Why a C Library Exists at All

| Reason | Example |
|---|---|
| **Buffering** | `printf` buffers; only flushes on newline, buffer full, or `exit` |
| **Portability** | Same source compiles on different kernels |
| **Convenience** | `open()` is a 3-line wrapper around `openat(AT_FDCWD, ...)` |
| **Error handling** | Converts `-errno` return into `-1` + `errno` |

```c
/* glibc's write() is basically this */
ssize_t write(int fd, const void *buf, size_t count)
{
    long ret = syscall(SYS_write, fd, buf, count);
    if (ret < 0 && ret > -4096) {
        errno = -ret;
        return -1;
    }
    return ret;
}
```

---

# What Is a Utility Program?

## Definition

A utility program is a user-space program shipped with the OS that performs a specific task on behalf of the user or the system.

| Category | Examples |
|---|---|
| File management | `ls`, `cp`, `mv`, `rm`, `tar` |
| Process management | `ps`, `top`, `kill`, `nice` |
| Text processing | `grep`, `sed`, `awk`, `sort` |
| Development | `gcc`, `make`, `gdb`, `ld` |
| System config | `mount`, `ip`, `systemctl`, `chown` |

## The Shell Is a Utility Program

`bash` is a **command interpreter**.

```c
/* How a shell works, conceptually */
while (read_a_command_line()) {
    if (is_builtin(cmd)) {
        run_builtin(cmd);
    } else {
        if (strchr(cmd_line, '|')) {
            build_pipeline(cmd_line);
        } else {
            pid = fork();
            if (pid == 0) {
                execvp(cmd_name, argv);
            }
            waitpid(pid, NULL, 0);
        }
    }
}
```

The shell is a **user program**, not part of the kernel.

This is why you can replace it: install `zsh` and `bash` still exists.

---

# Service Interfaces in Linux

| Service | Primary interface |
|---|---|
| Process control | `fork()`, `exec*()`, `wait*()`, signals |
| File system | VFS → `open/read/write/stat` |
| Memory | `mmap`, `brk`, `mprotect` |
| Devices | `/dev` device files + `ioctl()` |
| Network | BSD sockets API |
| IPC | pipes, FIFOs, message queues, shared memory, sockets |
| Logging | `dmesg`, `journalctl`, syslog |
| Configuration | `/etc` + systemd unit files |

### /etc — the configuration backbone

```bash
ls /etc
/etc/os-release       # distro identity
/etc/fstab            # filesystems to mount
/etc/passwd           # user accounts
/etc/systemd/system/  # unit definitions
```

---

# Comparing Interface Types

| Interface | Runs in | Cost | Flexibility |
|---|---|---|---|
| Function call (library) | user | ~1 ns | full userspace only |
| System call | kernel | ~50–500 ns | privileged actions |
| Kernel module | kernel | call cost | new hardware support |
| ioctl | kernel | ~100 ns–10 µs | device-specific control |
| /sys and /proc reads | kernel | syscall cost | introspection/config |

---

# A Complete C Example Using Multiple Services

```c
#include <fcntl.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/mman.h>
#include <sys/stat.h>
#include <unistd.h>

int main(void)
{
    /* Service 1: file management */
    int fd = open("data.bin", O_RDWR | O_CREAT, 0644);
    if (fd < 0) {
        perror("open");
        return 1;
    }

    if (ftruncate(fd, 4096) < 0) {
        perror("ftruncate");
        close(fd);
        return 1;
    }

    /* Service 2: memory management — map the file into memory */
    void *p = mmap(NULL, 4096, PROT_READ | PROT_WRITE,
                   MAP_SHARED, fd, 0);
    if (p == MAP_FAILED) {
        perror("mmap");
        close(fd);
        return 1;
    }

    /* Service 3: I/O through the mapping */
    memcpy(p, "hello", 6);

    /* Service 4: process control */
    pid_t pid = fork();
    if (pid == 0) {
        printf("child sees: %s (pid %d)\n", (char *)p, getpid());
        _exit(0);
    }

    waitpid(pid, NULL, 0);

    munmap(p, 4096);
    close(fd);
    return 0;
}
```

Compile and run:

```bash
gcc -Wall -Wextra -O2 -o demo demo.c
./demo
```

---

# Common Misconceptions

### ❌ "System calls are ordinary function calls."

Incorrect.

A system call switches the CPU from user mode to kernel mode via a trap instruction. The stack, the privilege level, and the code path all change.

---

### ❌ "Calling `printf` is a system call."

Incorrect.

`printf` is a glibc function. It formats into an internal buffer and eventually calls `write`. Most `printf` calls do **not** cause a system call at all.

```bash
strace -e write ./a.out   # shows the real write() calls
```

---

### ❌ "The OS is just the kernel."

Incorrect.

Conventionally the OS = kernel + system libraries + utilities + system programs. Userspace tools are shipped and maintained as part of the OS even though they run unprivileged.

---

### ❌ "The shell is part of the kernel."

Incorrect.

`bash` is a normal user program. It has no privileges. It requests services through the same system calls your own C program uses.

---

### ❌ "Every function in a header is a system call."

Incorrect.

`man 3` is library functions, `man 2` is system calls. Many library functions never leave user space.

---

# Interview Questions

### Basic

- What are system calls? Why are they needed?
- Differentiate between system programs and user programs.
- What is the role of the system call interface?
- What is a utility program?

### Intermediate

- Why does the OS provide abstractions instead of letting programs access hardware directly?
- What is the difference between `man 2` and `man 3`?
- Explain the user-space / kernel-space boundary.
- Why does glibc buffer `printf` output?

### Advanced

- How does the VFS allow the same `open()` call to work on ext4, btrfs and procfs?
- Explain the path from `read()` in a C program to bytes arriving in a buffer.
- How would you implement a new system call in Linux?
- What are the security implications of exposing `/dev/mem`?
- How do capabilities differ from traditional root-based privilege models?

---

# University Exam Notes

### Definitions

- **System call:** A controlled interface through which a user-mode program requests a privileged kernel service.
- **System utility:** A user-space program that performs a specific task using system calls.
- **System program:** Software developed and shipped as part of the operating system distribution.
- **System call interface:** The complete set of system calls exposed by the kernel.

### Frequently Asked Questions

- What are the functions of an operating system?
- What are system calls? Explain with a diagram.
- Differentiate between system programs and user programs.
- What is the difference between a system call and a library function?
- Explain the types of services provided by an OS.
- What is the role of a utility program?

---

# Key Takeaways

- The OS provides **services** so that programs do not touch hardware directly.
- Services are reached exclusively through **system calls**, the user/kernel boundary.
- The major service groups are: process control, file systems, memory, devices, communication, and protection.
- **System utilities** such as `ls` and `ps` are user programs that consume these services.
- Linux wraps system calls in **glibc**, which adds buffering, portability and errno handling.
- Know the man sections: `man 2` = system calls, `man 3` = library functions.
- The VFS is what makes different filesystems look identical to a program.

---

# References

- Operating System Concepts — Silberschatz
- Modern Operating Systems — Tanenbaum and Bos
- The Linux Programming Interface — Kerrisk
- `man 2 syscalls`, `man 2 open`, `man 2 read`
- `man 7 vfs`, `man 5 proc`, `man 5 capabilities`
