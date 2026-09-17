# AES (Advanced Encryption Standard)

## What It Is

**AES** is a symmetric block cipher — the most widely used encryption standard in the world (used in HTTPS, WiFi, file encryption, databases). It encrypts a 128-bit (16-byte) block using a 128/192/256-bit key through multiple rounds of deterministic transformations.

```
Plaintext (16 bytes) + Key → 10 rounds of mixing → Ciphertext (16 bytes)
```

| Variant | Key Size | Rounds | Block Size |
|---|---|---|---|
| AES-128 | 128 bits (16 bytes) | 10 | 128 bits |
| AES-192 | 192 bits (24 bytes) | 12 | 128 bits |
| AES-256 | 256 bits (32 bytes) | 14 | 128 bits |

---

## The State

AES operates on a **4×4 byte matrix** called the *State* — the 16 input bytes arranged column-first.

```
Input bytes:  [12 11 11 13] [14 16 15 11] [14 02 b1 5b] [6a cb be 39]
                                    ↓ column-major fill
State:        ┌──────────────────────┐
              │ 12  14  14  6a       │
              │ 11  16  02  cb       │
              │ 11  15  b1  be       │
              │ 13  11  5b  39       │
              └──────────────────────┘
```

---

## The Four Transformations

Each round applies four steps in order:

### 1. ShiftRows — Rotate each row left

Row 0: no shift | Row 1: shift left 1 | Row 2: shift left 2 | Row 3: shift left 3

```
Before:          After:
12 14 14 6a      12 14 14 6a    (row 0: no change)
11 16 02 cb  →   16 02 cb 11    (row 1: shift left 1)
11 15 b1 be      b1 be 11 15    (row 2: shift left 2)
13 11 5b 39      39 13 11 5b    (row 3: shift left 3)
```

**Why:** spreads bytes across columns — ensures each column of the output depends on multiple columns of the input.

### 2. SubBytes — Nonlinear substitution using the S-box

Each byte is replaced using the **S-box** — a fixed 256-entry lookup table that provides **non-linearity** (the key ingredient that makes AES hard to crack).

```
byte → row = high nibble, col = low nibble → S-box lookup

Example: byte 0x12
  row = 1, col = 2
  sbox[16×1 + 2] = 0x7c

  0x12 → 0x7c
```

**S-box properties:**
- Designed to resist linear and differential cryptanalysis
- Not random — carefully constructed from inversion in GF(2⁸) + affine transform
- Each input maps to a unique output (permutation)

### 3. MixColumns — Multiply each column by a fixed matrix

Each column is treated as a polynomial and multiplied (in GF(2⁸)) by a fixed matrix.

```
Multiplication matrix (fixed, same for every round):

│ 2  3  1  1 │
│ 1  2  3  1 │
│ 1  1  2  3 │
│ 3  1  1  2 │

For each column [a0 a1 a2 a3] → output:
  x0 = 2·a0 ⊕ 3·a1 ⊕ 1·a2 ⊕ 1·a3
  x1 = 1·a0 ⊕ 2·a1 ⊕ 3·a2 ⊕ 1·a3
  x2 = 1·a0 ⊕ 1·a1 ⊕ 2·a2 ⊕ 3·a3
  x3 = 3·a0 ⊕ 1·a1 ⊕ 1·a2 ⊕ 2·a3
```

**The Galois Field multiply (`xf` function):**

```
"×2" in GF(2⁸):  shift left by 1, XOR with 0x1B if MSB was 1
  (0x1B = x⁸ + x⁴ + x³ + x + 1 — the AES irreducible polynomial)

"×3" = "×2" ⊕ original   (since 3 = 2 + 1 in GF(2))
  3·a = xf(a) ^ a
```

```c
// Multiply by 2 in GF(2⁸)
unsigned char xf(unsigned char s) {
    unsigned char temp = s << 1;
    if ((s >> 7) == 1) temp ^= 0x1B;  // if overflow, reduce mod poly
    return temp;
}

// Mix one column using the fixed matrix
void mixCol(unsigned char x[4]) {
    unsigned char a0 = x[0], a1 = x[1], a2 = x[2], a3 = x[3];
    x[0] = xf(a0) ^ (xf(a1) ^ a1) ^ a2 ^ a3;           // 2·a0 ⊕ 3·a1 ⊕ 1·a2 ⊕ 1·a3
    x[1] = a0 ^ xf(a1) ^ (xf(a2) ^ a2) ^ a3;           // 1·a0 ⊕ 2·a1 ⊕ 3·a2 ⊕ 1·a3
    x[2] = a0 ^ a1 ^ xf(a2) ^ (xf(a3) ^ a3);           // 1·a0 ⊕ 1·a1 ⊕ 2·a2 ⊕ 3·a3
    x[3] = (xf(a0) ^ a0) ^ a1 ^ a2 ^ xf(a3);           // 3·a0 ⊕ 1·a1 ⊕ 1·a2 ⊕ 2·a3
}
```

**Why:** ensures every output byte depends on all four input bytes in the column — full diffusion.

### 4. AddRoundKey — XOR state with the round key

```
State ⊕ RoundKey

  state[i][j] ^= roundKey[i * 4 + j]
```

Simple XOR — this is where the key enters the algorithm.

---

## Key Expansion (Key Schedule)

AES-128 needs **11 round keys** (initial + 10 rounds), each 16 bytes. The original 16-byte key is expanded into all round keys using a deterministic process.

```
Original key → roundKeys[0]
Round keys 1-10 generated iteratively from roundKeys[0]
```

### The Expansion Process

```
For each round r (1 to 10):

  1. Take last 4 bytes of previous round key → temp[4]
  2. Rotate left by 1 byte:     [b0 b1 b2 b3] → [b1 b2 b3 b0]
  3. Substitute each byte:      S-box lookup on all 4 bytes
  4. XOR with round constant:   temp[0] ^= rcon[r-1]
  5. XOR into first 4 bytes:    new[0:3] = prev[0:3] ⊕ temp
  6. Generate remaining bytes:  new[i] = prev[i] ⊕ new[i-4]
```

**Round constants (Rcon):**

```
rcon[10] = {0x01, 0x02, 0x04, 0x08, 0x10, 0x20, 0x40, 0x80, 0x1B, 0x36}

Each is a power of 2 in GF(2⁸):
  rcon[1] = 0x01,  rcon[2] = 0x02,  rcon[3] = 0x04, ...
  (used only in the first byte of the rotation step)
```

---

## The Full AES-128 Structure

```
  ┌────────────────────────────┐
  │ AddRoundKey (key[0])       │  ← initial key mixing
  ├────────────────────────────┤
  │ Round 1-9 (×9 times):      │
  │   ├─ ShiftRows             │
  │   ├─ SubBytes              │
  │   ├─ MixColumns            │
  │   └─ AddRoundKey (key[r])  │
  ├────────────────────────────┤
  │ Final round (10):          │
  │   ├─ ShiftRows             │
  │   ├─ SubBytes              │
  │   └─ AddRoundKey (key[10]) │  ← NO MixColumns in final round
  └────────────────────────────┘
  → Ciphertext
```

**Why no MixColumns in the final round?** It's an invertible linear operation — omitting it simplifies decryption without reducing security (SubBytes + ShiftRows already provide sufficient mixing).

---

## The Code

### Helper Functions

```c
// S-box lookup
unsigned char subbyte(unsigned char x) {
    unsigned char row = x >> 4;        // high nibble → row
    unsigned char col = x & 0x0F;      // low nibble → column
    return sbox[16 * row + col];
}

// GF(2⁸) ×2
unsigned char xf(unsigned char s) {
    unsigned char temp = s << 1;
    if ((s >> 7) == 1) temp ^= 0x1B;  // reduce mod AES polynomial
    return temp;
}

// MixColumns on a single column
void mixCol(unsigned char x[4]) {
    unsigned char a0 = x[0], a1 = x[1], a2 = x[2], a3 = x[3];
    x[0] = xf(a0) ^ (xf(a1) ^ a1) ^ a2 ^ a3;
    x[1] = a0 ^ xf(a1) ^ (xf(a2) ^ a2) ^ a3;
    x[2] = a0 ^ a1 ^ xf(a2) ^ (xf(a3) ^ a3);
    x[3] = (xf(a0) ^ a0) ^ a1 ^ a2 ^ xf(a3);
}
```

### Key Expansion

```c
unsigned char roundKeys[11][16];
roundKeys[0] = key;  // original key

for (round = 1; round <= 10; round++) {
    // Last 4 bytes of previous round key
    temp = roundKeys[round-1][12..15]

    // Rotate, SubBytes, XOR round constant
    temp = [temp[1], temp[2], temp[3], temp[0]]
    temp = subbyte(temp[0]), subbyte(temp[1]), ...
    temp[0] ^= rcon[round-1]

    // First 4 bytes: previous XOR temp
    roundKeys[round][0..3] = roundKeys[round-1][0..3] ⊕ temp

    // Remaining 12 bytes: each = previous round's same pos ⊕ current round's prev 4 bytes
    roundKeys[round][4..15] = roundKeys[round-1][4..15] ⊕ roundKeys[round][4..12]
}
```

### Main Loop (9 rounds + final)

```c
// Initial AddRoundKey
state ⊕ roundKeys[0]

// Rounds 1-9
for round = 1 to 9:
    ShiftRows(state)
    SubBytes(state)
    MixColumns(state)           // on each column
    state ⊕ roundKeys[round]

// Final round (no MixColumns)
ShiftRows(state)
SubBytes(state)
state ⊕ roundKeys[10]
→ Ciphertext
```

---

## Walkthrough: One Round

```
Key:   11 12 13 14  15 16 11 13  12 14 12 14  15 11 11 12
Plain: 12 11 11 13  14 16 15 11  14 02 b1 5b  6a cb be 39

Step 1 — AddRoundKey (key[0]):
  State ⊕ Key = [12^11  14^15  14^12  6a^15]
                [11^16  16^11  02^14  cb^11]
                [11^12  15^14  b1^12  be^14]
                [13^15  11^11  5b^11  39^12]

Step 2 — ShiftRows
  Row 0: unchanged
  Row 1: shift left 1
  Row 2: shift left 2
  Row 3: shift left 3

Step 3 — SubBytes (S-box on every byte)

Step 4 — MixColumns (matrix multiply each column in GF(2⁸))

Step 5 — AddRoundKey (key[round])
```

---

## S-box: The Heart of AES

```
The 256-entry S-box is a carefully designed permutation:

  Input byte 0x00 → output 0x63
  Input byte 0x01 → output 0x7c
  ...
  Input byte 0xFF → output 0x16

Design goals:
  ✗ No fixed points: S(x) ≠ x for any x
  ✗ No inverse fixed points: S(x) ≠ x̄ for any x
  ✗ High non-linearity (resists linear cryptanalysis)
  ✗ High差分 uniformity (resists differential cryptanalysis)

Construction:
  1. Compute multiplicative inverse in GF(2⁸)
  2. Apply an affine transformation over GF(2)
```

---

## Galois Field GF(2⁸) — The Math Behind MixColumns

```
AES works in GF(2⁸) — the finite field of 256 elements

Polynomial basis: x⁷ + x⁴ + x³ + x + 1  (irreducible, = 0x11B)

Multiplication by 2:
  shift left 1, XOR with 0x1B if the MSB was set
  (this is polynomial reduction mod the irreducible polynomial)

Multiplication by 3:
  3 = 2 ⊕ 1, so 3·a = xf(a) ⊕ a

The fixed matrix [2,3,1,1; 1,2,3,1; ...] is chosen so that:
  - it's invertible (for decryption)
  - it provides maximum diffusion (each output depends on all inputs)
  - it's efficient (only ×2 and ×3 needed, no arbitrary multiply)
```

---

## Security

| Property | AES |
|---|---|
| Key size (128) | 2¹²⁸ possible keys → brute-force infeasible |
| Known attacks | No practical attack better than brute-force |
| Side-channel | Vulnerable to timing/cache attacks if not constant-time |
| Mode of operation | Block cipher needs a mode (ECB, CBC, CTR, GCM) |

**Why AES is secure:**
- 4 transformations create full diffusion (each output byte depends on every input byte after ~2 rounds)
- Non-linearity from S-box resists algebraic attacks
- Key schedule ensures each round key is independent
- Designed by Joan Daemen and Vincent Rijmen — won the AES competition in 2001

---

## Programming Analogy

```
AES = a carefully designed hash-like transformation

State (4×4) = a mutable matrix that gets transformed each round

SubBytes    = a non-linear lookup table
  (like a random oracle / hash — hard to reverse algebraically)

ShiftRows   = positional permutation (like byte-level shuffling)
  (breaks column alignment — each column now depends on multiple)

MixColumns  = matrix multiplication in GF(2⁸)
  (full diffusion within each column — like a mixing function)
  xf() = ×2 in a Galois field = shift + conditional XOR
  (like polynomial arithmetic mod an irreducible)

AddRoundKey = XOR with key material
  (the only step where the secret enters — like a keyed hash)

Round keys  = derived from the main key via expansion
  (like a PRG generating many keys from one seed)

Full AES = N rounds of the 4 steps
  (like iterating a compression function — each round
   increases the mixing until it's irreversible in practice)

Why no MixColumns in last round?
  It's linear and invertible — removing it simplifies
  decryption without weakening security.
  (like omitting a reversible step in a pipeline)
```

---

## Code Gotchas

| Issue | The Code's Approach |
|---|---|
| `sbox` lookup | `row = x >> 4`, `col = x & 0x0F` → index = `16*row + col` |
| GF(2⁸) overflow | `if (s >> 7) temp ^= 0x1B` — polynomial reduction |
| MixColumns matrix | Hardcoded multiply: `xf(a) ^ a` = ×3, `xf(a)` = ×2 |
| Round constants | Powers of 2 in GF(2⁸): `{0x01, 0x02, 0x04, ...}` |
| Key schedule | Rotate → SubBytes → XOR Rcon → chain XOR for remaining bytes |
| Final round | Skip MixColumns — only ShiftRows, SubBytes, AddRoundKey |

---

## Revision Summary

| Component | Purpose |
|---|---|
| State | 4×4 byte matrix (the data) |
| S-box | Non-linear substitution (the key to security) |
| ShiftRows | Row rotation (inter-column mixing) |
| MixColumns | Matrix × column in GF(2⁸) (diffusion) |
| AddRoundKey | XOR with round key (secret injection) |
| Key Expansion | Derives 11 round keys from original key |
| Rcon | Round constants (powers of 2 in GF(2⁸)) |
| xf() | GF(2⁸) ×2 via shift + conditional XOR |
| 0x1B | AES irreducible polynomial (x⁸+x⁴+x³+x+1) |
| Final round | No MixColumns (simplifies decryption) |

- AES-128 = 10 rounds, AES-256 = 14 rounds
- Each round: ShiftRows → SubBytes → MixColumns → AddRoundKey
- Key schedule generates round keys deterministically
- 2¹²⁸ keys → brute-force impossible
- SubBytes provides non-linearity; MixColumns provides diffusion
