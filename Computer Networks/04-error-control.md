# Error Control Codes

## What It Is

Bits get flipped during transmission (noise, interference). **Error control codes** let the receiver **detect** (and sometimes **correct**) these errors.

```
Sender:  sends data + redundancy (check bits)
Channel: flips some bits
Receiver: checks redundancy → detects errors → requests retransmission or corrects
```

---

## 1. Parity Bit Checker

Add one bit to make the total number of 1s **even** (even parity) or **odd** (odd parity).

```
Data: 1011001 (four 1s → even count)
Even parity bit: 0  → 1011001 0 (still four 1s)
Sent: 10110010

Receiver counts 1s:
  Even → accepted
  Odd  → error detected
```

```
Data: 1011011 (five 1s → odd count)
Even parity bit: 1  → 1011011 1 (six 1s → even)
Sent: 10110111
```

| Pros | Cons |
|---|---|
| Trivial to compute | CANNOT detect even number of bit flips |
| Zero overhead (1 bit) | Cannot correct errors |

**Key weakness:** if 2 bits flip (or any even number), parity stays valid → undetected error.

---

## 2. 2D Parity Checker

Arrange data in a **matrix** and compute parity for each row AND each column. The receiver checks all row and column parities.

```
Data bits (4×3 matrix):
  1 0 1  | row parity = 0 (even)
  0 1 0  | row parity = 1 (odd)
  1 1 0  | row parity = 0 (even)
  0 0 1  | row parity = 1 (odd)
  ───────
  col     col     col
  parity  parity  parity
  0       0       0

Full matrix with parities:
  1 0 1  0
  0 1 0  1
  1 1 0  0
  0 0 1  1
  0 0 0  0  ← row of column parities
```

**Detection:** receiver recomputes all row/column parities. If any don't match → error.

**Correction:** the intersection of the bad row and bad column pinpoints the flipped bit.

```
If row 2 has wrong parity AND column 1 has wrong parity:
  → bit at (row 2, col 1) is the flipped one → flip it back
```

| Pros | Cons |
|---|---|
| Can detect AND correct single-bit errors | More overhead (row + column parity bits) |
| Can detect 2-bit errors | Cannot correct multi-bit errors |

---

## 3. Checksum

Compute a **sum of all data units** (typically 16-bit words) and append the result.

```
Data (16-bit words):  0x4500  0x003C  0x1C46  0x0000

Checksum computation:
  Step 1: Add all words (with carry wrap-around)
    0x4500 + 0x003C = 0x453C
    0x453C + 0x1C46 = 0x6182
    0x6182 + 0x0000 = 0x6182

  Step 2: Take one's complement (flip all bits)
    ~0x6182 = 0x9E7D

  Checksum = 0x9E7D → appended to the message
```

**Receiver's check:**
```
Add all words INCLUDING the checksum:
  0x4500 + 0x003C + 0x1C46 + 0x0000 + 0x9E7D = 0xFFFF

If result == 0xFFFF → no errors
If result != 0xFFFF → error detected
```

| Pros | Cons |
|---|---|
| Simple to implement | Detects errors, doesn't correct |
| Used in IP, TCP, UDP | Can miss some error patterns |
| 16-bit overhead | Re-computation on every hop (IPv4) |

---

## 4. CRC (Cyclic Redundancy Check)

The most widely used error detection code. Treats data as a **polynomial** and divides by a fixed **generator polynomial**. The remainder is appended as the CRC.

```
Data:     1101011011  (degree 9 polynomial)
Generator: 1000100101  (degree 5 polynomial, G(x) = x⁵+x²+1)

Step 1: Append 5 zeros (degree of G) to data
  1101011011 00000

Step 2: Divide by generator using XOR (mod-2 division)
  (perform polynomial long division in GF(2))

Step 3: The remainder (5 bits) is the CRC
  Remainder = 11010

Step 4: Replace the appended zeros with the remainder
  Transmitted: 1101011011 11010
```

**Receiver's check:**
```
Divide received data by the same generator:
  1101011011 11010  ÷  1000100101

  Remainder = 00000 → no errors
  Remainder ≠ 00000 → error detected
```

**Common CRC standards:**

| Standard | Polynomial | Used In |
|---|---|---|
| CRC-8 | x⁸+x²+x+1 | Short data |
| CRC-16 | x¹⁶+x¹⁵+x²+1 | USB, Bluetooth |
| CRC-32 | x³²+... | Ethernet, ZIP |

| Pros | Cons |
|---|---|
| Detects all burst errors ≤ degree of G | Cannot correct errors |
| Very efficient in hardware | Requires polynomial division |
| High detection probability | Fixed generator polynomial |

---

## Comparison

| Method | Detect | Correct | Overhead | Use |
|---|---|---|---|---|
| Parity | Single-bit errors | No | 1 bit | Simple link |
| 2D Parity | Single + double-bit | Single-bit | (rows+cols) bits | RAM |
| Checksum | Most errors | No | 16 bits | IP, TCP, UDP |
| CRC | Burst errors ≤ degree | No | 8/16/32 bits | Ethernet, Wi-Fi |

---

## Quick Revision

| Code | Mechanism | Key Property |
|---|---|---|
| Parity | Count 1s | Simple, 1-bit overhead, even flips miss |
| 2D Parity | Row + column checks | Can correct single-bit errors |
| Checksum | Sum words, complement | Used in internet protocols |
| CRC | Polynomial division | Gold standard for detection |

- Parity = basic, single-bit
- 2D parity = adds correction capability
- Checksum = internet's choice (IP/TCP/UDP)
- CRC = best detection (Ethernet/Wi-Fi)

---

← [03-framing](03-framing.md) • [↑ Computer Networks](README.md) • [05-multiple-access](05-multiple-access.md) →
