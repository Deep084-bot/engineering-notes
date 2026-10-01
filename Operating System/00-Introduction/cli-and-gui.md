# CLI and GUI: User Interfaces in Linux

> [!NOTE]
> **Module:** Module I – Introduction
>
> **Difficulty:** ⭐⭐☆☆☆
>
> **Prerequisites:**
> - System Calls
> - OS Services and Functions
>
> **Scope:** Linux and C only. No Windows or Java.

---

# Learning Objective

After reading this note, you should be able to:

- Explain what a user interface layer is and why the OS needs one.
- Describe how a command line interface works, from keystroke to process.
- Describe how a graphical user interface works on Linux.
- Explain what a TTY and a PTY are.
- Explain what a display server and window manager do.
- Compare CLI and GUI on resource usage, scriptability and suitability.
- Explain how an application detects that it is running in a terminal.

---

# Why Do We Need a User Interface?

## The Problem

The kernel exposes system calls: `read`, `write`, `open`, `mmap`.

None of these understand the concept of a "window", a "cursor", or a "menu".

But a human being cannot type `mmap` and expect a picture.

There is a large gap between the machine's vocabulary and the human's:

```text
Human wants:                  Machine offers:

"show me the files"     →     getdents64()
"scroll down"           →     read() from stdin
"click that button"     →     read() from /dev/input/event*
"show this in a window" →     mmap() a shared framebuffer
```

The user interface layer is the machinery that translates between them.

---

# Real World Analogy

A building has two ways to be entered.

- The **front desk**: a human talks to a person. The person uses the internal
  system on your behalf. Convenient, flexible, needs staffing.
- The **loading bay**: you drive up, follow markings, and use the machinery
  directly. No staffing, scriptable, unforgiving of mistakes.

The front desk is a GUI.

The loading bay is a CLI.

A GUI is a *middle layer* that translates. A CLI is a *direct path* that
demands you speak the machine's language.

---

# Intuition

Both interfaces end at the same place: the terminal device.

```
   ┌──────────────────────┐        ┌──────────────────────┐
   │        GUI           │        │        CLI           │
   │  mouse → compositor  │        │  keystrokes → tty   │
   │  window → app        │        │  app → tty          │
   └──────────┬───────────┘        └──────────┬───────────┘
              │                               │
              │        /dev/pts/N  (a PTY)     │
              └───────────────┬───────────────┘
                              │
                     ┌────────▼────────┐
                     │  tty layer +    │
                     │  line discipline│
                     └────────┬────────┘
                              │
                     ┌────────▼────────┐
                     │  file system    │
                     │  read()/write() │
                     └─────────────────┘
```

A GUI application and a terminal program both ultimately read and write bytes
on a terminal device. The GUI is a program that draws pictures on top of that.

---

# Part 1 — The Command Line Interface

## Definition

A **CLI** is a user interface where the user communicates with the computer by
typing text commands interpreted by a **shell**.

## The Chain of Events

Type `ls` and press Enter:

```text
 1. You press Enter. The keyboard driver sets a key event.
 2. The tty line discipline buffers it as a byte.
 3. The terminal emulator (or real tty) has \n buffered, so it echoes.
 4. The byte \n is written to the shell's file descriptor 0 (stdin).
 5. The shell was blocked in read(0, ...). It wakes up.
 6. The shell parses the line, resolves "ls" to /bin/ls via PATH.
 7. The shell fork()s.  →  the child execs /bin/ls.
 8. The parent shell waitpid()s, blocking.
 9. ls writes directory bytes to fd 1. The shell does not read them;
    they go straight to the terminal.
10. ls exits. The shell reaps it and prints a new prompt.
```

```bash
# Step 6: PATH resolution
echo $PATH
# /usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

# Step 7/8: confirm from the shell's point of view
strace -f -e trace=execve,clone,wait4 ls
```

```text
execve("/bin/ls", ["ls"], ["PATH=...", "HOME=..."]) = 0
+++ exited with 0 +++
```

## Shells in Linux

| Shell | Common in | Character |
|---|---|---|
| `bash` | everywhere | the default; most documentation assumes it |
| `zsh` | macOS-ish workflows | powerful completion, themes |
| `fish` | interactive | friendly, but **not** POSIX sh compatible |
| `dash` | `/bin/sh` on Debian/Ubuntu | minimal, very fast, POSIX |
| `ksh` | enterprise / legacy | Korn shell |
| `csh`, `tcsh` | legacy | C-like syntax |
| `busybox sh` | embedded | tiny, single binary |

```bash
ls -l /bin/sh          # often a symlink to dash or bash
echo $0                # your current shell
cat /etc/shells        # valid shells listed by the system
```

## CLI Strengths

| Strength | Why |
|---|---|
| **Scriptable** | Input/output redirection and pipes are a built-in dataflow language. |
| **Low resource use** | No framebuffer, no compositor, no fonts. |
| **Composability** | Small tools combine into large behaviour: `find . -name '*.c' \| xargs grep TODO`. |
| **Remote-friendly** | Works over a 300-bit/s link. |
| **Automatable** | No mouse, no focus, no race with the user's clicks. |

## CLI Strengths — a Real Pipeline

```bash
# Find C files, search them, show matching lines with file names
find /usr/src -name '*.c' -print0 \
  | xargs -0 grep -n 'TODO'

# Count distinct syscall names a program makes
strace -c -e trace=all ./a.out 2>&1 | head -20

# Sum the RSS of every firefox process, in KB
ps -o rss= -C firefox | awk '{s+=$1} END {print s " KB"}'
```

Each stage is a separate process connected by a pipe. No shell feature beyond
"start a process, connect stdin to the previous stdout" is required.

## CLI Weaknesses

| Weakness | Why |
|---|---|
| **Learnability** | Commands have flags, not buttons. |
| **No discoverability** | You must already know the name of the thing you want. |
| **Poor for spatial tasks** | "Move this window slightly left" is a terrible sentence. |
| **Error-prone** | `rm -rf` with a bad variable is catastrophic. |

---

# Part 2 — The Graphical User Interface

## Definition

A **GUI** is a user interface where the user interacts with graphical elements —
windows, icons, menus, buttons — drawn by the system and manipulated with a
pointing device.

## The Linux GUI Stack

Linux has no single GUI. It is a stack of independent layers, each replaceable.

```text
┌─────────────────────────────────────────────────────┐
│  Desktop Environment:  GNOME, KDE Plasma, XFCE      │
│  (panels, settings, file manager, theming)          │
├─────────────────────────────────────────────────────┤
│  Toolkit:  GTK (C)      Qt (C++)   wlroots clients │
│  (drawing widgets, buttons, menus)                   │
├─────────────────────────────────────────────────────┤
│  Window Manager / Compositor: Mutter, KWin, Sway   │
│  (window placement, decoration, animation)           │
├─────────────────────────────────────────────────────┤
│  Display Server:  X.Org        |   Wayland          │
│  (routes input to clients, owns the screen)         │
├─────────────────────────────────────────────────────┤
│  Graphics Driver:  modesetting, amdgpu, i915, nvidia│
├─────────────────────────────────────────────────────┤
│  Kernel: DRM/KMS subsystem, /dev/dri/card0, evdev   │
└─────────────────────────────────────────────────────┘
```

Each layer only talks to the layer below through a documented interface.
This is why you can swap the desktop environment without recompiling anything.

## X.Org vs Wayland

| Aspect | X.Org | Wayland |
|---|---|---|
| Model | Clients may read each other's memory | Clients are isolated; no client sees another |
| Compositing | Limited / optional | Mandatory |
| Input | Legacy and protocol-specific | Unified, via libinput |
| Security | Effectively none | Isolated by default |
| Screen tearing | Common | Eliminated by the compositor |
| Protocol | X11, decades old | Modern, `wlr`/`wlroots` for compositors |
| GPU accel | Varies | Mandatory |

```bash
# Which display server is running?
echo $XDG_SESSION_TYPE
# x11   or   wayland

# The Wayland compositor socket
echo $WAYLAND_DISPLAY
# wayland-1

# X11 clients still run on Wayland via XWayland
ps aux | grep XWayland
```

> [!TIP]
> XWayland exists so that programs that only speak X11 keep working.
> This is a good example of a compatibility shim — a pattern you will
> meet in the bootloader (GRUB → shim) too.

## What the Display Server Actually Does

```text
  Keyboard / Mouse
        │
        ▼
  evdev:  /dev/input/event0   (raw input events)
        │
        ▼
  libinput  (maps device models to a simple event stream)
        │
        ▼
  Display server  (X server or Wayland compositor)
        │
        ├──▶ routes key event to focused client
        └──▶ clients render into their own buffers
                    │
                    ▼
              Compositor  ← scans out one final frame
                    │
                    ▼
                  GPU ──▶ monitor
```

```bash
# The DRM device the graphics driver uses
ls -l /dev/dri/

# Inspect display configuration (Wayland)
wayland-info
# X11:
xrandr
```

## Writing a GUI in C

In C, a GUI is a program that opens a connection to the display server and
loops on events.

A minimal **Xlib** program (C, X.Org):

```c
#include <X11/Xlib.h>
#include <stdio.h>
#include <stdlib.h>

int main(void)
{
    Display *dpy = XOpenDisplay(NULL);
    if (!dpy) {
        fprintf(stderr, "cannot open display\n");
        return 1;
    }

    int screen = DefaultScreen(dpy);
    Window win = XCreateSimpleWindow(dpy, RootWindow(dpy, screen),
                                    10, 10, 400, 300, 1,
                                    BlackPixel(dpy, screen),
                                    WhitePixel(dpy, screen));

    XStoreName(dpy, win, "C GUI Demo");
    XSelectInput(dpy, win, ExposureMask | KeyPressMask);
    XMapWindow(dpy, win);

    XEvent ev;
    for (;;) {
        XNextEvent(dpy, &ev);
        if (ev.type == KeyPress) {
            break;                      /* any key quits */
        }
        if (ev.type == Expose) {
            XDrawString(dpy, win,
                        DefaultGC(dpy, screen),
                        20, 40, "Hello from C", 14);
        }
    }

    XDestroyWindow(dpy, win);
    XCloseDisplay(dpy);
    return 0;
}
```

```bash
gcc -o cgui cgui.c -lX11
./cgui
```

The structure of every GUI event loop, in any toolkit, is the same:

```text
  open connection  →  create window  →  loop { wait for event; handle event }
```

## Writing a GUI in C with GTK

```c
#include <gtk/gtk.h>

static void on_activate(GtkApplication *app, void *data)
{
    GtkWidget *win = gtk_application_window_new(app);
    gtk_window_set_title(GTK_WINDOW(win), "C GUI");

    GtkWidget *btn = gtk_button_new_with_label("Click me");
    g_signal_connect(btn, "clicked", G_CALLBACK(gtk_window_close), win);
    gtk_widget_set_margin_top(btn, 12);
    gtk_container_add(GTK_CONTAINER(win), btn);

    gtk_window_present(GTK_WINDOW(win));
}

int main(int argc, char **argv)
{
    GtkApplication *app =
        gtk_application_new("com.example.cgui", G_APPLICATION_DEFAULT_FLAGS);
    g_signal_connect(app, "activate", G_CALLBACK(on_activate), NULL);
    return g_application_run(G_APPLICATION(app), argc, argv);
}
```

```bash
gcc -o gtkdemo gtkdemo.c $(pkg-config --cflags --libs gtk4)
```

Note the layered cost: a tiny "hello window" now links GTK, GLib, Pango,
cairo and the display client library. Compare `ldd` output with the CLI
version.

---

# Part 3 — TTY and PTY: The Interface Beneath Both

## Definition

A **TTY** is a character device that represents a physical or emulated terminal.

A **PTY** (pseudo-terminal) is a *pair* of character devices that behaves like
a terminal but has no hardware behind it.

```text
  Real terminal (physical)

     /dev/tty1  ◄──►  keyboard + display (hardware)
          ▲
          │  the shell runs on this
          ▼
        bash


  Emulated terminal (what your GUI terminal window is)

     /dev/pts/3 (master)  ◄──►  terminal emulator (gnome-terminal)
          ▲                                │
          │  bash reads/writes here        │ draws pixels via Wayland/X11
          ▼                                ▼
        bash                          GPU → monitor
```

```bash
# What am I attached to?
tty
# /dev/pts/3

tty
# /dev/tty1

# All ptys currently allocated
ls -l /dev/pts/
```

## Why a PTY Matters

A PTY makes a graphical terminal window look exactly like a real terminal to
the program running inside it.

That means `ls` cannot tell the difference, and neither can `vim`, `top`, or
`gdb`.

The terminal emulator allocates a PTY pair, spawns the shell on the slave
side, and draws whatever appears on the master side.

This is also why **output redirection to a GUI app behaves oddly**:

```bash
firefox > out.txt
# the PTY is not allocated, so the app cannot ask for a terminal
```

## Controlling a Terminal in C

```c
#include <fcntl.h>
#include <stdio.h>
#include <termios.h>
#include <unistd.h>

int main(void)
{
    int fd = open("/dev/tty", O_RDWR);
    if (fd < 0) {
        perror("open /dev/tty");
        return 1;
    }

    struct termios old, raw;

    if (tcgetattr(fd, &old) == -1) {
        perror("tcgetattr");
        return 1;
    }

    raw = old;

    /* Disable canonical mode and echo: we read keys one at a time. */
    raw.c_lflag &= ~(ICANON | ECHO);
    raw.c_cc[VMIN]  = 1;     /* min bytes before read() returns */
    raw.c_cc[VTIME] = 0;     /* no timer */

    if (tcsetattr(fd, TCSANOW, &raw) == -1) {
        perror("tcsetattr");
        return 1;
    }

    puts("raw mode: press q to quit");
    int c;
    while ((c = getchar()) != 'q' && c != EOF) {
        printf("key 0x%02x (%s)\n", c, (c >= 32 && c < 127) ? "printable" : "control");
    }

    tcsetattr(fd, TCSANOW, &old);     /* always restore */
    close(fd);
    return 0;
}
```

```bash
gcc -o rawmode rawmode.c
./rawmode
```

`ICANON` is the reason your terminal buffers a line before delivering it. The
line discipline sits between the tty device and your process:

```text
   keystrokes
       │
       ▼
  ┌──────────────────────────────────────┐
  │  Line discipline  (per tty)          │
  │  • canonical mode: buffer until \n  │
  │  • echo: reflect keys back to screen │
  │  • signal generation: Ctrl-C → SIGINT│
  │  • flow control: Ctrl-S / Ctrl-Q     │
  └──────────────────┬───────────────────┘
                   ▼
              your read(0, ...)
```

```bash
# See the line discipline settings
stty -a
```

```text
speed 38400 baud; rows 24; columns 80; line = 0;
intr = ^C; quit = ^\; erase = ^?; kill = ^U;
lnext = ^V; eof = ^D;
icanon  iexten  echo  echoe  echok  echonl
```

---

# Detecting Whether You Have a Terminal

```c
#include <stdio.h>
#include <unistd.h>

int main(void)
{
    if (isatty(STDOUT_FILENO)) {
        /* human is watching: use colour, boxes, progress bars */
        printf("\033[1;32mVerbose human output\033[0m\n");
    } else {
        /* piped into a file or another program: stay silent and clean */
        printf("machine-readable-output\n");
    }
    return 0;
}
```

```bash
./detect | cat      # piped:  no colour codes
./detect            # tty:    colour codes
```

> [!TIP]
> This is exactly why good CLI tools never emit progress bars or colours when
> stdout is a pipe. `git`, `gcc`, `ls` and `cargo` all call `isatty()`.

---

# CLI vs GUI

| Aspect | CLI | GUI |
|---|---|---|
| Input device | keyboard | keyboard + mouse + touch |
| Resource use | very low | high (GPU, compositor, fonts) |
| Runs over slow network | yes | painful |
| Scriptable | yes, native | awkward (usually scripting the GUI) |
| Discoverability | poor | excellent |
| Accessibility | poor for motor/visual impairment | generally better |
| Determinism | high | low (focus, animation, races) |
| Best for | servers, automation, debugging | design, media, general use |
| Startup time | ~1 ms | ~1 s |
| Concurrency view | `htop` | task switcher |

## They are not competitors

A modern Linux desktop is a stack where each layer is a CLI somewhere:

```text
  GNOME desktop  →  you click things
       │
       ▼
  gnome-terminal  →  you type
       │
       ▼
  bash            →  a scriptable interface
       │
       ▼
  systemd / journalctl  →  logs are text
```

`systemctl status`, `journalctl -f` and `htop` are GUIs in the sense that they
visualise system state, with no pixels involved. Most serious Linux work is
done in the CLI even on machines that have a desktop.

---

# Common Misconceptions

### ❌ "A terminal and a terminal emulator are the same thing."

Incorrect.

The *terminal* is the character device (`/dev/tty1` or `/dev/pts/3`).

The *terminal emulator* is a GUI program that draws those characters as pixels
and forwards your keystrokes into the PTY.

---

### ❌ "The GUI is part of the kernel."

Incorrect.

On Linux, the display server, compositor and desktop are all user-space
programs. The kernel contributes the DRM/KMS subsystem, the input subsystem,
and the `/dev/dri` device nodes — nothing more.

---

### ❌ "Ctrl-C is handled by the shell."

Incorrect.

Your terminal's line discipline generates `SIGINT` for the foreground process
group. The shell only *reports* it afterwards. `Ctrl-C` in a raw-mode program
would simply be byte `0x03`, because signal generation is a line-discipline
feature.

---

### ❌ "A GUI program is written in a different language from a CLI program."

Incorrect.

Both can be C. The difference is which libraries the program links: `ncurses`
versus `X11`/`GTK4`. Nothing in the language prevents a C program from opening
a window.

---

# Interview Questions

### Basic

- What is a CLI? What is a GUI?
- What is a shell, and what does it do?
- What is the role of a window manager?

### Intermediate

- Explain the Linux GUI stack from kernel to desktop environment.
- What is a PTY and why does a GUI terminal need one?
- What does the line discipline do?
- Why do tools check `isatty()`?

### Advanced

- What is XWayland and why does it exist?
- How does the compositor eliminate screen tearing?
- How would you implement a raw-mode key-reading program in C, and what must you restore?
- Why is a GUI more difficult to automate reliably than a CLI?
- What is the security difference between X11 and Wayland, and where is it enforced?

---

# University Exam Notes

### Definitions

- **CLI:** A user interface in which commands are typed as text and interpreted by a shell.
- **GUI:** A user interface in which the user manipulates graphical objects such as windows, icons and menus.
- **Shell:** A command interpreter that reads user input, parses it, and executes the requested programs.
- **PTY:** A pair of character devices that emulates a terminal without physical hardware.
- **Display server:** The system component that mediates between GUI applications and input/output devices.
- **Window manager:** The component responsible for window placement, decoration, focus and stacking.

### Frequently Asked Questions

- Differentiate between CLI and GUI with examples.
- What is a shell? Explain the steps of command execution.
- Explain the GUI architecture of Linux.
- What is a TTY and a PTY? Why is a PTY needed?
- What is the role of a window manager and a display server?
- What is X11? How is it different from Wayland?

---

# Key Takeaways

- The OS exposes only bytes; a user interface turns bytes into human-meaningful interaction.
- A **CLI** interprets typed text via a shell, which forks and execs programs.
- A **GUI** is a layered user-space stack: toolkit → window manager → display server → driver → kernel DRM.
- Both interfaces ultimately end at a terminal device, made uniform by the **PTY**.
- The **line discipline** provides canonical buffering, echo, and signal generation.
- CLI wins on scriptability, resources and determinism; GUI wins on discoverability and spatial tasks.
- Both can be written in C — the difference is the library you link.

---

# References

- Operating System Concepts — Silberschatz
- The Linux Programming Interface — Kerrisk (see `tty(4)`, `termios(3)`)
- `man 4 tty`, `man 7 pty`, `man 3 termios`
- X.Org documentation; Wayland Protocol documentation
- ncurses documentation — for TUI (text user interface) programming in C
