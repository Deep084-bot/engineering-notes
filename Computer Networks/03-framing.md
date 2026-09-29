# Framing

## What It Is

The data link layer receives a stream of bits from the physical layer and needs to group them into **frames** — discrete, manageable units. Framing is the process of marking where each frame starts and ends.

```
Physical layer gives: 0110101101001101010010110...
                      ^^^^^^^^^^ ^^^^^^^^^^^^^^^
Framing finds:        [Frame 1]  [Frame 2]  ...
```

---

## The Four Methods

### 1. Byte Count

The first field in each frame contains the **number of bytes** in the frame.

```
Frame 1:   [5][A][B][C][D][E]
Frame 2:   [8][A][B][C][D][E][F][G]
Frame 3:   [3][X][Y][Z]
```

**Problem:** if the count field is corrupted or lost, the receiver loses synchronization — it can't find the next frame.

```
What if [5] gets corrupted to [7]?
  Receiver reads 7 bytes, but the real frame has 5.
  Everything after is misaligned → frames are all wrong.
```

| Pros | Cons |
|---|---|
| Simple | Count field corruption = total loss of sync |
| Low overhead | No way to recover if count is wrong |

---

### 2. Flag Bytes with Byte Stuffing

Each frame starts and ends with a special **flag byte** (e.g., `0x7E`). If the data contains the flag byte, it's **stuffed** (escaped) so the receiver can distinguish it from a real flag.

```
Flag  Data                         Flag
7E    41 42 7E 43 44              7E
          ↑
     data contains flag → stuff it

After byte stuffing:
  7E  41 42 7D 5E  43 44  7E
      ↑           ↑
  7D = escape byte, 5E = 7E XOR 0x20
```

**Byte stuffing rule:**
```
If data byte == flag (0x7E):  insert 0x7D, then send 0x5E (7E XOR 20)
If data byte == escape (0x7D): insert 0x7D, then send 0x5D (7D XOR 20)
```

```
Original:    A B 7E C D
Stuffed:     A B 7D 5E C D
               ↑  ↑
             escape the flag

Original:    A B 7D C D
Stuffed:     A B 7D 5D C D
               ↑  ↑
             escape the escape
```

| Pros | Cons |
|---|---|
| Can detect frame boundaries reliably | Overhead from stuffing (extra bytes) |
| Simple | Flag pattern in data is common → many stuff bytes |

**Used in:** PPP (Point-to-Point Protocol), HDLC.

---

### 3. Flag Bits with Bit Stuffing

Same idea as byte stuffing but at the **bit level** — used with bit-oriented protocols.

```
Rule: whenever five consecutive 1s appear in the data,
      insert a 0 after them.

Data:  011011111101011111010
                 ↑↑↑↑↑ five 1s → insert 0
Stuffed: 01101111101010111110101
                      ↑
                    stuffed 0

Flag: 01111110 (0x7E) — unique 8-bit pattern
```

**Receiver's job:**
```
Read bits, look for 01111110 (flag)
Between flags, after seeing 011111 → remove the next 0
```

**Why insert a 0?** To prevent the data from accidentally containing the flag pattern `01111110`. Five 1s with a stuffed 0 can never form the 8-bit flag.

| Pros | Cons |
|---|---|
| Works at bit level (transparent to byte encoding) | Slightly more complex than byte stuffing |
| Robust for any data type | One bit overhead per stuffed occurrence |

**Used in:** HDLC, LLC.

---

### 4. Physical Layer Coding Violation

Uses **invalid physical layer codes** to mark frame boundaries — codes that would never appear in normal data.

```
Manchester encoding: each bit period has a transition
  0 = transition at middle (low→high)
  1 = transition at middle (high→low)

A code violation = a period with NO transition
  → impossible in valid data → must be a delimiter

Frame start:  [code violation]
Frame data:   [normal encoded bits]
Frame end:    [code violation]
```

| Pros | Cons |
|---|---|
| No overhead (no stuffing needed) | Tied to specific physical layer encoding |
| Clean delimiters | Not all encoding schemes have violations |

---

## Comparison

| Method | Granularity | Overhead | Recovery on corruption |
|---|---|---|---|
| Byte count | Byte | Low (1 byte) | Bad — count corruption loses sync |
| Byte stuffing | Byte | Medium (stuff bytes) | Good — flag re-syncs receiver |
| Bit stuffing | Bit | Low (1 bit per 5 1s) | Good — flag pattern re-syncs |
| Coding violation | Bit | Zero | Best — invalid codes are obvious |

---

## Quick Revision

| Method | How it works | Weakness |
|---|---|---|
| Byte count | Length field in frame header | Count corruption → lost sync |
| Byte stuffing | Flag bytes + escape special data | Overhead from extra bytes |
| Bit stuffing | Insert 0 after five 1s | Complexity at bit level |
| Coding violation | Invalid physical codes as delimiters | Encoding-specific |

---

← [02-osi-model](02-osi-model.md) • [↑ Computer Networks](README.md) • [04-error-control](04-error-control.md) →
