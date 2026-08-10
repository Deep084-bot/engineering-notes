# Caesar Cipher

## What It Is

The **Caesar cipher** is one of the oldest and simplest encryption techniques — a substitution cipher where each letter is shifted by a fixed number of positions in the alphabet.

```
Plaintext:  HELLO
Shift by 3: KHOOR
```

---

## How It Works

Each letter is shifted **s** positions forward (modulo 26).

```
Encryption:  E(x) = (x + s) mod 26
Decryption:  D(y) = (y − s) mod 26

x = plaintext letter position (0-25, a = 0, z = 25)
y = ciphertext letter position
s = shift key
```

**Example (shift = 3):**

```
Plaintext:  hello
Positions:  h(7) e(4) l(11) l(11) o(14)

Encrypt:    (7+3)=10→k  (4+3)=7→h  (11+3)=14→o  (14+3)=17→r
Ciphertext: khoor

Decrypt:    (10−3)=7→h  (7−3)=4→e  (14−3)=11→l  (17−3)=14→o
Plaintext:  hello
```

---

## Why Modulo 26

The alphabet wraps around — after `z` comes `a`.

```
y = x + s, but if the result exceeds 25, wrap around:

x = 24 (y), s = 5 → 24 + 5 = 29 → 29 mod 26 = 3 (d)

Without modulo, the shift would leave the alphabet.
```

---

## Security — Why It's Weak

The Caesar cipher has only **25 possible keys** (s = 1 to 25). It's trivially broken by brute force.

```
Try all 25 shifts, one will read as English:
  khoor → shift 23 → hello ✓

Or use frequency analysis: the most common letter in
  English ('e') maps to the most frequent ciphertext letter.
```

| Property | Caesar Cipher |
|---|---|
| Key space | 25 keys only |
| Key type | Symmetric (same shift to encrypt/decrypt) |
| Brute force | Instant (try all 25) |
| Use today | None for real security (educational only) |

---

## The Code

```c
#include <stdio.h>
#include <stdlib.h>

int main(){
    int s, n;
    printf("Enter the shift key : ");
    scanf("%d", &s);
    printf("Enter the length of the string : ");
    scanf("%d", &n);
    while (getchar() != '\n'); // flush newline left by scanf

    char *ch;
    ch = (char *)malloc(n * sizeof(char));
    if(ch == NULL){
        printf("Failed to load memory.");
        return 1;
    }
    for(int i = 0; i < n; i++) scanf("%c", &ch[i]);

    // Encryption: E(x) = (x + s) mod 26
    printf("Encrypted string : ");
    for(int i = 0; i < n; i++){
        char a = (((ch[i] - 'a') + s) % 26) + 'a';
        printf("%c", a);
    }
    printf("\n");

    char *arr;
    arr = (char *)malloc(n * sizeof(char));
    printf("Enter the string to be decrypted : ");
    for(int i = 0; i < n; i++) scanf("%c", &arr[i]);

    // Decryption: D(y) = (y - s) mod 26
    printf("Decrypted string : ");
    for(int i = 0; i < n; i++){
        char a = (((arr[i] - 'a') - s) % 26) + 'a';
        printf("%c", a);
    }
    printf("\n");
    return 0;
}
```

**Code walkthrough:**

| Line | Purpose |
|---|---|
| `scanf("%d", &s)` | Read the shift key |
| `ch[i] - 'a'` | Convert char to position (0-25) |
| `(... + s) % 26` | Apply the shift with wrap-around |
| `+ 'a'` | Convert position back to a char |

---

## Limitations

- **Lowercase only** — the code assumes all input is lowercase (`a`-`z`). Uppercase or digits will produce wrong output.
- **No negative-shift handling** — `(y − s) % 26` can be negative in C. Add 26 before taking the mod if `s > y`.

```
C's % operator returns a negative result for negative operands.

Correct: ((y - s) % 26 + 26) % 26
Naive:  (y - s) % 26   ← can be negative
```

---

## Programming Analogy

```
Caesar Cipher = naive obfuscation, not encryption

A fixed shift is like ROT13 or base64 — it looks encoded
  but has no real key. Security through obscurity.

Position mapping (x + s) mod 26 = a fixed rotation
  of the alphabet — like a simple look-up table.

Brute-forcing 25 keys = exhaustive search over a tiny
  key space (like a 5-bit key). Trivial to crack.

Lesson: real security needs a LARGE key space and
  resistance to frequency analysis (which is why the
  Affine cipher and modern AES are far stronger).
```

---

## Revision Summary

| Concept | Value |
|---|---|
| Algorithm | Shift each letter by s positions |
| Encryption | E(x) = (x + s) mod 26 |
| Decryption | D(y) = (y − s) mod 26 |
| Key space | 25 keys |
| Security | None (educational only) |
| Weakness | Tiny key space + frequency analysis |

- Simplest substitution cipher
- Wrap-around handled by mod 26
- Broke by brute force in seconds
