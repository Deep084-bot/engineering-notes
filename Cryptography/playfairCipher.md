# Playfair Cipher

## What It Is

The **Playfair cipher** is a digraph (2-letter) substitution cipher invented by Charles Wheatstone in 1854. Unlike Caesar/Affine which encrypt one letter at a time, Playfair encrypts **pairs of letters** — making frequency analysis much harder.

```
Plaintext:  BALLOON
Digraphs:   BA LX LO ON
Ciphertext: IK VG KD YK
```

---

## How It Works

### 1. Build the 5×5 Key Grid

1. Remove duplicates from the key
2. Replace J → I (25 letters, 26 need to fit in 5×5)
3. Fill remaining cells with unused letters in alphabetical order

```
Key: MONARCHY
Grid:    M  O  N  A  R
         C  H  Y  B  D
         E  F  G  I/J K
         L  P  Q  S  T
         U  V  W  X  Z

All 25 letters present, J merged into I
```

### 2. Prepare the Plaintext

```
1. Convert to uppercase, J → I
2. Split into digraphs (pairs)
3. If a pair has duplicate letters → insert X between them
4. If total length is odd → append X to make it even

BALLOON → BA LX LO ON  (L and L split with X)
```

### 3. Encrypt Each Pair

Find both letters in the grid, then apply one of three rules:

```
┌──────────────────────────────────────────────────────────────┐
│ Rule 1: Same ROW → move RIGHT  (wrap to start)              │
│         B O → B→D, O→N → BN                                 │
│                                                              │
│ Rule 2: Same COLUMN → move DOWN  (wrap to top)              │
│         M U → M→U, U→V → UV                                 │
│                                                              │
│ Rule 3: Different row AND column → swap corners of rectangle │
│         B A → B is at (1,3), A is at (0,3)                  │
│         Rectangle corners: swap columns → (1,3) and (0,3)   │
│         Wait — different rows, same col? No:                 │
│         B at (1,3), Y at (0,2) → rectangle with corners     │
│         B→(1,2)=G, Y→(0,3)=A → GA                            │
└──────────────────────────────────────────────────────────────┘
```

**Visual: the three rules**

```
SAME ROW:            SAME COLUMN:         RECTANGLE:
→ move right         ↓ move down          swap corners

a b c d e           a d g j m            a c e
f g h i k           b e h k n            b d f
l m n o p           c f i l o            g h i
                    d g j m p

'bc' → 'cd'         'ag' → 'di'          'af' → 'cb' (swap col)
```

---

## Complete Example

```
Key: MONARCHY
Grid:    M  O  N  A  R
         C  H  Y  B  D
         E  F  G  I  K
         L  P  Q  S  T
         U  V  W  X  Z

Plaintext: BALLOON

Step 1 — Prepare digraphs:
  B A L L O O N
  BA LX LO ON X   (insert X between duplicate L-L and O-O,
                     pad final X because ON has nothing left)

Step 2 — Encrypt:
  B(1,3) A(0,3) → same column? B col=3, A col=3 → YES
    → rule 2: move down
    → B→(2,3)=I, A→(3,3)=S → IS

  L(3,0) X(4,3) → rectangle
    → swap corners: L→(3,3)=S, X→(4,0)=U → SU

  L(3,0) O(0,1) → rectangle
    → L→(3,1)=P, O→(0,0)=M → PM

  O(0,1) N(0,2) → same row
    → rule 1: move right
    → O→(0,2)=N, N→(0,3)=A → NA

Ciphertext: IS SU PM NA → "ISSUPMNA"
```

---

## Code Walkthrough

```c
// 1. Build grid from key (deduplicated, J→I)
// 2. Fill remaining letters A-Z skipping duplicates
// 3. Prepare plaintext: J→I, insert X between duplicates, pad odd length
// 4. For each digraph pair, find positions in grid
// 5. Apply the three rules:
//    Same row     → shift both right with wrap
//    Same column  → shift both down with wrap
//    Rectangle    → swap the column indices
```

**The key encoding lines:**

```c
// Same row → move right
if (r1 == r2) {
    ans[k++] = grid[r1][(c1 + 1) % 5];
    ans[k++] = grid[r2][(c2 + 1) % 5];
}
// Same column → move down
else if (c1 == c2) {
    ans[k++] = grid[(r1 + 1) % 5][c1];
    ans[k++] = grid[(r2 + 1) % 5][c2];
}
// Rectangle → swap corners (swap column indices)
else {
    ans[k++] = grid[r1][c2];
    ans[k++] = grid[r2][c1];
}
```

---

## The Insert-X Rule

When two adjacent letters are the same, insert X to split them.

```
BALLOON
  B A L L O O N
  ↓   ↓ ↑ ↓   ↓
  BA  L-X  O-X  ON

Why? Without X, LL would encrypt as a digraph —
  same-row rule on same letters = same letter output,
  which leaks information.
```

**Alternative:** some implementations use Q instead of X, or only insert X if the pair would otherwise be identical.

---

## Decryption (reverse of encryption)

```
Same row     → move LEFT (wrap to end)
Same column  → move UP (wrap to bottom)
Rectangle    → swap corners (same as encryption — it's symmetric!)
```

---

## Key Properties

| Property | Value |
|---|---|
| Type | Digraph substitution |
| Key | 25! possible grids (but realistically less) |
| Alphabet | 25 letters (J merged into I) |
| Block size | 2 letters |
| Breakable? | Yes — frequency analysis of digraphs |

### Why digraphs are harder to break than single letters

```
Single-letter frequency:
  E T A O I N S H R  ← very distinct peaks
  Easy to see which letter maps to which

Digraph frequency (400 possible pairs):
  TH HE IN ER AN RE  ← many pairs with similar frequency
  Much flatter distribution → harder to crack
```

---

## Security

| Strength | Weakness |
|---|---|
| 25! possible grids (~1.5 × 10²⁵) | Digraph frequency analysis possible |
| No single-letter frequency pattern | Only 25 letters (J=I loses information) |
| Same-letter pairs handled (X insert) | Key must be short (practical) |
| Historically: unbreakable for decades | Modern computers can break it |

**How it was broken:** digraph frequency analysis + crib-dragging (guessing plaintext pairs). WWII cryptanalysts routinely broke Playfair.

---

## Programming Analogy

```
Playfair = a 2D grid with a stateful position-lookup

5×5 grid  = a hash map (letter → (row, col)) with
            rectangular wrap-around semantics

Digraph prep = a tokenization step
  (merge duplicates, pad odd length)
  = like preprocessing input into fixed-size chunks
  (similar to block padding in AES)

Three rules = a conditional state machine:
  if same row:     shift horizontally
  if same column:  shift vertically
  else:            transpose (swap column indices)

J→I mapping = lossy normalization
  (like normalizing unicode to ASCII — information lost)

The grid is deterministic given a key:
  same key = same grid = same encryption
  = a key-dependent permutation (like a keyed hash)
```

---

## Code Notes (Gotchas)

| Issue | The Code's Fix |
|---|---|
| `getchar()` after `scanf` | Flushes leftover newline from input buffer |
| `fgets` + `strcspn` | Safely removes trailing newline |
| J → I in key AND plaintext | Consistent mapping (25-letter alphabet) |
| Same-char X insertion | Prevents LL encrypting as itself |
| `% 5` for row/col shift | Wrap-around for grid boundaries |
| `(c + 1) % 5` for right shift | `0→1, 1→2, 2→3, 3→4, 4→0` |

---

## Revision Summary

| Concept | Definition |
|---|---|
| Digraph substitution | Encrypts letter pairs, not singles |
| 5×5 Grid | Key fills first, remaining letters follow |
| J → I | Maps 26 letters to 25 cells |
| X insertion | Splits duplicate adjacent letters |
| Same row → shift right | Wrap-around |
| Same column → shift down | Wrap-around |
| Rectangle → swap corners | Swap column indices |
| Decryption | Reverse direction for row/column rules |

- Two-letter encryption defeats single-letter frequency analysis
- Three geometric rules from grid position
- J merged with I (25-letter alphabet)
- Block cipher precursor — like a manual AES
