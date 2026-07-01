# CPU Modes (User Mode & Kernel Mode)

> [!NOTE]
> **Module:** Module I – Introduction
>
> **Difficulty:** ⭐⭐⭐☆☆
>
> **Prerequisites:**
> - What is an Operating System
> - Kernel
> - Von Neumann Architecture

---

# Learning Objective

After reading this note, you should be able to:

- Understand why CPU modes are required.
- Differentiate between User Mode and Kernel Mode.
- Explain how CPU modes improve system security and stability.
- Understand privileged instructions.
- Explain why applications cannot directly access hardware.
- Build the foundation required for System Calls, Interrupts and Process Management.

---

# Why Do We Need CPU Modes?

## The Problem

Imagine there were **no restrictions** on what a program could do.

Suppose you download a simple calculator application.

Instead of only performing calculations, it could:

- Format your SSD.
- Read your passwords.
- Shut down the computer.
- Disable antivirus software.
- Modify another application's memory.
- Crash the entire operating system.

Clearly, this would make computers extremely unsafe.

Modern operating systems solve this problem by introducing **different privilege levels**.

The CPU itself enforces these privilege levels.

These privilege levels are called **CPU Modes**.

---

# Real World Analogy

Imagine a bank.

There are two types of people:

### Customers

Customers can:

- Deposit money
- Withdraw money
- Check their balance

But they **cannot**

- Open the bank vault
- Modify customer records
- Change bank policies

---

### Bank Manager

The manager can:

- Access the vault
- Modify records
- Approve loans
- Control every department

The manager has much higher privileges.

Similarly,

Applications are like customers.

The **Kernel** is like the bank manager.

Only the Kernel is trusted enough to access critical system resources.

---

# Intuition

Think of the CPU as a highly secure building.

It has two floors.

```
+------------------------------+
|        User Mode             |
| Applications execute here    |
+------------------------------+

              │
      System Call / Interrupt

              ▼

+------------------------------+
|       Kernel Mode            |
| Operating System executes    |
+------------------------------+
```

Applications spend almost all their time in **User Mode**.

Whenever they need to perform a privileged operation, they request help from the Kernel.

The CPU temporarily switches to **Kernel Mode**, performs the operation, and then returns to User Mode.

---

# What are CPU Modes?

CPU Modes are **different privilege levels provided by the processor** to control what instructions a program is allowed to execute.

Modern operating systems primarily use two CPU modes:

1. **User Mode**
2. **Kernel Mode**

These modes ensure that applications cannot accidentally or intentionally damage the operating system.

---

# User Mode

## Definition

User Mode is the restricted execution mode in which normal applications run.

Applications running in User Mode have limited privileges and cannot directly access hardware or execute sensitive CPU instructions.

Examples include:

- Google Chrome
- VS Code
- Spotify
- Discord
- Games
- Java Applications
- Python Programs

---

## Characteristics

- Limited permissions
- Cannot access hardware directly
- Cannot modify kernel memory
- Cannot execute privileged instructions
- Must use System Calls to request OS services

---

## What Can User Mode Programs Do?

✔ Execute normal instructions

✔ Perform calculations

✔ Allocate memory (through OS services)

✔ Read and write files (using system calls)

✔ Create processes (through system calls)

✔ Communicate over networks

---

## What Can User Mode Programs NOT Do?

❌ Disable interrupts

❌ Modify page tables

❌ Directly communicate with hardware

❌ Access kernel memory

❌ Change CPU scheduling

❌ Execute privileged instructions

These restrictions prevent applications from crashing the operating system.

---

# Kernel Mode

## Definition

Kernel Mode is the privileged execution mode in which the Operating System's kernel runs.

In this mode, the CPU grants unrestricted access to hardware resources and privileged instructions.

---

## Characteristics

- Highest privilege level
- Complete hardware access
- Can execute privileged instructions
- Can access any memory location
- Manages CPU, memory, files, devices and processes

---

## Responsibilities of Kernel Mode

The Kernel performs operations such as:

- Process Scheduling
- Memory Management
- File System Management
- Device Driver Management
- Interrupt Handling
- System Call Handling
- Security Enforcement

Without Kernel Mode, the Operating System would not be able to manage the computer.

---

# User Mode vs Kernel Mode

| Feature | User Mode | Kernel Mode |
|----------|-----------|-------------|
| Privilege Level | Low | Highest |
| Hardware Access | ❌ No | ✅ Yes |
| Execute Privileged Instructions | ❌ No | ✅ Yes |
| Memory Access | Limited | Complete |
| Used By | Applications | Operating System |
| Security | High | Critical |
| Crash Impact | Usually crashes only one application | Can crash the entire system |

---

# Why Can't Applications Access Hardware Directly?

Consider a simple Java program.

```java
FileReader reader = new FileReader("notes.txt");
```

It looks like Java is directly reading the file.

But that is **not** what actually happens.

The real flow is:

```
Java Application

        │

        ▼

Java Virtual Machine (JVM)

        │

        ▼

Operating System

        │

        ▼

Kernel

        │

        ▼

Storage Device (SSD/HDD)
```

The application **never communicates directly with the SSD**.

Instead, it requests the Operating System, which safely performs the operation.

This design provides:

- Security
- Stability
- Hardware independence
- Controlled resource sharing

---

# Privileged Instructions

Some CPU instructions are considered extremely sensitive.

Only the Operating System is allowed to execute them.

These are called **Privileged Instructions**.

If a User Mode program attempts to execute one, the CPU immediately raises an exception.

---

## Examples of Privileged Instructions

### Memory Management

- Modify page tables
- Change memory mappings

---

### Interrupt Management

- Enable interrupts
- Disable interrupts

---

### Device Access

- Read hardware registers
- Write hardware registers
- Access I/O ports

---

### Process Management

- Perform context switching
- Change CPU scheduling
- Modify process priorities

---

### Security Operations

- Change privilege levels
- Load kernel modules
- Modify access permissions

---

# Why Are Privileged Instructions Necessary?

Imagine if any application could execute:

```
Disable All Interrupts
```

The operating system would lose control of the CPU.

Or suppose an application could directly change memory mappings.

It could read another application's passwords, banking information, or personal data.

Therefore, privileged instructions can only be executed in **Kernel Mode**.

---

# Engineering Insight

> [!TIP]
> CPU Modes are not implemented by the Operating System alone.
>
> They are **hardware features provided by the processor** (Intel, AMD, ARM, etc.).
>
> The Operating System simply makes use of these hardware-supported privilege levels to enforce security.

---

# Common Misconceptions

### ❌ User Mode is slower than Kernel Mode.

Not necessarily.

User Mode is simply more restricted. Performance depends on the operation being performed.

---

### ❌ Applications directly access hardware.

Incorrect.

Applications request services from the Kernel using **System Calls**.

---

### ❌ Kernel Mode is only used during system startup.

Incorrect.

The CPU switches into Kernel Mode thousands (or even millions) of times every second while handling system calls, interrupts, and hardware events.

---

## Key Takeaways (Part 1)

- CPU Modes are hardware-supported privilege levels.
- Modern operating systems primarily use **User Mode** and **Kernel Mode**.
- Applications run in User Mode with restricted permissions.
- The Operating System runs in Kernel Mode with full privileges.
- Privileged instructions can only execute in Kernel Mode.
- CPU Modes are fundamental to system security and stability.

---

## Process Control Block (PCB) - A Brief Introduction

> [!NOTE]
> The **Process Control Block (PCB)** is a core data structure used by the Operating System to manage processes.
>
> We will study it in detail in **Module II – Process Management**. Here, we only introduce it because it plays an important role during **mode switching** and **context switching**.

Whenever the CPU switches from one process to another, the Operating System must save the current state of the running process.

This information is stored inside a **Process Control Block (PCB)**.

Without a PCB, the Operating System would not know:

- Which instruction the process was executing.
- What values were stored in the CPU registers.
- How much memory the process was using.
- Which files the process had opened.

After storing this information, the Operating System loads the PCB of another process and resumes its execution.

### Context Switching

```text
                Process A Running
                       │
                       ▼
          Interrupt / Time Quantum Ends
                       │
                       ▼
          Save CPU State into PCB(A)
                       │
                       ▼
         Load CPU State from PCB(B)
                       │
                       ▼
                Process B Running
```

### What Does a PCB Store?

A PCB typically contains:

- Process ID (PID)
- Process State
- Program Counter (PC)
- CPU Registers
- Scheduling Information
- Memory Information
- Open Files
- I/O Status Information

> [!TIP]
> Think of a PCB as the **identity card and progress report** of a process. It contains everything the Operating System needs to pause and later resume that process exactly where it left off.