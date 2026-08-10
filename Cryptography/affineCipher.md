# Affine Cipher

## What It Is

The **Affine cipher** is a substitution cipher that combines multiplication and addition:

```
Encryption:  E(x) = (a·x + b) mod 26
Decryption:  D(y) = a⁻¹·(y − b) mod 26

x = plaintext position (0-25), y = ciphertext position
a = multiplicative key, b = shift key
a⁻¹ = modular inverse of a (mod 26)
```

It's a generalization of the Caesar cipher (Caesar is the case `a = 1`).

---

## The Key Requirements

### 1. `a` must be coprime with 26

The multiplicative key `a` must have `gcd(a, 26) = 1`. This is what makes the modular inverse exist.

```
Why? Decryption needs a⁻¹ such that a·a⁻¹ ≡ 1 (mod 26).
  a⁻¹ exists ONLY if gcd(a, 26) = 1.

Valid a values (coprime with 26): 1, 3, 5, 7, 9, 11, 15, 17, 19, 21, 23, 25
Invalid (gcd ≠ 1): 2, 4, 6, 8, 10, ... (share a factor with 26)

gcd(5, 26) = 1 → 5 is a valid key
gcd(10, 26) = 2 → 10 is NOT valid
```

### 2. Modular inverse

```
a⁻¹ mod 26 is the number that satisfies: a · a⁻¹ ≡ 1 (mod 26)

5⁻¹ mod 26 = 21  (because 5 × 21 = 105 ≡ 1 mod 26)
```

**How to find it:** the Extended Euclidean Algorithm (covered below).

---

## Example Walkthrough

```
Keys: a = 5, b = 8

Encrypt "hi":  h = 7, i = 8

E(7) = (5×7 + 8) mod 26 = 43 mod 26 = 17 → r
E(8) = (5×8 + 8) mod 26 = 48 mod 26 = 22 → w
Ciphertext: "rw"

Decrypt:  a⁻¹ = 5⁻¹ mod 26 = 21

D(17) = 21 × (17 − 8) mod 26 = 21 × 9 = 189 mod 26 = 7 → h
D(22) = 21 × (22 − 8) mod 26 = 21 × 14 = 294 mod 26 = 8 → i
Plaintext: "hi" ✓
```

---

## Key Space — Stronger Than Caesar

| Cipher | Keys | Security |
|---|---|---|
| Caesar | 25 | Trivial |
| Affine | 12 valid `a` × 26 `b` = **312 keys** | Better, still weak |

```
Brute force: 312 combinations — still instant to crack
  on a computer. Affine is a teaching cipher, not secure
  for real data.
```

---

## The Math Building Blocks

### GCD (Greatest Common Divisor)

```
Euclidean algorithm:
  gcd(x, y) = gcd(y, x mod y), until y = 0

gcd(12, 8) = gcd(8, 4) = gcd(4, 0) = 4
```

### Extended Euclidean Algorithm

Finds `x` and `y` such that: `a·x + b·y = gcd(a, b)` — beyond just the gcd, it gives the coefficients needed for the modular inverse.

```
Extended gcd(5, 26):
  finds x = 21, y = −4 such that 5×21 + 26×(−4) = 1

Since gcd = 1, 5 × 21 ≡ 1 (mod 26) → 5⁻¹ = 21
```

### Modular Inverse

```
modInverse(a, m):
  g = extendedGCD(a, m, &x, &y)
  if g != 1: no inverse exists (a not coprime with m)
  else: return (x % m + m) % m   // make positive
```

---

## The Code

```c
#include <stdio.h>
#include <stdlib.h>

int gcd(int x, int y);
int extendedGCD(int a, int b, int *x, int *y);
int modInverse(int a, int m);

int main()
{
    int n, a, b = 20, m = 26;
    printf("Enter the length of string: ");
    scanf("%d", &n);
    while (1){
        printf("Enter key 'a' (must be coprime with 26): ");
        scanf("%d", &a);
        if (gcd(a, m) == 1) break;
        printf("Invalid key! %d is not coprime with 26.\n", a);
    }
    char *ch = malloc(n * sizeof(char));
    if (ch == NULL){
        printf("Memory allocation failed.\n");
        return 1;
    }
    getchar(); // remove the '\n' left by scanf
    printf("Enter the string to be encrypted: ");
    for (int i = 0; i < n; i++) scanf("%c", &ch[i]);

    // E(x) = (a*x + b) mod 26
    printf("Encrypted string: ");
    for (int i = 0; i < n; i++){
        int x = ch[i] - 'a';
        int encrypted = (a * x + b) % m;
        char c = encrypted + 'a';
        printf("%c", c);
    }
    printf("\n");

    int aInv = modInverse(a, m);
    printf("Modular inverse of a = %d\n", aInv);

    if (aInv == -1){
        printf("Modular inverse does not exist.\n");
        free(ch);
        return 1;
    }

    char *arr = malloc(n * sizeof(char));
    if (arr == NULL){
        printf("Memory allocation failed.\n");
        free(ch);
        return 1;
    }
    getchar();
    printf("Enter the string for decryption: ");
    for (int i = 0; i < n; i++) scanf("%c", &arr[i]);

    // D(y) = a^-1 * (y - b) mod 26
    printf("Decrypted string: ");
    for (int i = 0; i < n; i++){
        int y = arr[i] - 'a';
        int decrypted = aInv * (y - b);
        decrypted = decrypted % m;
        if (decrypted < 0) decrypted += m; // fix negative remainder
        char c = decrypted + 'a';
        printf("%c", c);
    }
    printf("\n");

    free(ch);
    free(arr);
    return 0;
}

int gcd(int x, int y){
    if (y == 0) return x;
    return gcd(y, x % y);
}

// ======================================================
// EXTENDED EUCLIDEAN ALGORITHM
// Finds: gcd(a, b)
// and also finds x and y such that: a*x + b*y = gcd(a, b)
// ======================================================

int extendedGCD(int a, int b, int *x, int *y){
    if (a == 0){
        *x = 0;
        *y = 1;
        return b;
    }
    int x1, y1;
    int g = extendedGCD(b % a, a, &x1, &y1);
    *x = y1 - (b / a) * x1;
    *y = x1;
    return g;
}

// ======================================================
// MODULAR INVERSE
// Finds: a^-1 mod m  such that: a * a^-1 ≡ 1 (mod m)
// ======================================================

int modInverse(int a, int m){
    int x, y;
    int g = extendedGCD(a, m, &x, &y);
    if (g != 1) return -1;              // inverse needs gcd = 1
    return (x % m + m) % m;             // make positive
}
```

**Code walkthrough:**

| Part | Purpose |
|---|---|
| Input validation loop | Keeps asking for `a` until `gcd(a, 26) = 1` |
| Encryption loop | Applies `(a·x + b) mod 26` |
| `modInverse` | Computes `a⁻¹` via Extended Euclidean |
| Decryption loop | Applies `a⁻¹·(y − b) mod 26`, fixes negative remainder |
| `free()` | Releases allocated memory (no leaks) |

---

## Negative Remainder Gotcha

C's `%` can return negative values.

```
In C:  (21 × (−12)) % 26 can be negative

Fix: after taking the mod, add 26 if the result is negative:
  if (decrypted < 0) decrypted += 26;

Python's % always returns non-negative, but C/C++ does not.
```

---

## Programming Analogy

```
Affine Cipher = a small, teachable crypto algorithm

Key constraint (a coprime with m) = a pre-condition check,
  like validating inputs before running a function.

Modular inverse = the "undo" operation. Encryption must
  be reversible — a⁻¹ guarantees round-tripping.
  (like an invertible transform in a pipeline)

Extended Euclidean = the algorithm that finds the inverse,
  not just the gcd — like a solver that returns both
  the answer and the coefficients.

312 keys still trivially brute-forced = small key space;
  modern ciphers (AES-256) have 2²⁵⁶ keys.
  Lesson: cipher structure matters, but key space
  is what makes cracking infeasible.
```

---

## Revision Summary

| Concept | Value |
|---|---|
| Encryption | E(x) = (a·x + b) mod 26 |
| Decryption | D(y) = a⁻¹·(y − b) mod 26 |
| Requirement | gcd(a, 26) = 1 |
| Modular inverse | a·a⁻¹ ≡ 1 (mod 26) |
| Key space | 312 keys (12 × 26) |
| Special case | Caesar cipher = affine with a = 1 |
| Security | Educational only |

- `a` must be coprime with 26
- Inverse found via Extended Euclidean Algorithm
- C's `%` needs negative-result handling
- Stronger than Caesar, still trivially broken
