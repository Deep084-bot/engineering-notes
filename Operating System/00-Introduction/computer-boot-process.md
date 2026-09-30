# Computer Boot Process (Linux)

> [!NOTE]
> **Module:** Module I – Introduction
>
> **Difficulty:** ⭐⭐⭐☆☆
>
> **Prerequisites:**
> - What is an Operating System
> - Kernel

---

# Learning Objective

After reading this note, you should be able to:

- Explain what happens between pressing the power button and seeing a login screen.
- Describe the role of BIOS/UEFI firmware.
- Understand the Master Boot Record (MBR) and EFI System Partition.
- Explain what a bootloader does and why GRUB is used on Linux.
- Describe how the Linux kernel is loaded and decompressed.
- Explain the purpose of the initial RAM disk (initramfs).
- Describe how `systemd` becomes PID 1 and starts services.
- Use Linux commands to observe the boot process.

---

# Why Do We Need a Bootloader?

## The Problem

The moment you press the power button, the CPU has **no instructions to execute**.

The CPU does not know:

- Which device to read from
- Where the operating system is stored
- How much memory it has
- How to start any program

Yet we need to somehow load a very large, very complex program (the Linux kernel) into memory and start it.

But the kernel itself cannot load the kernel.

Something must exist *before* the operating system.

That "something" is the **bootloader**.

---

# Real World Analogy

Imagine you land at an airport in a new country.

The aircraft is on the runway, fully fuelled, ready to fly.

But the pilot has never been here before.

They do not know:

- Which runway to use
- How long it is
- Whether there is fog
- Where the terminal building is
- How to contact air traffic control

Before the aircraft can move, it needs a **ground crew** to guide it.

That ground crew is the bootloader.

Once the aircraft is guided, the real operation (the OS) takes over.

---

# Intuition

Booting a Linux system is a chain of handoffs.

Each stage does one job and passes control to the next.

```
Power Button
      │
      ▼
  Firmware (BIOS/UEFI)         ← test hardware, pick boot device
      │
      ▼
  Bootloader (GRUB2)           ← load kernel from disk
      │
      ▼
  Linux Kernel (vmlinuz)       ← initialise core subsystems
      │
      ▼
  initramfs                   ← mount real root, load modules
      │
      ▼
  systemd (PID 1)              ← start services
      │
      ▼
  Login Shell                  ← user sees a prompt
```

---

# The Complete Boot Sequence

## Stage 1 — Power On and Firmware Init

When you press the power button:

1. The **power supply** delivers stable power to all components.
2. The **firmware** (stored in a chip on the motherboard) executes.
3. On a modern system, the firmware is **UEFI** (Unified Extensible Firmware Interface).

The firmware performs a **POST** (Power-On Self-Test):

- Checks that the CPU is alive
- Checks that RAM is present and readable
- Checks that the boot device is connected
- Initialises other hardware

> [!TIP]
> The POST writes a small message to the screen, usually from a
> video device that is initialised very early, because the OS
> drivers are not loaded yet.

```text
Dell Inc. UEFI BIOS v2.10.0
Copyright (c) 2010 Dell Inc.
CPU: Intel(R) Core(TM) i7-9750H CPU @ 2.60GHz
Memory Test:  16777216K OK
Detecting IDE drives...
  Primary Master:   Samsung SSD 980 PRO 1TB
Booting from Hard Disk...
```

---

## Stage 2 — Boot Device Selection

The firmware searches for a bootable device according to the **boot order** stored in its configuration.

Typical boot order on a Linux laptop:

1. NVMe SSD (internal)
2. USB drive
3. Network (PXE boot) — rarely used by default

The firmware reads the **first sector (512 bytes)** of the chosen device.

That first sector is the **Master Boot Record (MBR)** on a legacy BIOS system, or a **EFI System Partition (ESP)** on a UEFI system.

> [!NOTE]
> Legacy BIOS systems use MBR (partition table + 446-byte boot code).
> UEFI systems use an ESP partition (FAT32) and load `EFI/BOOT/BOOTX64.EFI`.
> Modern Linux systems are almost always UEFI.

---

## Stage 3 — The Bootloader (GRUB2)

The bootloader is the first real program loaded into memory by the firmware.

On Ubuntu, Fedora, Arch and most modern distributions, the bootloader is **GRUB2** (GRand Unified Bootloader version 2).

### What GRUB Does

```
GRUB's responsibilities:

1. Locate the Linux kernel image (vmlinuz-<version>)
2. Locate the initramfs image (initrd.img-<version>)
3. Allow the user to select which kernel to boot
4. Pass boot parameters to the kernel (root=, quiet, etc.)
5. Load the kernel into memory and jump to it
```

When you install Linux, GRUB writes its own code into the MBR (or ESP) and a configuration file on disk.

You can inspect it:

```bash
# GRUB configuration files
ls /boot/grub/
cat /boot/grub/grub.cfg

# What's installed on the EFI partition
sudo efibootmgr -v
```

### The GRUB Menu

When GRUB starts, it may show a menu:

```text
Ubuntu 22.04.3 LTS
Ubuntu 22.04.3 LTS (recovery mode)
Memory test (memtest86+)
Advanced options for Ubuntu
```

The default timeout is often 5 seconds. After that, it boots the default entry automatically.

> [!TIP]
> If your Linux system does not show the GRUB menu, hold or repeatedly
> tap `Shift` (BIOS) or `Esc` (UEFI) during boot to reveal it.

---

## Stage 4 — Loading the Kernel

GRUB reads the two critical files from the filesystem:

| File | Purpose |
|---|---|
| `/boot/vmlinuz-<version>` | The Linux kernel image (compressed) |
| `/boot/initrd.img-<version>` | The initial RAM disk |

The kernel image is usually **compressed** (gzip, zstd, lz4, or bzImage format).

This matters because the kernel must fit in a limited amount of memory and be loaded fast.

```bash
# See what is currently booted
ls -lh /boot/
uname -r
```

The kernel is not a normal ELF executable that the kernel can run.

It is loaded by GRUB using a special **Linux boot protocol**:

```text
GRUB:
  1. Put the kernel image at physical address 0x100000
  2. Put the initrd right after it
  3. Point a register at a "setup header" in the kernel
  4. Jump to the kernel entry point
     → starts executing in 32-bit protected mode
     → the kernel sets up 64-bit long mode itself
```

GRUB also passes **command-line parameters** to the kernel. You can see them here:

```bash
cat /proc/cmdline
```

Typical output:

```text
BOOT_IMAGE=/boot/vmlinuz-5.15.0-91-generic root=UUID=8f3e2a1b-4c5d-6e7f-8a9b-0c1d2e3f4a5b ro quiet splash
```

| Parameter | Meaning |
|---|---|
| `BOOT_IMAGE=` | Which kernel image was loaded |
| `root=UUID=...` | Which device/directory is the real root filesystem |
| `ro` | Mount root as read-only initially |
| `quiet` | Suppress most boot messages |
| `splash` | Show a graphical boot splash screen |

Removing `quiet` and `splash` gives you the full boot log — the most useful thing when debugging boot problems.

---

## Stage 5 — Kernel Initialisation

Once GRUB jumps to the kernel entry point, the kernel takes over completely.

At this point **GRUB is no longer in memory**. It has done its job.

The Linux kernel now initialises its core subsystems in a fixed order:

```
Kernel early boot
   │
   ├── Detect CPU (vendor, features, cores, NUMA topology)
   ├── Parse kernel command line
   ├── Set up initial memory map (detect usable RAM)
   ├── Build the initial page tables
   ├── Enable virtual memory (MMU)
   ├── Set up the kernel stack
   ├── Initialise the Interrupt Descriptor Table (IDT)
   ├── Set up the timer interrupt (needed for preemption)
   ├── Initialise the scheduler
   ├── Initialise memory management (buddy allocator, slab)
   ├── Initialise the VFS (Virtual File System)
   ├── Register root filesystem
   └── Mount the initramfs as a temporary root
```

You can see kernel messages with:

```bash
dmesg
```

Or during boot, remove `quiet` from the kernel command line.

The tail end of `dmesg` during boot typically shows the timestamped hand-off:

```text
[    0.000000] Linux version 5.15.0-91-generic (buildd@lcy02-amd64-051)
[    0.000000] Command line: BOOT_IMAGE=/boot/vmlinuz-5.15.0-91-generic root=UUID=8f3e... ro
[    3.412345] systemd[1]: Startup finished in 2.1s (kernel) + 1.3s (userspace) = 3.4s.
```

---

## Stage 6 — The initramfs (initrd)

The kernel at this point has almost no device drivers loaded. It cannot read your SSD yet.

But it must somehow find and mount the real root filesystem, which lives *on* that SSD.

This is a chicken-and-egg problem:

```
Kernel needs a driver to read the SSD
Driver lives on the SSD
SSD cannot be read without the driver
```

### The Solution

The **initramfs** (initial RAM filesystem) is a small compressed archive containing:

- A minimal root filesystem
- BusyBox (a tiny multi-call binary with basic tools)
- Essential device drivers (storage, filesystems)
- The `init` script that performs the real root mount

```bash
# Inspect the initramfs contents
ls -lh /boot/initrd.img-$(uname -r)

# You must be root, and the tools vary by distribution
lsinitramfs /boot/initrd.img-$(uname -r) | head -40
```

The kernel **unpacks the initramfs into a tmpfs in RAM** and executes its `init` as the first userspace process.

That `init` then:

1. Loads the modules needed to read the real root device
2. Waits for the device to appear
3. Mounts the real root filesystem
4. Removes the initramfs from RAM to free memory
5. Calls `switch_root` to hand control to the real `/sbin/init`

```c
/* In kernel/switch_root.c — conceptually */
int __init prepare_namespace(void)
{
    mount_root();                 /* mount the real / */
    if (ramdisk_execute_command) { /* old initramfs /init */
        run_init_process(ramdisk_execute_command);
        panic("Failed to execute /init");
    }
    /* switch_root() lives in do_mounts_initrd() */
    switch_root(new_root->mnt, old_root->mnt);
    return 0;
}
```

### Why the initramfs Matters

| Reason | Explanation |
|---|---|
| **Encrypted disks** | Decrypting LUKS volumes needs userspace tools and a key |
| **RAID arrays** | Assembling md arrays needs a userspace helper |
| **LVM** | Activating volume groups needs `lvm2` |
| **Root on network/iSCSI** | Network drivers must load before root is reachable |
| **Modern filesystems** | Btrfs, XFS, ZFS device-mapper modules are large |

Without the initramfs, none of these would be mountable at boot.

---

## Stage 7 — systemd Becomes PID 1

After `switch_root`, the kernel executes the real init program.

On modern Linux distributions this is **systemd**, located at `/sbin/init` (usually a symlink to `/lib/systemd/systemd`).

systemd becomes **PID 1** — the first and last process.

### What PID 1 Means

```
PID 1 is special:

- It is the ancestor of every other process
- It is never killed by signals in the normal way
- The kernel gives it special privileges, e.g.:
    - If a normal process dies, init reaps it
    - If PID 1 dies, the kernel panics ("Attempted to kill init!")
```

### systemd's Boot Work

systemd executes a chain of **targets**:

```text
default.target
      │
      ├── basic.target
      │      ├── sysinit.target
      │      │      ├── mount units      (/, /home, /boot, swap)
      │      │      ├── swap.target
      │      │      ├── local-fs.target
      │      │      ├── device units     (udev)
      │      │      └── ...
      │      └── sockets.target
      │
      ├── multi-user.target
      │      ├── getty@tty1.service
      │      ├── sshd.service
      │      ├── NetworkManager.service
      │      ├── cron.service
      │      └── ... every service you enabled
      │
      └── graphical.target
             └── display-manager.service  (GDM, SDDM, LightDM)
```

To see the whole tree and measure boot time:

```bash
systemd-analyze
systemd-analyze blame
systemd-analyze critical-chain
```

Example output:

```text
$ systemd-analyze blame

  1.204s NetworkManager-wait-online.service
  812ms systemd-udev-settle.service
  431ms snapd.service
   98ms getty@tty1.service
   42ms user@1000.service
```

`systemd-analyze` is the single most useful boot-debugging tool on Linux.

---

## Stage 8 — Login and Shell

The final step is a **login prompt** appearing on your terminal.

This is provided by a **getty** process (one per virtual terminal, `tty1` through `tty6`):

```text
Ubuntu 22.04.3 LTS my-laptop tty1

my-laptop login: alice
Password:
Last login: Mon Oct  9 14:22:31 2023
alice@my-laptop:~$ 
```

The getty is running `agetty`, which invokes `login`, which authenticates the user, then starts the user's **shell** — usually `bash`.

Now the CPU is running your commands, and the OS is fully in control.

---

# Full Boot Flow Diagram

```text
   ┌──────────┐
   │  Press   │
   │  Power   │
   └────┬─────┘
        ▼
   ┌─────────────────────────────────────────┐
   │ Firmware (UEFI/BIOS)                    │
   │  • POST — test CPU, RAM, devices        │
   │  • Select boot device from boot order    │
   │  • Load 1st sector (MBR or ESP)         │
   └────────────────┬────────────────────────┘
                    ▼
   ┌─────────────────────────────────────────┐
   │ GRUB2 Bootloader                        │
   │  • Show menu, read /boot/grub/grub.cfg  │
   │  • Load vmlinuz-<version>               │
   │  • Load initrd.img-<version>            │
   │  • Pass root=, quiet, ... to kernel     │
   │  • Jump to kernel entry point           │
   └────────────────┬────────────────────────┘
                    ▼
   ┌─────────────────────────────────────────┐
   │ Linux Kernel (vmlinuz)                  │
   │  • Decompress kernel image              │
   │  • Detect CPU, RAM, NUMA nodes          │
   │  • Build page tables, enable MMU        │
   │  • Init IDT, exceptions, timer          │
   │  • Init scheduler, memory, VFS          │
   │  • Unpack initramfs into tmpfs          │
   └────────────────┬────────────────────────┘
                    ▼
   ┌─────────────────────────────────────────┐
   │ initramfs /init (in RAM)                │
   │  • Load disk/filesystem/crypto modules  │
   │  • Mount real root filesystem           │
   │  • Free the initramfs memory            │
   │  • switch_root to real /                │
   └────────────────┬────────────────────────┘
                    ▼
   ┌─────────────────────────────────────────┐
   │ systemd (PID 1)                         │
   │  • sysinit.target: mounts, udev, sockets│
   │  • basic.target: paths, timers, swap    │
   │  • multi-user.target: your services     │
   │  • getty@tty1 → login → bash            │
   └────────────────┬────────────────────────┘
                    ▼
   ┌─────────────────────────────────────────┐
   │ Login Shell                             │
   │  alice@my-laptop:~$                     │
   └─────────────────────────────────────────┘
```

---

# Dual Boot and the Boot Partition

On a dual-boot machine, the bootloader is shared, and the firmware's boot order decides which OS starts.

GRUB handles this with entries in `/etc/grub/grub.cfg` or `/boot/grub/grub.d/`.

```bash
# See available GRUB entries
grep menuentry /boot/grub/grub.cfg

# Change the default timeout
sudoedit /etc/default/grub
# GRUB_TIMEOUT=5
sudo update-grub
```

A typical EFI system partition layout:

```bash
sudo efibootmgr -v
```

```text
BootCurrent: 0002
Timeout: 3 seconds
BootOrder: 0002,0001,0000
Boot0002* ubuntu HD(1,GPT,8f3e2a1b-...)/File(\EFI\ubuntu\shimx64.efi)
Boot0001* Windows Boot Manager HD(1,GPT,c3d4e5f6-...)/File(\EFI\Microsoft\Boot\bootmgfw.efi)
Boot0000* Fedora HD(1,GPT,a1b2c3d4-...)/File(\EFI\fedora\shimx64.efi)
```

> [!TIP]
> Secure Boot (UEFI) requires the bootloader to be signed by a trusted key.
> Linux distributions ship signed shims (`shimx64.efi`) that chain to
> GRUB, so Secure Boot can stay enabled.

---

# Booting Without a Bootloader (initramfs only)

For deeply embedded Linux devices, you can sometimes skip GRUB entirely.

The kernel can be loaded directly by the firmware using an EFI stub:

```bash
# Inspect the EFI stub embedded in the kernel image
objdump -h /boot/vmlinuz-5.15.0-91-generic | grep -i efi
```

The kernel itself then contains a small EFI application header, so UEFI can execute it directly. This is called the **EFI stub**.

It is used by:

- systemd-boot
- UKI (Unified Kernel Images)
- Fully encrypted root setups where GRUB needs to be trusted anyway

---

# Observing the Boot Process in Linux

Here is a practical toolkit.

## Kernel messages

```bash
dmesg                       # all kernel ring-buffer messages
dmesg | tail -50            # most recent 50
dmesg -T                    # human-readable timestamps
dmesg -l err,warn            # only errors and warnings
dmesg -w                    # follow live (like tail -f)
```

## Boot timing

```bash
systemd-analyze                    # summary
systemd-analyze time               # total boot time
systemd-analyze blame               # slowest units
systemd-analyze critical-chain      # the critical path
systemd-analyze blame | head       # top 10 slowest
```

## Units and dependencies

```bash
systemctl list-units --type=target
systemctl list-unit-files --state=enabled
systemctl status systemd-udevd.service
systemctl cat multi-user.target     # see the target's contents
systemctl show sshd.service         # all properties
```

## Failing a unit to see dependency failures

```bash
systemctl start sshd
systemctl status sshd
journalctl -u sshd -b               # logs for this boot
journalctl -xe                      # last events, with explanations
```

## Emergency and rescue targets

If the system will not boot normally:

```bash
# At the GRUB menu, press 'e' to edit the entry and append:
systemd.unit=rescue.target      # minimal system, root shell
systemd.unit=emergency.target   # even more minimal

# Or, from a running system:
sudo systemctl isolate rescue.target
sudo systemd-analyze            # see what is slow
```

---

# Boot Performance Optimisation

Once you can measure boot time, you can improve it.

| Technique | Effect | Risk |
|---|---|---|
| Remove `quiet splash` while debugging | See real messages | None (noise) |
| `systemd-analyze blame` → disable slow units | Often large gains | Feature loss |
| Reduce `GRUB_TIMEOUT=0` | Saves 5 seconds | Cannot select kernel easily |
| `systemctl mask` unneeded services | Fewer units to start | Breaks that feature |
| `systemd-analyze critical-chain` | Finds the actual bottleneck | None |
| Enable `systemd-udevd` rules tuning | Fewer udev settles | May miss hotplug events |
| Use an SSD instead of HDD | 5–10× faster overall | Cost |

A real example of a slow boot:

```text
$ systemd-analyze blame
        12.481s NetworkManager-wait-online.service
         8.204s systemd-udev-settle.service

$ systemd-analyze critical-chain
NetworkManager-wait-online.service
→ systemd-udev-settle.service
```

Both were waiting for a device that will never appear (a docked network adapter that was unplugged).

Disabling `NetworkManager-wait-online` cut 12 seconds off boot time with no functional loss on a laptop.

---

# Common Misconceptions

### ❌ "The bootloader loads the whole operating system."

Incorrect.

The bootloader only loads two files: the kernel image and the initramfs.

Everything else — all services, all drivers, your desktop environment — is started later by `systemd`.

---

### ❌ "The kernel starts first, then the bootloader."

Incorrect.

The order is: firmware → bootloader → kernel.

The bootloader is a normal program, but it runs before the kernel exists in memory. The kernel is what it loads.

---

### ❌ "initramfs is just an optimisation."

Incorrect.

The initramfs solves a fundamental chicken-and-egg problem: the kernel needs storage drivers to read the disk, but those drivers live on the disk.

It is *mandatory* for any non-trivial root filesystem — especially with encryption, LVM, or RAID.

---

### ❌ "PID 1 is just the first process."

Incorrect.

PID 1 has special kernel-level privileges. It is the parent of all processes, reaps orphans, and the kernel halts if it dies. A normal process being killed has no such effect.

---

### ❌ "systemd and init are interchangeable."

Incorrect.

`init` (SysVinit) was a single process that executed scripts sequentially.

`systemd` is a parallel, dependency-aware, socket-activated service manager. They behave very differently under failure and under heavy service counts.

---

# Interview Questions

### Basic

- What happens when you press the power button?
- What is the role of the bootloader?
- What is GRUB?
- What is initramfs and why is it needed?

### Intermediate

- Explain the difference between MBR and UEFI boot.
- What does `switch_root` do and why is it necessary?
- What is the chicken-and-egg problem that initramfs solves?
- What is PID 1 and why is it special?

### Advanced

- How does Secure Boot interact with GRUB on Linux?
- How would you diagnose a system that hangs during boot?
- What is a Unified Kernel Image (UKI) and how does it improve boot security?
- How does systemd's dependency-based parallel startup reduce boot time compared to SysVinit?
- Why does `systemd-analyze critical-chain` sometimes report a different result from `blame`?

---

# University Exam Notes

### Definition

The boot process is the sequence of stages a computer follows from power-on until the operating system hands control to a user login. On Linux this is: firmware (UEFI/BIOS) → bootloader (GRUB2) → kernel (vmlinuz) → initramfs → systemd (PID 1) → login shell.

### Frequently Asked Questions

- Explain the Linux boot process step by step.
- What is the role of the bootloader?
- What is initramfs? Why is it required?
- What is GRUB? What files does it load?
- What is PID 1? Why is it important?
- What is `switch_root`?
- Explain UEFI vs legacy BIOS boot.
- What is Secure Boot and how does Linux support it?

---

# Key Takeaways

- The CPU cannot execute anything until firmware runs and loads a bootloader.
- The bootloader (GRUB2) loads only two things: the kernel image and the initramfs.
- The kernel initialises core subsystems but cannot read disks without drivers.
- The initramfs solves this by providing a temporary root with just enough drivers to mount the real root.
- `systemd` becomes PID 1 and starts services in parallel based on dependencies.
- The final step is a getty providing a login prompt, then the user's shell.
- `dmesg`, `systemd-analyze`, and `journalctl` are the primary boot-debugging tools.

---

# References

- Operating System Concepts — Silberschatz
- Modern Operating Systems — Andrew S. Tanenbaum
- The Linux Bootloader — GRUB2 documentation
- systemd.io — official systemd documentation
- `man 8 systemd`, `man 5 initramfs`, `man 7 hier7` (Linux man-pages)
