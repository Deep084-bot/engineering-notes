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