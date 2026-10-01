# Multiple Access Protocols

## What It Is

When multiple devices share a **single broadcast channel** (like a wireless LAN or Ethernet), they need a protocol to decide **who transmits when** — otherwise, packets collide and nothing gets through.

```
Shared channel:
  A ──┐
  B ──┼──── channel ──── receiver
  C ──┘

If A and B transmit at the same time → COLLISION → garbled data
```

---

## Three Families

```
Multiple Access Protocols
├── Channel Partitioning
│   ├── TDMA
│   ├── FDMA
│   └── CDMA
├── Random Access
│   ├── ALOHA (Pure / Slotted)
│   └── CSMA (CSMA/CA / CSMA/CD)
└── Taking Turns
    └── (轮询, token passing — not in syllabus)
```

---

## 1. Channel Partitioning Protocols

Divide the channel into **dedicated portions** — each device gets a fixed share.

### TDMA — Time Division Multiple Access

Divide time into **slots**. Each device gets a fixed slot.

```
Time:  |←─── frame ───→|←─── frame ───→|
Slot 0: [  A  ][  B  ][  C  ][  D  ]
        ─────────────────────────────→ time

A transmits only in slot 0
B transmits only in slot 1
...
```

| Pros | Cons |
|---|---|
| No collisions | Wastes bandwidth when a device is idle |
| Fair (equal slots) | Fixed allocation, not adaptive |
| Simple | Doesn't scale well (N devices = N slots per frame) |

### FDMA — Frequency Division Multiple Access

Divide the frequency band into **sub-bands**. Each device gets a frequency.

```
Frequency:
  ↑ [  A  ][  B  ][  C  ][  D  ]
  └─────────────────────────────→ frequency

A always uses frequency band 1
B always uses frequency band 2
...
```

| Pros | Cons |
|---|---|
| No collisions | Same waste — idle slots can't be used by others |
| Simple | Bandwidth fixed per device |

### CDMA — Code Division Multiple Access

All devices transmit **simultaneously** on the full frequency, but each is assigned a unique **code** (spreading code). The receiver uses the code to extract a specific device's signal.

```
A uses code [+1 -1 +1 -1]
B uses code [+1 +1 -1 -1]

A sends +1: signal = [+1 -1 +1 -1]
B sends +1: signal = [+1 +1 -1 -1]

Combined channel: [+2  0  0 -2]

Receiver uses A's code to extract A's signal:
  dot product of combined signal with A's code = +4 → A sent +1
```

| Pros | Cons |
|---|---|
| All transmit simultaneously | Complex code assignment |
| No time/frequency waste | Near-far problem (strong signal drowns weak) |
| Good for wireless (CDMA2000, GPS) | Harder to implement |

---

## 2. Random Access Protocols

No fixed allocation — devices **transmit whenever they want**. If collision occurs, they **retry after a random delay**.

### ALOHA — Pure ALOHA

The simplest protocol: transmit whenever you have data. If collision → wait a random time and retry.

```
A sends: ──[data]────────
B sends:     ──[data]──────
                ↑ COLLISION
A retries:                    ──[data]──
B retries:                      ──[data]──
```

**Efficiency:** maximum throughput = **18.4%** (only 18.4% of channel capacity is used successfully).

```
Vulnerability period = 2 × frame transmission time
  (any other transmission within this window causes collision)
```

### Slotted ALOHA

Improve Pure ALOHA by synchronizing transmissions to **slot boundaries** — you can only start transmitting at the beginning of a slot.

```
Slots:  |  slot 1  |  slot 2  |  slot 3  |
A:              [data]
B:                        [data]
                ↑ collision in slot 2
A retries:                        [data] (slot 3)
```

**Efficiency:** maximum throughput = **36.8%** (double Pure ALOHA).

```
Vulnerability period = 1 × frame transmission time
  (only one slot to worry about)
```

| Protocol | Vulnerability | Max Throughput |
|---|---|---|
| Pure ALOHA | 2T | 18.4% (1/2e) |
| Slotted ALOHA | 1T | 36.8% (1/e) |

---

### CSMA — Carrier Sense Multiple Access

Before transmitting, **listen to the channel** (carrier sense). If busy → don't transmit. This reduces collisions but doesn't eliminate them (propagation delay means two nodes can start transmitting at nearly the same time).

```
"Listen before you talk"
```

**Three variants:**

#### CSMA (basic)

```
If channel idle  → transmit immediately
If channel busy → wait and retry

Problem: two nodes sense "idle" at the same time → both transmit → collision
```

#### CSMA/CD — Collision Detection (Ethernet)

Same as CSMA, but **detect collision while transmitting** and abort immediately.

```
Node A starts transmitting
Node B starts transmitting (didn't hear A yet due to propagation delay)
A detects collision → stop → send jam signal → wait random backoff → retry
B detects collision → stop → wait random backoff → retry
```

```
Backoff algorithm (Binary Exponential Backoff):
  After collision n:
    wait random time from [0, 2^n - 1] slots
  Example:
    1st collision: random from [0, 1]
    2nd collision: random from [0, 3]
    3rd collision: random from [0, 7]
    ...up to 10 retries, then give up
```

| Pros | Cons |
|---|---|
| Detects collision early → saves bandwidth | Still has collisions |
| Efficient under low load | Complex implementation |
| Used in wired Ethernet | Not suitable for wireless |

#### CSMA/CA — Collision Avoidance (Wi-Fi)

Can't detect collisions in wireless (can't listen while transmitting due to signal strength). Instead, **avoid** collisions.

```
1. Listen before transmit (carrier sense)
2. If idle → wait DIFS (short interframe space) → transmit
3. If busy → wait, then use random backoff
4. Receiver sends ACK after successful reception
5. No ACK → assume collision → retry with backoff
```

```
RTS/CTS (Request to Send / Clear to Send):
  Before data, sender sends RTS → receiver replies CTS
  → clears the channel for everyone (reserves it)

  A → [RTS] → B
  A ← [CTS] ← B
  A → [DATA] → B
  A ← [ACK]  ← B
```

**Why RTS/CTS?** Solves the **hidden terminal problem** (see below).

| Pros | Cons |
|---|---|
| Avoids most collisions | ACK overhead |
| Works in wireless | RTS/CTS adds latency |
| RTS/CTS solves hidden terminal | Still has collisions under heavy load |

---

## Hidden Terminal & Exposed Terminal

### Hidden Terminal Problem

Two nodes that can't hear each other both transmit to the same receiver → collision.

```
    A ────────── B ────────── C
    ↑ can't hear C     ↑ can't hear A

A sends to B: ──[data]──→ B
C sends to B:              ──[data]──→ B
                           ↑ COLLISION (A and C can't hear each other)
```

**Solution:** RTS/CTS — A sends RTS, B replies CTS (heard by C too), C defers.

### Exposed Terminal Problem

A node is near a transmission but NOT in the path of the intended receiver — it unnecessarily defers.

```
    A ────── B ────── C
              ↑
              D

B is transmitting to A.
C wants to transmit to D.
C hears B → thinks channel is busy → defers.
But C's transmission to D wouldn't interfere with B→A!
```

**Result:** C unnecessarily waits → reduced throughput.

---

## Quick Revision

| Protocol | Mechanism | Efficiency |
|---|---|---|
| TDMA | Fixed time slots | Low if traffic bursty |
| FDMA | Fixed frequency bands | Low if traffic bursty |
| CDMA | Unique codes, simultaneous | High but complex |
| Pure ALOHA | Send anytime, retry on collision | 18.4% |
| Slotted ALOHA | Send at slot boundaries | 36.8% |
| CSMA/CD | Listen, detect collision, abort (wired) | High |
| CSMA/CA | Listen, avoid collision, ACK (wireless) | High |

- Channel partitioning = fair but wasteful
- Random access = efficient under light load
- CSMA/CD = wired Ethernet (detect while sending)
- CSMA/CA = Wi-Fi (avoid + ACK)
- Hidden terminal → RTS/CTS solves it

---

← [04-error-control](04-error-control.md) • [↑ Computer Networks](README.md) • [06-sliding-window](06-sliding-window.md) →
