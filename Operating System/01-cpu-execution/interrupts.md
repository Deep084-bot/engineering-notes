# Interrupts and I/O

> [!NOTE]
> **Module:** Module I – Introduction
>
> **Difficulty:** ⭐⭐⭐⭐☆
>
> **Prerequisites:**
> - Modes of Operation (cpu-modes.md)
> - System Calls
>
> **Scope:** Linux and C only. No Windows or Java.

---

# Learning Objective

After reading this note, you should be able to:

- Explain what an interrupt is and why hardware needs it.
- Distinguish interrupts, exceptions and traps.
- Explain polling-driven I/O and interrupt-driven I/O, and when each wins.
- Describe the full interrupt handling path on x86-64 Linux.
- Explain what an IDT and an ISR are.
- Explain hard IRQ vs softirq, top half vs bottom half.
- Handle signals in C using `sigaction()`.
- Read `/proc/interrupts` and `/proc/softirqs` to observe real interrupts.

---

# Why Do We Need Interrupts?

## The Problem

The CPU can only do one thing at a time.

Now consider a network card receives a packet. Someone has to find out, and then
react.

There are exactly two possible designs.

### Design A — Polling

```text
  while (1) {
      status = read_network_status_register();
      if (status == PACKET_ARRIVED) {
          handle_packet();
      }
  }
```

- The CPU asks the device "anything for me?" in an endless loop.
- It gets an answer millions of times per second: "no".
- On an idle machine this burns 100% of one core, doing nothing.
- Response time is at worst one poll interval — excellent.

### Design B — Interrupt-driven

```text
  /* CPU is free. It does something else. */

  [network card receives a packet]
          │
          │  raises an interrupt on a line
          ▼
  [CPU finishes current instruction, saves state, jumps to handler]
          │
          ▼
  [handler processes the packet, returns]
          │
          ▼
  [CPU resumes the interrupted program]
```

- The CPU is idle 99.999% of the time when nothing arrives.
- Response happens within microseconds of arrival — better than polling.
- The cost is complexity: the CPU must be able to be interrupted safely.

**Design B is the interrupt.** The whole reason it exists is that Design A
wastes the resource the device was built to be fast at.

---

# Real World Analogy

A **polling** receptionist checks the door every few seconds.

- Reliable, simple, always there.
- Wasteful when nobody arrives.
- Maximum delay: however long the check interval is.

An **interrupt-driven** receptionist wears a pager.

- Does nothing until the pager buzzes.
- Reacts within seconds of an arrival.
- Wastes no effort on empty seconds.
- The pager itself (interrupt controller) must be dependable.

Interrupt-driven I/O is the pager. Polling is the receptionist.

---

# Intuition

The key insight: an interrupt is a **software-visible hardware event**.

```
  ┌────────┐          ┌────────────┐        ┌────────┐
  │ Device │  ─(1)─► │ Interrupt  │ ─(2)► │  CPU   │
  │ (NIC,  │         │ controller │        │        │
  │  disk, │         │ (APIC)     │        │        │
  │ timer) │         └────────────┘        └────┬───┘
  └────────┘                                     │
                                        (3) save PC & flags
                                                 │
                                                 ▼
                                          ┌─────────────┐
                                          │  ISR /      │
                                          │  handler    │
                                          └──────┬──────┘
                                                 │ (4) acknowledge
                                                 ▼
                                          resume interrupted code
```

The critical safety property: the CPU only jumps to a handler at a point where
it is **safe to do so** — between two instructions, with the ability to restore
everything afterwards.

---

# Traps, Exceptions, and Interrupts

These are three different things that all use the same CPU mechanism.

| | Interrupt | Exception | Trap (syscall) |
|---|---|---|---|
| **Origin** | external device | CPU executing an instruction | deliberate `syscall` |
| **Synchronous?** | no (asynchronous) | yes | yes |
| **When** | any time between instructions | during the offending instruction | at the `syscall` instruction |
| **Return address** | next instruction | next / faulting instruction | next instruction |
| **Error possible?** | no | yes (page fault, div by zero) | no |
| **Example** | timer tick, key press, NIC packet | divide by zero, page fault, `#DE` | `read()`, `open()` |
| **Linux name** | hardware IRQ | synchronous exception | system call |

```text
  asynchrony:  ──────■────────────────────►   time
                   ↑ IRQ arrives whenever

  exception:    ───────────■─────────────►
                          ↑ CPU hits a bad instruction right here
```

## Why the distinction matters

| Reason | Explanation |
|---|---|
| **Safety** | Only kernel code may handle a bad instruction. If a *user* program could cause a page fault, it could point the exception table at its own code and become the kernel. |
| **Signal delivery** | The kernel distinguishes "this is a fault, kill the process" from "the device wants attention, handle it and continue". |
| **Debugging** | Asynchronous bugs are far harder to reproduce than synchronous ones. |

> [!IMPORTANT]
> An interrupt is **not** the same as a signal. A signal is a software
> mechanism; an interrupt is a hardware mechanism. Linux uses signals to
> deliver events that are *generated by* interrupts — for example, when
> Ctrl-C raises `SIGINT`, the keyboard IRQ caused it, but `SIGINT` is
> delivered asynchronously to a user process.

---

# Polling vs Interrupt-Driven I/O

## Comparison

| Aspect | Polling | Interrupt-driven |
|---|---|---|
| CPU usage when idle | 100% of a core | ~0% |
| Latency | bounded by poll interval | bounded by interrupt latency |
| Implementation | simple loop | requires ISR + dispatcher |
| Overhead per event | cost of one poll | cost of the trap |
| Best when | latency-critical, one device | many devices, bursty arrivals |
| Risk | CPU starvation | lost interrupts, reentrancy bugs |

## Real Linux cases

| Mechanism | Where used |
|---|---|
| Polling | `epoll` busy mode, NIC interrupt coalescing, some USB drivers |
| Interrupt | default for keyboards, disks, NICs, timers |
| DMA | bulk data transfer, interrupts only for completion |
| Busy-wait spinlock | very short kernel critical paths, to avoid a context switch |

## The Hybrid: Interrupt Coalescing

Modern NICs interrupt **once per batch** rather than once per packet, because
an interrupt costs far more than moving a few hundred bytes.

```c
/* Conceptually: read a whole ring of packets, then one interrupt */
while (packets_pending) {
    struct packet *p = ring_dequeue();
    process(p);
}
kfree(p);
/* one IRQ for 64 packets */
```

This is called interrupt coalescing, and it is a real tuning knob:

```bash
# NIC interrupt coalescing on Intel ixgbe
ethtool -c eth0
ethtool -C eth0 rx-usecs 200 rx-frames 64
```

---

# The Full Interrupt Path (x86-64 Linux)

```text
 1. Device signals on its IRQ line (or writes to a doorbell register in MSI mode)

 2. Interrupt controller (Local APIC / IO-APIC) raises the interrupt
    on the appropriate CPU's interrupt pin

 3. CPU checks for pending interrupts
    • between instructions
    • immediately after a `syscall`, `int`, or `iret`
    • it does NOT interrupt inside a `cli` region

 4. CPU looks up the gate in the IDT (Interrupt Descriptor Table)
    ┌──────────────────────────────────────────┐
    │ vector 0  │  #DE  divide error           │
    │ vector 8  │  #DF  double fault            │
    │ vector 12 │  #PF  page fault              │
    │ vector 14 │  #GP  general protection      │
    │ vector 16 │  #HW  hardware interrupt      │
    │ vector 32 │  timer tick                  │
    │ ...      │                               │
    └──────────────────────────────────────────┘

 5. CPU pushes SS, RSP, RFLAGS, CS, RIP onto the current stack
    (or the IST stack for critical vectors like #DF)

 6. CPU sets the IF flag (interrupts disabled) and loads the handler RIP
    from the IDT gate

 7. Jump to the registered ISR

 8. The ISR: acknowledge the device, do minimal work, mark deferred work

 9. Return → `iretq` pops RIP, CS, RFLAGS, RSP, SS and resumes
```

```c
/* Structuring the IDT gate. Conceptually what the kernel does. */

struct idt_entry {
    uint16_t offset_low;
    uint16_t selector;
    uint8_t  ist;        /* which stack: 0 = current, 1..7 = dedicated */
    uint8_t  type_attr;  /* interrupt gate vs trap gate */
    uint16_t offset_mid;
    uint32_t offset_high;
    uint32_t zero;
} __attribute__((packed));

#define IDT_ENTRIES 256
static struct idt_entry idt[IDT_ENTRIES];

/* Type 0x0E = 64-bit interrupt gate: interrupts are DISABLED on entry.
   This is what makes a handler non-reentrant on the same CPU. */
static struct idt_entry
make_gate(void (*handler)(void), uint16_t sel, uint8_t ist)
{
    struct idt_entry g = {0};
    uint64_t a = (uint64_t)handler;
    g.offset_low  = a & 0xFFFF;
    g.offset_mid  = (a >> 16) & 0xFFFF;
    g.offset_high = (a >> 32) & 0xFFFFFFFF;
    g.selector    = sel;
    g.ist         = ist;
    g.type_attr   = 0x8E;      /* present, ring 0, 64-bit interrupt gate */
    return g;
}
```

> [!NOTE]
> An **interrupt gate** (0x8E) clears IF on entry, so the handler runs with
> interrupts off. A **trap gate** (0xF) leaves IF set. Linux uses interrupt
> gates for almost everything, and dedicated IST stacks for double fault.

---

# Hard IRQ vs Soft IRQ

Once the kernel is running, not all deferred work should happen in the ISR.

Linux splits the work in two.

| | Hard IRQ | Soft IRQ |
|---|---|---|
| Also called | top half, hardware IRQ | bottom half, deferred |
| Masked by | disabling interrupts on that CPU | `local_irq` too — all are run in hard-IRQ context unless explicitly deferred |
| Triggered by | hardware | marked by the hard IRQ |
| Runs | immediately, in interrupt context | at the end of the hard IRQ handling, on return to idle, or via `ksoftirqd` |
| May sleep? | **never** | **never** (softirq context) |
| Typical work | read the device register, clear the IRQ, queue the work | process the networking stack, run timers, do block-layer completion |

```c
/* A hard IRQ that does almost nothing */

static irqreturn_t my_irq_handler(int irq, void *dev)
{
    /* Runs in interrupt context: cannot sleep, cannot allocate with GFP_KERNEL */

    /* 1. Tell the device we heard about the event */
    writel(STATUS_ACK, MY_DEVICE->reg + OFF_STATUS);

    /* 2. Defer everything expensive */
    my_pending_events++;                    /* atomic_inc() */
    tasklet_schedule(&my_deferred_tasklet);  /* or raise a softirq */

    return IRQ_HANDLED;
}

/* The bottom half: now we can do the real work */
static void my_tasklet_fn(struct tasklet_struct *t)
{
    /* Still non-sleepable, but interrupts are on and we are not on the ISR stack */
    while (my_pending_events--) {
        struct event ev = read_event_from_ringbuffer();
        process_event(&ev);
    }
    tasklet_enable(t);
}
```

## The modern picture: softirqs, tasklets, and workqueues

```text
  HARD IRQ
    │
    ├── device-specific work (must be fast, atomic, no sleeping)
    │
    └── raise_softirq() / tasklet_schedule() / schedule_work()
              │
              ▼
      ┌───────────────────────────────────────────────┐
      │ Deferred work, in rough order of preference:  │
      │                                               │
      │  softirq  ── runs at end of hard IRQ, on      │
      │              return from idle. Cannot sleep.  │
      │                                               │
      │  tasklet  ── a softirq-based mechanism.       │
      │              Cannot sleep. Limited to CPU 0   │
      │              on older kernels.                │
      │                                               │
      │  workqueue ── runs in a kernel thread         │
      │              context. CAN sleep, can allocate  │
      │              with GFP_KERNEL. Almost always    │
      │              the right answer.                 │
      └───────────────────────────────────────────────┘
```

> [!TIP]
> Rule of thumb for kernel-style C: if the work might sleep, take it out of
> interrupt context and put it on a workqueue. Sleeping in atomic context is
> the single most common cause of kernel crashes.

## The core softirqs

| Softirq | Handles |
|---|---|
| `NET_RX_SOFTIRQ` | received packets, from the network driver |
| `NET_TX_SOFTIRQ` | transmit completion |
| `BLOCK_SOFTIRQ` | block I/O completion |
| `TIMER_SOFTIRQ` | timer callbacks |
| `RCU_SOFTIRQ` | read-copy-update grace periods |

```bash
# Watch softirqs happening in real time
watch -n1 "cat /proc/softirqs"

# Column meanings
head -3 /proc/softirqs
```

```text
                    TIMER    NET_TX    NET_RX      BLOCK    TASKLET
CPU0               128734    1204     881234        3312        44
CPU1               121903     1198     870112        3288        41
```

Differences between the CPU rows are what you are looking for: a CPU handling
far more network packets than its peers is a load-imbalance signal.

---

# Signals: Software Interrupts from the Kernel's Point of View

A **signal** is the most common way the kernel tells a user process that
something happened.

```c
#include <signal.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

static volatile sig_atomic_t got_signal = 0;

static void handler(int sig)
{
    /* Async-signal-safe operations only */
    got_signal = sig;
}

int main(void)
{
    struct sigaction sa;
    sa.sa_handler = handler;
    sigemptyset(&sa.sa_mask);
    sa.sa_flags = SA_RESTART;

    if (sigaction(SIGINT, &sa, NULL) == -1) {
        perror("sigaction");
        return 1;
    }

    printf("waiting for SIGINT (press Ctrl-C)...\n");

    while (got_signal == 0) {
        sleep(1);
    }

    printf("caught signal %d, exiting cleanly\n", got_signal);
    return 0;
}
```

```bash
gcc -Wall -Wextra -o sigdemo sigdemo.c
./sigdemo
^Ccaught signal 2, exiting cleanly
```

## The handler runs on the interrupted stack

```text
  main:                      printf("waiting\n")
    │
    ▼
  Ctrl-C ──► line discipline generates SIGINT
    │
    ▼
  kernel delivers signal at a safe point, pushing a signal frame
    │
    ▼
  handler(SIGINT)  runs on the *same stack* as main
    │
    ▼
  handler returns ──► kernel restores the old frame
    │
    ▼
  main resumes after the interrupted system call, returns EINTR
```

## Async-signal-safe rules

A handler may run in the middle of anything, including inside `malloc`.

So inside a handler you may only use async-signal-safe functions:

```c
/* SAFE inside a handler */
write(fd, buf, len);
_exit(0);
raise(sig);
kill(getpid(), sig);
signal(sig, handler);

/* NOT SAFE inside a handler */
printf("...");      /* malloc, stdio locks */
malloc(100);
free(p);
pthread_mutex_lock(&m);
```

> [!IMPORTANT]
> `printf` inside a signal handler is a classic bug. If the signal arrives
> while `printf` holds the stdio lock, the handler deadlocks against itself.
> Use `write(STDERR_FILENO, ...)` instead.

---

# Observing Interrupts in Linux

## /proc/interrupts

```bash
cat /proc/interrupts
```

```text
            CPU0       CPU1       CPU2       CPU3
  0:         18          0          0          0   IO-APIC   2-edge      timer
  1:          0          0          0          0   IO-APIC   1-edge      i8042
  8:          0          0          0          0   IO-APIC   8-edge      rtc0
  9:       1042        998       1077       1001   IO-APIC   9-fasteoi   acpi
 12:      88123      87904      89001      87552   IO-APIC  12-fasteoi   nvme0q0
 14:      33012      33145      32988      33201   IO-APIC  14-fasteoi   ehci_hcd
141:     102934     104211     101887     103002   PCI-MSI 318721-edge    nvme0q0
231:         12          9         14          11   PCI-MSI 514688-edge    xhci_hcd
```

| Column | Meaning |
|---|---|
| `0:`, `1:` | IRQ line number |
| `CPU0..CPU3` | how many interrupts this CPU handled for that line |
| `IO-APIC` | routed by the global interrupt controller |
| `PCI-MSI` | message-signalled, no shared line — direct to one CPU |
| `edge` / `level` | edge-triggered vs level-triggered |
| last field | the driver that claimed the line |

```bash
# Just the NVMe line, updating
watch -n0.5 "grep -E 'nvme|eth' /proc/interrupts"
```

## Reading the counters correctly

If the NVMe line is evenly spread, your I/O is balanced.

If one column dominates, that CPU is doing all the work:

```text
 12:  88123  87904  89001  87552   ← balanced
141: 102934      0      0      0   ← CPU0 starved the others
```

This is a very common finding on a 16-core machine, and the cause is usually
that interrupts were pinned at boot:

```bash
# Fix: stop pinning NVMe IRQs to CPU0
echo 0 > /proc/irq/141/smp_affinity
cat /proc/irq/141/smp_affinity
```

## Interrupts-per-second

```bash
# Count total interrupts over one second
grep -E '^ *[0-9]+:' /proc/interrupts | awk '{s+=$2+$3+$4+$5} END {print s}'
sleep 1
grep -E '^ *[0-9]+:' /proc/interrupts | awk '{s+=$2+$3+$4+$5} END {print s}'

# Or, much simpler, via /proc/stat
vmstat 1 5
```

```text
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa
 2  0      0 3800000  12000 2800000   0    0    14     2 1204  88  4  2 94  0
```

`in` = interrupts/sec, `cs` = context switches/sec, `wa` = iowait.

A high `wa` with low `in` means: slow disk, not an interrupt problem.

---

# Common Misconceptions

### ❌ "An interrupt is a kind of function call."

Incorrect.

A function call pushes a return address and jumps. An interrupt pushes the
*entire machine state* (RIP, CS, RFLAGS, RSP, SS) and changes privilege
potential, and the CPU can take one at any instruction boundary.

---

### ❌ "Higher interrupt numbers mean higher priority."

Incorrect.

IRQ 0 is the timer, IRQ 1 is the keyboard, IRQ 12 is often the disk. These are
*device* identifiers, not priorities. Priority comes from the interrupt
controller's configuration and from which CPU is targeted.

---

### ❌ "Signals and interrupts are the same thing."

Incorrect.

Interrupts are hardware, asynchronous, and handled in kernel space. Signals are
software, delivered by the kernel at a safe point, and handled in user space.
Signals are often *caused by* interrupts, but they are a different mechanism.

---

### ❌ "A syscall is an interrupt."

Partly, and this is a fine point worth being precise about.

On x86-64, `syscall` is a dedicated instruction, not `int 0x80`. It is a
*trap*, taken deliberately at a known point, with a known register layout. It
is architecturally separate from hardware interrupts, though both go through
the IDT-like gate machinery and both save state.

---

### ❌ "Softirqs can sleep."

Incorrect.

Softirqs run in interrupt context. They cannot sleep, cannot allocate with
`GFP_KERNEL`, and cannot take a mutex. If you need to sleep, use a workqueue.

---

# Interview Questions

### Basic

- What is an interrupt? Why does hardware need it?
- Differentiate between polling and interrupt-driven I/O.
- What are the steps involved in handling an interrupt?
- What is the difference between an interrupt, an exception and a trap?

### Intermediate

- What is the IDT? What happens when an interrupt occurs on x86-64?
- Explain top half and bottom half processing. Why split them?
- What are softirqs, tasklets and workqueues, and when do you use each?
- Why is `printf` unsafe inside a signal handler?

### Advanced

- Explain the full Linux IRQ path, from device to C ISR, including MSI.
- How would you determine whether a machine is interrupt-bound?
- Why does Linux use a per-CPU interrupt affinity mask, and how does it help?
- How do interrupts interact with preemptive scheduling?
- What is a spurious interrupt, and how do you debug one?

---

# University Exam Notes

### Definitions

- **Interrupt:** A hardware-generated signal that suspends the CPU's current
  instruction, saves its state, and transfers control to a handler.
- **Exception:** A synchronous, CPU-generated trap caused by the instruction
  currently being executed.
- **Trap:** A deliberate, synchronous software entry into the kernel, such as a
  system call.
- **Top half:** The interrupt handler itself. Runs immediately, does minimal
  work, cannot sleep.
- **Bottom half:** Deferred work scheduled from the top half, executed later.
- **Soft IRQ:** A kernel-internal deferral mechanism, run in interrupt context.
- **Signal:** A software notification delivered by the kernel to a process.

### Frequently Asked Questions

- What is an interrupt? Explain the interrupt handling process.
- Differentiate between polling and interrupt-driven I/O.
- What is the role of the IDT and ISR?
- Explain the concept of top half and bottom half interrupt processing.
- What are softirqs, tasklets and workqueues?
- What are signals? How are they different from interrupts?
- What is interrupt-driven I/O and why is it better than polling?

---

# Key Takeaways

- Interrupts exist so the CPU does not have to **poll** devices, wasting cycles
  on answers that are almost always "no".
- Interrupts, exceptions and traps are distinct sources but share one hardware
  mechanism: save state, jump through a gate, later restore.
- Polling is simple and low-latency but burns CPU; interrupts are efficient but
  add complexity and require careful hardware support.
- The x86-64 path: device → APIC → CPU → IDT gate → handler → `iretq`.
- Linux defers work out of the ISR using softirqs, tasklets and workqueues.
  Only a workqueue can sleep.
- Signals are the user-space consequence of kernel events; handle them with
  `sigaction()` and only async-signal-safe functions.
- `/proc/interrupts`, `/proc/softirqs` and `vmstat` are your ground truth for
  interrupt load and imbalance.

---

# References

- Operating System Concepts — Silberschatz
- Intel 64 and IA-32 Architectures Software Developer's Manual, Vol. 3 (Ch. 6, Interrupts)
- The Linux Programming Interface — Kerrisk (see `signal(7)`, `sigaction(2)`)
- Linux Device Drivers — Corbet, Rubini, Kroeker-Hatfield
- `man 7 signal`, `man 4 console_codes`, `man 2 request_irq`
- `/proc/interrupts`, `/proc/softirqs`, `/proc/stat` — kernel documentation
