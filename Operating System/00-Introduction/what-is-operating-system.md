# What is an Operating System?

## Why do we need OS?
Imagine there is no os, and you bought a new laptop. It has a powerful CPU, huge RAM and SSD, a keyboard, mouse, speakers, wifi card. Everything looks perfect. But wehen we press power button, nothing happens. The hardware exists, but none of the components know how to work together.
We can also think it as a hotel where there are. many guests willing to get a room for stay but there's no hotel manager, then there would be a huge mishap. 
- Guests would enter any room they like. 
- Multiple guests could claim the same room.
- Nobody would know which room is available.
- Housekeeping wouldn't know which room to clean.
- Security wouldn't know who is authorized to enter.
- The hotel would quickly become chaotic.
An OS plays the role of the hotel manager.

We have very limited resources in our computer and at the same time there are many applications competing for those resources, so we need OS to solve this problem of allocation.
So we can think of OS as the manager of the entire computer.

Instead of every application talking directly to hardware, all communication goes through the Operating System.

              User
                │
                ▼
        +----------------+
        | Applications   |
        +----------------+
                │
                ▼
      +--------------------+
      | Operating System   |
      +--------------------+
        │    │      │
        │    │      │
        ▼    ▼      ▼
      CPU   RAM   Storage
              │
        Keyboard, Mouse,
        Display, Network

## What is OS
An operating system acts as an intermediary between the computer hardware and the user. It prevents the user from directly accessing the hardware and manages all system resources.
- It provide an environment in which a user can execute programs conveniently and efficiently.
- Operating system (OS) is a program that runs at all times on a computer.
- It assigns resources such as memory, processors, and input/output devices to processes that need them.

## Real World Applications
Every computing device relies on an Operating System:
- Personal computers (Windows, Linux, macOS)
- Smartphones (Android, iOS)
- Servers running websites and databases
- Cloud platforms
- Supercomputers
- ATMs
- Cars
- Smart TVs
- Medical devices
- Embedded systems

## Components of OS
There are two basic components of an Operating System.
- **Shell** is the outermost layer of the Operating System and handles user interaction. It interprets input for the OS and handles the output from the OS.
- **Kernel** is the core component of the operating system. The kernel is the primary interface between the Operating system and Hardware.

## Diagram

```text
+--------------+
| Applications |
+--------------+
       |
       v
+------------------+
| Operating System |
+------------------+
       |
       v
+------------------+
|     Hardware     |
+------------------+
```

## Types of Operating Systems

Operating Systems can be classified based on **how they execute programs**, **how they utilize hardware**, and **the environment in which they operate**.

> **Note:** An operating system can belong to more than one category. For example, Linux is a multitasking, multiprocessing, and network-capable operating system.

---

### 1. Batch Operating System

A Batch Operating System executes a collection of jobs (called a **batch**) without requiring user interaction during execution.
Jobs are submitted to the system, grouped together, and executed one after another. Similar to non multi-tasking OS ie one task at a time.

#### Characteristics

- No direct user interaction
- Jobs are executed sequentially
- High throughput for repetitive tasks
- Long turnaround time

#### Advantages

- Efficient for large repetitive workloads
- Minimal CPU idle time

#### Disadvantages

- No immediate user feedback
- Difficult to debug jobs
- Not suitable for interactive applications

**Example:** Early IBM Mainframe Systems

---

### 2. Multiprogramming Operating System

A Multiprogramming Operating System keeps multiple programs in memory simultaneously.
When one program waits for I/O, the CPU switches to another program instead of remaining idle.

#### Characteristics

- Multiple programs reside in memory
- Better CPU utilization
- Improves overall throughput

#### Example

Early UNIX Systems

---

### 3. Multitasking (Time-Sharing) Operating System

A Multitasking Operating System allows multiple tasks to execute seemingly at the same time by rapidly switching the CPU among them.
Each task receives a small time interval called a **time quantum**.
This creates the illusion that all applications are running simultaneously.

#### Characteristics

- Interactive
- Quick response time
- Supports multiple users or multiple applications

#### Examples

- Windows
- Linux
- macOS

---

### 4. Multiprocessing Operating System

A Multiprocessing Operating System uses two or more CPUs (or processor cores) to execute processes simultaneously.
This increases computational power and improves system reliability.

#### Characteristics

- Parallel execution
- Higher throughput
- Better fault tolerance

#### Examples

- Linux
- Windows Server

---

### 5. Distributed Operating System

A Distributed Operating System connects multiple independent computers through a network and makes them appear as a single unified system.
Each computer has its own processor and memory but cooperates with others to share resources.

#### Characteristics

- Resource sharing
- Remote execution
- Scalability
- Fault tolerance

#### Examples

- Amoeba
- Plan 9

---

### 6. Network Operating System (NOS)

A Network Operating System manages communication and resource sharing between multiple computers connected through a network.
Unlike a Distributed Operating System, users are aware that multiple computers exist.

#### Characteristics

- File sharing
- Printer sharing
- User authentication
- Centralized administration

#### Examples

- Windows Server
- Novell NetWare
- UNIX Server

---

### 7. Real-Time Operating System (RTOS)

A Real-Time Operating System guarantees that tasks are completed within a specified time limit.
The correctness of the system depends not only on the result but also on when the result is produced.

#### Characteristics

- Predictable response time
- High reliability
- Deterministic scheduling

#### Applications

- Air Traffic Control
- Medical Equipment
- Robotics
- Missile Guidance Systems
- Automotive Control Systems

#### Types

- Hard Real-Time OS
- Soft Real-Time OS