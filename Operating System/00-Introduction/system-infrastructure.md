# System Infrastructure

> [!NOTE]
> **Module:** Module I – Introduction
>
> **Difficulty:** ⭐⭐☆☆☆
>
> **Prerequisites:**
> - OS Services and Functions
> - What is an Operating System
>
> **Note:** This note covers the layered structure, subsystems and support
> interfaces that make up an operating system's infrastructure on Linux.

---

# Learning Objective

After reading this note, you should be able to:

- Describe the layered structure of a modern operating system.
- Explain the subsystems that make up the Linux kernel.
- Explain what the virtual filesystem, the proc filesystem and sysfs are.
- Explain how the kernel, libraries and utilities fit together.
- Describe how a Linux system is organised on disk.
- Use the tools that let you inspect the infrastructure at runtime.

---

# Why Do We Need a Layered Structure?

## The Problem

A monolithic kernel that puts all logic in one place has a familiar failure mode:

- Every component can call every other component.
- A bug in the network driver can corrupt the memory allocator.
- Nobody can change the filesystem without risking the scheduler.
- Nobody can test the scheduler without a disk.
- A single compilation unit becomes unmaintainable.

This is called **spaghetti architecture**, and it is what the early operating systems looked like.

The problem is not that the components are in the same binary.

The problem is that there is no **discipline** about who may depend on whom.

---

# Real World Analogy

Consider a large office building.

- The **foundation and structural frame** are the kernel.
- **Lift shafts and stairwells** are the interfaces between layers.
- **Rooms** are subsystems. A room in one floor does not wire itself directly into a room on another floor.
- **Signs and rules** define who may go where.

The building stays maintainable because layers only talk to the layer directly above or below them, and each layer offers a small, documented interface.

---

# Intuition

Infrastructure is about **boundaries**.

```
        ┌─────────────────────────────────────────┐
        │              USER SPACE                  │
        │  Shell · Editors · Compilers · Servers  │
        └──────────────────┬──────────────────────┘
                           │  system calls
        ┌──────────────────▼──────────────────────┐
        │            SYSTEM CALL API              │  ← stable, narrow
        └──────────────────┬──────────────────────┘
        ┌──────────────────▼──────────────────────┐
        │              KERNEL SPACE                │
        │  ┌──────────┐ ┌────────┐ ┌──────────┐   │
        │  │ Scheduler│ │  VFS   │ │   MM     │   │
        │  └────┬─────┘ └───┬────┘ └────┬─────┘   │
        │       └───────────┴───────────┘          │
        │      ┌──────────────────────────┐         │
        │      │  Block / Network / Char  │  drivers
        │      └────────────┬─────────────┘         │
        │                   ▼                       │
        │        ┌────────────────────┐             │
        │        │  Hardware (MMIO,   │             │
        │        │  IRQ, DMA, buses)   │             │
        │        └────────────────────┘             │
        └─────────────────────────────────────────┘
```

Everything above a line is a contract.

Breaking a contract is expensive; that is why the system call interface and the VFS interface have remained stable for decades.

---

# Layer 1 — Hardware

The bottom layer. The OS is a controller for it, not a replacement for it.

| Hardware | How the OS uses it |
|---|---|
| CPU | Executes kernel and user instructions |
| MMU | Enforces address-space isolation via page tables |
| APIC / PIC | Delivers interrupts |
| Timer | Generates preemption ticks |
| RAM | Holds kernel and process address spaces |
| Disk / NVMe | Block storage behind the block layer |
| NIC | Network I/O behind the network stack |
| GPU | Often accessed through the DRM subsystem |

The kernel never speaks to hardware registers directly for common devices.

It talks to a **driver**, which is a kernel module dedicated to that hardware.

---

# Layer 2 — The System Call Interface

## Definition

The system call interface is the **only** sanctioned way for user space to request a kernel service.

On Linux it is a single, stable, numeric ABI:

```c
/* Every one of these is a request into the kernel */
#include <unistd.h>
read(fd, buf, n);
open(path, flags);
mmap(addr, len, prot, flags, fd, off);
```

The same binary compiled once will run on any Linux kernel with a compatible ABI, even if the internal implementation is completely different.

> [!TIP]
> This is why Linux has such strong binary compatibility compared to
> other systems. The system call interface is a promise that rarely breaks.

---

# Layer 3 — Kernel Subsystems

The Linux kernel is organised into subsystems, each responsible for one resource class.

## Process Scheduler

Decides which task runs on which CPU.

```text
      ┌──────────────────────┐
      │  Scheduler (CFS)     │
      │  picks a task from   │
      │  the run queue       │
      └──────────┬───────────┘
                 ▼
      ┌──────────────────────┐
      │  Dispatcher           │
      │  context switch       │
      └──────────────────────┘
```

Relevant files: `kernel/sched/`, `include/linux/sched.h`.

## Memory Management (MM)

Owns virtual memory, page tables, the page allocator, and mmap.

```text
  Virtual address space
  ┌────────┬────────┬────────┬────────┐
  │ code   │ rodata │  data  │  heap  │ ← user-space view
  ├────────┴────────┴────────┼────────┤
  │        stack             │  guard  │
  └──────────────────────────┴────────┘
                 │
                 ▼  page tables (MMU enforces)
  Physical RAM frames
```

Relevant files: `mm/`, `include/linux/mm.h`.

The kernel allocates physical memory with the **buddy allocator**, and kernel objects with the **slab allocator**:

```bash
grep -E 'MemTotal|MemFree|Slab|SReclaimable' /proc/meminfo
```

## Virtual File System (VFS)

The layer that makes every filesystem look like files and directories.

```text
        open("notes.md")
              │
              ▼
    ┌───────────────────┐
    │        VFS        │  name → inode lookup, caching
    └─────────┬─────────┘
      ┌───────┼────────┬──────────┬──────────┐
      ▼       ▼        ▼          ▼          ▼
   ext4     btrfs     XFS     tmpfs     procfs
      │       │        │          │          │
   block    block    block      RAM    kernel
    layer    layer    layer              objects
```

```bash
# Which filesystem backs a path?
df -T .
findmnt -T /etc
cat /proc/mounts
```

## Block Layer

Turns block devices into a queue of I/O requests, reordering and merging them for throughput.

```text
  read(fd)
     │
     ▼
  bdev file
     ▼
  /dev/sda  ──►  block layer queue  ──►  I/O scheduler
                                             │
                              none / mq-deadline / bfq
                                             ▼
                                    NVMe / SCSI driver
                                             ▼
                                          hardware
```

```bash
cat /sys/block/sda/queue/scheduler
[none] mq-deadline kyber bfq
```

## Character Devices and TTY

Byte-oriented devices: terminals, serial ports, `/dev/null`, `/dev/random`.

```text
  keyboard ──► i8042 driver ──► input subsystem ──► evdev
                                                      │
                                                      ▼
                                          terminal emulator (GUI)
                                                      │
                                                          tty layer
                                                              │
                                                    ┌─────────┴─────────┐
                                                    ▼                   ▼
                                              line discipline      pty pair
                                              (canonical mode)    (terminal emulator)
```

## Networking Stack

```text
  socket()
     │
     ▼
  BSD socket layer
     │
     ▼
  TCP / UDP        (transport)
     │
     ▼
  IP / IPv6         (network)
     │
     ▼
  netfilter / conntrack  (firewall, NAT)
     │
     ▼
  Ethernet / Wi-Fi driver
```

## Signal Delivery and IPC

`struct signal_struct` per signal, `sigqueue` for real-time signals, and the syscall-based IPC mechanisms (`pipe`, `msgget`, `shmget`, futex).

## Synchronisation Primitives

The kernel's own locking infrastructure: spinlocks, mutexes, semaphores, read-write semaphores, and the futex fast path for userspace threads.

---

# Layer 4 — The Virtual Filesystems as Infrastructure

Linux exposes much of its own state as files. This is not a gimmick; it is a deliberate design decision.

## /proc — process and system information

`/proc` is a virtual, in-memory filesystem. Files are generated on read.

```bash
ls /proc
```

```text
1          2      3    4    5   ...   1234        ← per-process directories
self       thread-self
uptime     meminfo   cpuinfo   version   cmdline
devices    interrupts  softirqs  modules
uptime     loadavg    meminfo   stat
```

Per-process entries are the process control block, exposed as files:

```bash
ls /proc/$$/          # $$ is the current shell's PID
cat /proc/$$/status
cat /proc/$$/stat
readlink /proc/$$/exe
```

```text
# /proc/PID/stat — the key fields of the PCB
pid   comm  state  ppid  pgrp  session  tty_nr  tpgid  flags
minflt cminflt  majflt  cmajflt  utime  stime  cutime  cstime
priority  nice  num_threads  itrealvalue  starttime  vsize  rss
```

This is how `ps`, `top` and `htop` are implemented — they read `/proc`.

## /sys — device and driver information

sysfs describes the device tree and exposes tunables as files.

```bash
ls /sys/class/
```

```text
block  input  net  power_supply  thermal  tty  usb  ...
```

```bash
# CPU frequency scaling
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
# pick performance | powersave | ondemand | schedutil

# USB devices
lsusb
cat /sys/bus/usb/devices/*/product

# Block device
cat /sys/block/nvme0n1/queue/nr_requests
```

> [!TIP]
> "Everything is a file" is not marketing. `ip link`, `cat` of a sysfs
> attribute, and `ps` are all just reading structured text. You can debug
> most of a Linux system with `cat`, `ls` and `strace`.

## /dev — device nodes

```bash
ls -l /dev/sda /dev/null /dev/tty1
```

```text
brw-rw---- 1 root disk  8,  0 ...  /dev/sda
crw-rw-rw- 1 root root  1,  3 ...  /dev/null
crw-rw-rw- 1 root tty   4,  1 ...  /dev/tty1
```

- `b` = block device, `c` = character device
- `8, 0` = major 8, minor 0, which the kernel uses to find the driver

```c
/* Device numbers are the kernel's driver dispatch key */
#include <sys/sysmacros.h>
#include <stdio.h>

struct stat st;
stat("/dev/sda", &st);
printf("major=%ld minor=%ld\n",
       major(st.st_rdev), minor(st.st_rdev));
```

---

# Layer 5 — C Libraries and Utilities

User space has its own structure.

```text
  ┌────────────────────────────────────────────┐
  │  Applications: gcc, vim, python3, nginx    │
  ├────────────────────────────────────────────┤
  │  C library (glibc)                         │
  │    printf, malloc, pthread_create,         │
  │    and thin wrappers over system calls     │
  ├────────────────────────────────────────────┤
  │  System utilities: ls, ps, mount, systemctl│
  ├────────────────────────────────────────────┤
  │  System call interface  ◄── the boundary   │
  ├────────────────────────────────────────────┤
  │  Kernel subsystems                          │
  └────────────────────────────────────────────┘
```

```bash
# Which libraries does a binary need?
ldd /bin/ls
```

```text
	linux-vdso.so.1 (0x00007ffd...)
	libselinux.so.1 => /lib/x86_64-linux-gnu/libselinux.so.1
	libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6
	/lib64/ld-linux-x86-64.so.2 (0x00007f...)
```

---

# On-Disk Layout of a Linux System

```text
/
├── boot/          kernel image, initramfs, GRUB
├── etc/           configuration
│   ├── fstab            which filesystems to mount
│   ├── passwd           user accounts
│   └── systemd/system/  unit definitions
├── proc/          kernel + process state (virtual)
├── sys/           device + driver state (virtual)
├── dev/           device nodes
├── tmp/           temporary files
├── var/           logs, databases, packages
│   └── log/               system logs
├── home/          user home directories
├── lib/           shared libraries
├── bin/ sbin/     binaries
└── usr/           most modern userland lives here
    ├── bin/ sbin/ lib/
    └── share/
```

Check what a path actually is:

```bash
df -T /home
mount | column -t
```

---

# A Layering Violation, Demonstrated

This program is legal on Linux and it is exactly what a layering violation looks like:

```c
/* Reading /dev/mem gives raw physical memory access. */
#include <fcntl.h>
#include <stdio.h>
#include <unistd.h>

int main(void)
{
    int fd = open("/dev/mem", O_RDONLY);
    if (fd < 0) {
        perror("open /dev/mem");
        return 1;
    }

    unsigned char byte;
    ssize_t n = read(fd, &byte, 1);
    if (n == 1)
        printf("lowest physical byte = 0x%02x\n", byte);
    else
        perror("read");

    close(fd);
    return 0;
}
```

```bash
gcc -o mempeek mempeek.c
./mempeek
Permission denied
```

Why it fails:

1. The `open()` is a system call — it crossed the boundary legally.
2. The **permission check inside the driver** denies it.
3. `CAP_SYS_RAWIO` is required, which root has and normal users do not.

The interface was used correctly; the policy inside the kernel refused.

> [!TIP]
> This is the value of the layering. The interface is small and stable;
> the policy is inside the kernel and can be tightened without touching
> any application.

---

# Tools for Inspecting the Infrastructure

```bash
# What is running?
ps aux
top -H                      # include threads
systemctl status

# What is the kernel doing?
dmesg
cat /proc/interrupts
cat /proc/softirqs
cat /proc/stat               # includes 'ctxt' and 'processes'

# What is on disk?
df -hT
lsblk -f
findmnt

# What devices exist?
lsusb
lspci -nn
ls /sys/class/net

# What does a program actually ask for?
strace -f -c ./program       # syscall summary with timings
ltrace ./program             # library calls
```

Read the interrupt and context-switch counters to see real kernel activity:

```bash
grep -E 'ctxt|processes' /proc/stat
```

A jump in `ctxt` while the machine feels slow is a strong signal that
context switching, not compute, is the bottleneck.

---

# Common Misconceptions

### ❌ "A monolithic kernel has no structure."

Incorrect.

Linux is monolithic in *linkage* — everything is built into one image — but it is not unstructured. Subsystems have clear internal layering, enforced by headers, `EXPORT_SYMBOL`, and lock-discipline rules.

---

### ❌ "/proc, /sys and /dev are real files on disk."

Incorrect.

All three are virtual filesystems generated by the kernel in memory. There is no inode on a disk, and no data blocks exist. Reading is a function call that formats text on the fly.

---

### ❌ "Layering means performance is sacrificed for beauty."

Incorrect.

Layers are how you keep performance optimisable. The VFS lets Linux add a filesystem without touching any application, and let it be fast by adding caching at exactly one layer.

---

### ❌ "A system call is only about files and processes."

Incorrect.

System calls cover memory (`mmap`), time (`clock_gettime`), threads (`clone`), networking (`socket`), and device control (`ioctl`).

---

# Interview Questions

### Basic

- What is meant by the layered structure of an operating system?
- What are the main subsystems of the Linux kernel?
- What is the role of the VFS?
- What is /proc used for?

### Intermediate

- Why does the VFS exist, and what does it abstract away?
- How does sysfs let you tune device behaviour without recompiling the kernel?
- What is the difference between /proc, /sys and /dev?
- How does the block layer improve disk throughput?

### Advanced

- Linux is monolithic. Why is that still considered layered?
- How would you add a new filesystem to Linux, and where would it attach?
- What is the cost of the VFS layer, and how does Linux minimise it?
- How do capabilities make /dev/mem safe without a hard-coded UID check?

---

# University Exam Notes

### Definitions

- **System infrastructure:** The layered collection of hardware, kernel subsystems, interfaces, libraries and utilities that together provide the OS services.
- **VFS:** A kernel layer providing a uniform interface over all filesystems.
- **sysfs:** A virtual filesystem exposing kernel objects and their attributes as files.
- **procfs:** A virtual filesystem exposing process and kernel state as files.

### Frequently Asked Questions

- Draw the layered structure of an operating system.
- Explain the subsystems of the Linux kernel.
- What is the VFS? Why is it needed?
- Differentiate between /proc, /sys and /dev.
- Explain the role of system libraries and utilities in system infrastructure.

---

# Key Takeaways

- Infrastructure is a set of **layers and contracts**, not just a list of components.
- The **system call interface** is the narrow, stable boundary between user and kernel space.
- The Linux kernel is organised into subsystems: scheduler, MM, VFS, block layer, networking, character devices, signals and synchronisation.
- `/proc`, `/sys` and `/dev` are **virtual filesystems** that expose kernel state and devices as files.
- glibc and system utilities sit above the boundary and consume kernel services.
- Knowing `df -T`, `findmnt`, `lsblk`, `ls /sys/class` and `strace -c` covers most infrastructure debugging.

---

# References

- Operating System Concepts — Silberschatz
- The Linux Kernel — documentation at kernel.org/doc/html/latest/
- `man 5 proc`, `man 5 sysfs`, `man 4 vfs`
- `man 7 namespaces`, `man 7 capabilities`
- Linux Kernel Development — Robert Love
