# SHA-1 (Secure Hash Algorithm 1)

## What It Is

**SHA-1** is a cryptographic hash function that takes any message and produces a fixed **160-bit (20-byte) digest** — a unique fingerprint of the input.

```
任意长度输入 → SHA-1 → 固定 160-bit 输出
"hello"     → SHA-1 → aaf4c61ddcc5e8a2dabede0f3b482cd9aea9434d
"hello!"    → SHA-1 → af53e5b27c3f5d3c0c3f2b1a8e4f6d9c2a1b3e5f
```

| Property | Value |
|---|---|
| Output size | 160 bits (20 bytes) |
| Block size | 512 bits (64 bytes) |
| Input size | Any length |
| Rounds | 80 |
| Internal state | 5 × 32-bit words = 160 bits |

---

## Why Hash Functions Matter

| Use | How |
|---|---|
| **Integrity check** | Compare hashes to detect tampering |
| **Password storage** | Store hash, not plaintext |
| **Digital signatures** | Sign the hash, not the full message |
| **Data deduplication** | Hash as a unique key |

---

## The Algorithm

### Step 1: Initialize Hash Values

Five 32-bit constants — the starting state, derived from square roots of the first five primes.

```
h0 = 0x67452301
h1 = 0xEFCDAB89
h2 = 0x98BADCFE
h3 = 0x10325476
h4 = 0xC3D2E1F0
```

### Step 2: Pad the Message

The message must be a multiple of 512 bits (64 bytes).

```
Original message:  "abc" (3 bytes)

Pad:
  1. Append 0x80 (one bit)
  2. Append zeros until length ≡ 56 mod 64
  3. Append original length in bits as 64-bit big-endian

"abc" padded:
  61 62 63 80 00 00 00 ... 00 00 00 00 00 00 00 18
  ↑"abc" ↑0x80    ↑zeros (51 bytes)         ↑24 bits (3×8)
```

```
Padding rule:
  len(msg) + 1 + padding + 8 = multiple of 64 bytes
  (8 bytes = 64-bit length field at the end)
```

### Step 3: Process Each 512-bit Block

For each 64-byte block:

#### 3a. Expand into 80 words

```
Original block:  W[0] ... W[15]   (16 × 32-bit words)
Extended:        W[0] ... W[79]   (80 × 32-bit words)

For j = 16 to 79:
  W[j] = rotate_left(W[j-3] ⊕ W[j-8] ⊕ W[j-14] ⊕ W[j-16], 1)
```

**Why expand?** The 80 rounds need 80 different "message words" — expanding from 16 to 80 creates non-linear mixing across the block.

#### 3b. Initialize working variables

```
a = h0,  b = h1,  c = h2,  d = h3,  e = h4
```

#### 3c. 80 rounds of compression

```
For each round j (0 to 79):

  1. Pick round function F and constant K based on j:
     ┌──────────────────────────────────────────────────────┐
     │ Round 0-19:   F = (b & c) | (~b & d)    K = 0x5A827999  │
     │ Round 20-39:  F = b ⊕ c ⊕ d             K = 0x6ED9EBA1  │
     │ Round 40-59:  F = (b & c) | (b & d) | (c & d)  K = 0x8F1BBCDC │
     │ Round 60-79:  F = b ⊕ c ⊕ d             K = 0xCA62C1D6  │
     └──────────────────────────────────────────────────────┘

  2. temp = rotate_left(a, 5) + F + e + K + W[j]

  3. Shift registers:
     e = d
     d = c
     c = rotate_left(b, 30)
     b = a
     a = temp
```

#### 3d. Add to hash values

```
h0 += a,  h1 += b,  h2 += c,  h3 += d,  h4 += e
```

### Step 4: Output

Concatenate h0 h1 h2 h3 h4 → 160-bit (20-byte) digest.

```
digest[0..3]   = h0 (big-endian)
digest[4..7]   = h1
digest[8..11]  = h2
digest[12..15] = h3
digest[16..19] = h4
```

---

## The Four Round Functions

```
Round 0-19:   choose(b,c,d)  = (b AND c) OR (NOT b AND d)
              → selects bits from c when b=1, d when b=0
              → "if b is set, use c; else use d"

Round 20-39:  parity(b,c,d)  = b XOR c XOR d
              → parity of three bits

Round 40-59:  majority(b,c,d) = (b AND c) OR (b AND d) OR (c AND d)
              → the bit that appears in at least 2 of 3

Round 60-79:  parity(b,c,d) = b XOR c XOR d
              → same as round 20-39
```

**Round constants (K):**

```
K[0-19]   = 0x5A827999  (floor(2³⁰ × √2))
K[20-39]  = 0x6ED9EBA1  (floor(2³⁰ × √3))
K[40-59]  = 0x8F1BBCDC  (floor(2³⁰ × √5))
K[60-79]  = 0xCA62C1D6  (floor(2³⁰ × √10))
```

---

## Key Helper Functions

```c
// Rotate bits left (circular shift)
uint32 rotate_left(uint32 val, int bits){
    return (val << bits) | (val >> (32 - bits));
}

// Convert 4 big-endian bytes → uint32
uint32 bytes_to_uint32(const unsigned char *p){
    return ((uint32)p[0] << 24) | ((uint32)p[1] << 16) |
           ((uint32)p[2] << 8)  | ((uint32)p[3]);
}

// Convert uint32 → 4 big-endian bytes
void uint32_to_bytes(uint32 val, unsigned char *p){
    p[0] = (val >> 24) & 0xFF;
    p[1] = (val >> 16) & 0xFF;
    p[2] = (val >> 8)  & 0xFF;
    p[3] = val & 0xFF;
}
```

**Why rotate_left and not shift?** A left shift loses the top bits; a rotation moves them to the bottom. This preserves entropy — no bits are thrown away.

---

## The Code

```c
#include <stdio.h>
#include <string.h>
#include <stdlib.h>

typedef unsigned int uint32;

uint32 rotate_left(uint32 val, int bits){
    return (val << bits) | (val >> (32 - bits));
}

uint32 choose(uint32 x, uint32 y, uint32 z){
    return (x & y) | (~x & z);
}

uint32 parity(uint32 x, uint32 y, uint32 z){
    return x ^ y ^ z;
}

uint32 majority(uint32 x, uint32 y, uint32 z){
    return (x & y) | (x & z) | (y & z);
}

uint32 bytes_to_uint32(const unsigned char *p){
    return ((uint32)p[0] << 24) | ((uint32)p[1] << 16) |
           ((uint32)p[2] << 8)  | ((uint32)p[3]);
}

void uint32_to_bytes(uint32 val, unsigned char *p){
    p[0] = (val >> 24) & 0xFF;
    p[1] = (val >> 16) & 0xFF;
    p[2] = (val >> 8)  & 0xFF;
    p[3] = val & 0xFF;
}

void sha1(const char *message, unsigned char *digest){
    uint32 h0 = 0x67452301, h1 = 0xEFCDAB89, h2 = 0x98BADCFE,
           h3 = 0x10325476, h4 = 0xC3D2E1F0;
    uint32 a, b, c, d, e, f, k, temp;
    uint32 w[80];
    int i, j;
    unsigned long long msg_len = strlen(message);
    int padded_len = ((int)msg_len + 9);
    while (padded_len % 64 != 0) padded_len++;

    unsigned char *padded = (unsigned char *)calloc(padded_len, 1);
    memcpy(padded, message, msg_len);
    padded[msg_len] = 0x80;
    {
        unsigned long long bit_len = msg_len * 8;
        for (int x = 0; x < 8; x++)
            padded[padded_len - 8 + x] = (bit_len >> (56 - 8 * x)) & 0xFF;
    }

    for (i = 0; i < padded_len; i += 64){
        for (j = 0; j < 16; j++)
            w[j] = bytes_to_uint32(padded + i + j * 4);
        for (j = 16; j < 80; j++)
            w[j] = rotate_left(w[j-3] ^ w[j-8] ^ w[j-14] ^ w[j-16], 1);

        a = h0; b = h1; c = h2; d = h3; e = h4;

        for (j = 0; j < 80; j++){
            if (j < 20)      { f = choose(b, c, d);   k = 0x5A827999; }
            else if (j < 40) { f = parity(b, c, d);   k = 0x6ED9EBA1; }
            else if (j < 60) { f = majority(b, c, d);  k = 0x8F1BBCDC; }
            else             { f = parity(b, c, d);   k = 0xCA62C1D6; }

            temp = rotate_left(a, 5) + f + e + k + w[j];
            e = d; d = c; c = rotate_left(b, 30); b = a; a = temp;
        }
        h0 += a; h1 += b; h2 += c; h3 += d; h4 += e;
    }
    free(padded);
    uint32_to_bytes(h0, digest);
    uint32_to_bytes(h1, digest + 4);
    uint32_to_bytes(h2, digest + 8);
    uint32_to_bytes(h3, digest + 12);
    uint32_to_bytes(h4, digest + 16);
}

int main(){
    char message[1024];
    unsigned char digest[20];
    printf("Enter message: ");
    fgets(message, sizeof(message), stdin);
    message[strcspn(message, "\n")] = '\0';
    sha1(message, digest);
    printf("SHA-1 Hash: ");
    for (int i = 0; i < 20; i++) printf("%02x", digest[i]);
    printf("\n");
    return 0;
}
```

---

## Walkthrough: "abc"

```
Input: "abc" = 0x61 0x62 0x63

Padding:
  61 62 63 80 00...00 00 00 00 00 00 00 18
                              ↑ length = 24 bits (3×8)

Expanded words (first few):
  W[0]  = 0x61626380
  W[1]  = 0x00000000
  ...
  W[15] = 0x00000018
  W[16] = rotate_left(W[13] ⊕ W[8] ⊕ W[2] ⊕ W[0], 1) = ...
  ...continues to W[79]

After 80 rounds:
  h0 = 0xA9993E36
  h1 = 0x4706816A
  h2 = 0xBA3E2571
  h3 = 0x7850C26C
  h4 = 0x9CD0D89D

Output: a9993e364706816aba3e25717850c26c9cd0d89d
```

---

## Security — Why SHA-1 Is Broken

| Year | Attack | Significance |
|---|---|---|
| 2005 | Theoretical collision attack | First cracks |
| 2017 | SHAttered (Google/CWI) | First actual collision produced |
| 2020 | chosen-prefix collision | Practical attacks feasible |

```
SHA-1 produces a 160-bit hash.
  Theoretical birthday attack: 2⁸⁰ operations (infeasible)
  Actual attacks: far cheaper due to weaknesses in the compression function

SHAttered (2017):
  Two different PDF files with the same SHA-1 hash
  → hash is broken for collision resistance
```

**What to use instead:**

| Hash | Output | Status |
|---|---|---|
| SHA-1 | 160 bits | Broken — don't use |
| SHA-256 | 256 bits | Secure, recommended |
| SHA-3 | 224/256/384/512 | Secure, newer alternative |
| BLAKE2/BLAKE3 | Variable | Fast and secure |

---

## Programming Analogy

```
SHA-1 = a one-way compression function

One-way:  easy to compute hash from message,
          impossible to recover message from hash
          (like a trapdoor with no key)

Compression: arbitrary-length input → fixed 160-bit output
  (like a many-to-one function — inevitable collisions exist)

Padding = standardizing input to a fixed block size
  (like framing/padding in network protocols)

80 rounds of mixing = iterative hash compression:
  each round applies a nonlinear function (F) + a constant (K)
  + a message word (W[j]) + a rotate + modular addition
  → the result is a irreversible-looking scramble

Merklam–Damgård construction:
  SHA-1 processes one 64-byte block at a time,
  feeding the output as input to the next block
  (like a chained reduction)

Birthday attack = finding two inputs with same hash
  takes ~2^(n/2) operations for an n-bit hash
  (SHA-1: 2⁸⁰ — now practical for attackers)

Why collisions matter: if attacker can produce two messages
  with the same hash, they can forge signatures.
  (sign the innocent message → the malicious one also validates)
```

---

## Key Observations from the Code

| Design Choice | Why |
|---|---|
| Big-endian processing | Standard byte order for network protocols |
| `rotate_left` not shift | Preserves all bits (no entropy loss) |
| `calloc` for padding | Ensures zero-filled padding bytes |
| Length appended last 8 bytes | 64-bit length field in big-endian |
| 5 working variables (a-e) | 160-bit internal state, updated each round |
| Mod addition (not XOR) | Adds non-linearity — can't be undone by XOR |

---

## Revision Summary

| Concept | Value |
|---|---|
| Output | 160 bits (20 bytes) |
| Block size | 512 bits (64 bytes) |
| Rounds | 80 (20 per phase × 4 phases) |
| Initial hash | h0-h4 from square roots of primes |
| Padding | 0x80 + zeros + 64-bit length |
| Word expansion | 16 → 80 via rotate_left(⊕ of 4 words, 1) |
| Round functions | choose, parity, majority, parity |
| Round constants | K₀₋₁₉=√2, K₂₀₋₃₉=√3, K₄₀₋₅₉=√5, K₆₀₋₇₉=√10 |
| State update | temp = rot(a,5) + F + e + K + w[j], then shift |
| Final output | h0 ‖ h1 ‖ h2 ‖ h3 ‖ h4 (big-endian bytes) |
| Status | Broken (2017 collision), use SHA-256 instead |

- SHA-1 = 80 rounds of nonlinear mixing over 5 registers
- Padding ensures multiple-of-512-bit input
- Word expansion creates 80 message-dependent values from 16
- 4 different round functions prevent linear attacks
- Birthday attack = 2⁸⁰ → now practical → SHA-1 deprecated
