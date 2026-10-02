# Sliding Window Protocols

## What It Is

When a sender transmits frames, it needs a way to handle **lost, corrupted, or delayed frames**. ARQ (Automatic Repeat reQuest) protocols define how the sender retransmits. **Sliding window** controls how many frames can be "in-flight" at once.

```
Without flow control:
  Sender blasts frames → receiver overflow, frames lost

With sliding window:
  Sender sends W frames → waits for ACK → sends next W frames
```

---

## Stop-and-Wait ARQ

The simplest ARQ — sender sends **one frame**, then waits for ACK before sending the next.

```
Sender                Receiver
  ──── Frame 0 ────→
                    ←──── ACK 0 ────
  ──── Frame 1 ────→
                    ←──── ACK 1 ────
  ──── Frame 2 ────→
  (if lost)
                    (no ACK → timeout)
  ──── Frame 2 ────→  (retransmit)
                    ←──── ACK 2 ────
```

```
Window size = 1 (only one frame in-flight)
Efficiency = 1 / (1 + 2a)   where a = propagation delay / transmission time
```

| Pros | Cons |
|---|---|
| Very simple | Extremely inefficient (sends one, waits) |
| No buffering needed | Channel idle most of the time |

---

## Go-Back-N ARQ

Sender can send **up to W frames** without waiting for ACK. If one frame is lost, **go back and retransmit that frame AND all subsequent frames**.

```
Sender window size W = 4

Sender:    F0  F1  F2  F3  F4  F5  F6
Receiver:  ✓   ✓   ✗   (F2 lost)
Sender:         retransmit from F2: F2  F3  F4  F5  F6
Receiver:       ✓   ✓   ✓   ✓   ✓
```

**Key rules:**
```
- Sender maintains a window of W frames
- Receiver sends cumulative ACKs: ACK(n) = "received everything up to n"
- If frame n is lost → receiver discards frames n+1, n+2, ...
- Sender times out on frame n → retransmits frames n, n+1, ..., n+W-1
```

**Cumulative ACK example:**
```
Receiver gets F0, F1, F2 → sends ACK 2
"Everything up to F2 is received, send F3 next"
```

| Pros | Cons |
|---|---|
| Better than Stop-and-Wait | Retransmits frames that were already received correctly |
| Simple receiver (no out-of-order buffer) | Wasted bandwidth on retransmission |
| Receiver doesn't buffer out-of-order | Window limit ties up sender |

---

## Selective Repeat ARQ

Sender sends **up to W frames**. If one frame is lost, **only retransmit the lost frame** — not the ones that arrived correctly.

```
Sender window size W = 4

Sender:    F0  F1  F2  F3  F4  F5  F6
Receiver:  ✓   ✓   ✗   ✓   ✓
           (F2 lost, but F3 and F4 are buffered)
Sender:         only retransmit F2: F2
Receiver:       ✓ (F2 arrives, now deliver F2 F3 F4 in order)
```

**Key rules:**
```
- Sender sends W frames without ACK
- Receiver buffers out-of-order frames
- Receiver sends individual ACKs: ACK(n) = "frame n received"
- Sender retransmits ONLY the unACKed frames
- Sender and receiver windows MUST be ≤ 2^(n-1)
  (where n = sequence number bits)
```

**Why the window limit?**
```
If window > 2^(n-1), the receiver can't distinguish new frames
  from retransmissions (sequence numbers wrap around).

With n=3 bits (sequence 0-7), window ≤ 4:
  Sender sends: 0 1 2 3
  Receiver ACKs all → advances to expect 4 5 6 7
  If sender window was > 4, it might send 4 5 6 7 0...
  Receiver can't tell if "0" is new or a retransmit
```

| Pros | Cons |
|---|---|
| Retransmits only lost frames | More complex receiver (must buffer) |
| Better throughput than Go-Back-N | Requires individual ACKs |
| More efficient under high loss | Receiver needs more memory |

---

## Comparison

| | Stop-and-Wait | Go-Back-N | Selective Repeat |
|---|---|---|---|
| Window size | 1 | W | W |
| Frames in flight | 1 | W | W |
| On loss | retransmit 1 | retransmit from lost onwards | retransmit only lost |
| Receiver buffer | 0 | 0 | W |
| ACK type | Single | Cumulative | Individual |
| Complexity | Low | Medium | High |
| Efficiency | Very low | Medium | Highest |
| Protocol | — | TCP (simplified) | TCP SACK option |

**TCP connection:** TCP uses a hybrid — cumulative ACKs (like Go-Back-N) but with SACK (Selective ACK) option that lets the receiver report which blocks were received, so the sender only retransmits what's missing.

---

## Quick Revision

| Protocol | Window | On Loss | Efficiency |
|---|---|---|---|
| Stop-and-Wait | 1 frame | Retransmit 1 | Lowest |
| Go-Back-N | W frames | Retransmit from lost frame onward | Medium |
| Selective Repeat | W frames | Retransmit only lost frame | Highest |

- Stop-and-Wait = send one, wait (wasteful)
- Go-Back-N = cumulative ACKs, retransmit everything from loss
- Selective Repeat = individual ACKs, retransmit only what's missing
- Window size ≤ 2^(n-1) for Selective Repeat (sequence space)

---

← [05-multiple-access](05-multiple-access.md) • [↑ Computer Networks](README.md) • [07-ipv4-addressing](07-ipv4-addressing.md) →
