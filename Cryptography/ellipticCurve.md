# Elliptic Curve Cryptography (ECC)

## What It Is

**Elliptic Curve Cryptography** is a public-key cryptosystem built on the algebra of points on an elliptic curve. It provides the same services as RSA — key exchange, encryption, digital signatures — but with much smaller keys.

```
RSA-3072  ≈  ECC-256  in security strength

Smaller keys → less bandwidth, less storage, less power.
```

ECC is used in:
- TLS/HTTPS (ECDHE key exchange)
- SSH (ECDSA / Ed25519 keys)
- Bitcoin and Ethereum (secp256k1)
- Apple Secure Enclave, Android Keystore
- Smart cards, IoT devices
- Signal, WhatsApp (Curve25519)

---

## The Curve

An elliptic curve over a finite field `F_p` (where `p` is prime) is defined by:

```
y² ≡ x³ + a·x + b  (mod p)
```

```
a, b, p are the curve parameters.
The curve is nonsingular when:

  4a³ + 27b² ≢ 0  (mod p)
```

For this note (and the C program):

```
a = 2, b = 3

p = 97:   4(2³) + 27(3²) = 32 + 243 = 275 ≡ 81 (mod 97)  ≠ 0 ✓
p = 101:  275 ≡ 73 (mod 101)                             ≠ 0 ✓
```

A point `(x, y)` is on the curve if it satisfies the equation. The point at infinity `O` (written `Θ` in the code) is also part of the group, but has no affine coordinates.

---

## Points on the Curve

Only some pairs `(x, y)` satisfy the equation. For `p = 97`, the affine points are:

```
(0, 10)   (0, 87)   (1, 43)   (1, 54)   (3, 6)
(3, 91)   (4, 47)   (4, 50)   (10, 21)  (10, 76)
...
```

```
Number of affine points = 99
Order of the group (including Θ) = 100
```

For `p = 101`:

```
Number of affine points = 95
Order of the group (including Θ) = 96
```

### Hasse's bound

The number of points `#E(F_p)` is close to `p + 1`:

```
| #E(F_p) − (p + 1) | ≤ 2√p

p = 97:  |100 − 98| = 2 ≤ 2√97 ≈ 19.7  ✓
p = 101: |96 − 102| = 6 ≤ 2√101 ≈ 20.1 ✓
```

```
For a valid curve, #E(F_p) is always in the Hasse interval.
The group order matters for security: it should have a large prime factor.
```

---

## The Group Law

The points on a nonsingular elliptic curve form an **abelian group** under a special addition operation.

### Identity

```
P + Θ = Θ + P = P
```

`Θ` is the point at infinity — the "zero" of the group.

### Inverse

```
−P = (x, −y mod p)

P + (−P) = Θ
```

```
On the curve, (x, y) and (x, −y) are mirror images across the x-axis.
A line through them is vertical, so it meets the curve only at infinity.
```

### Point addition (P ≠ Q)

Draw a line through `P` and `Q`. It intersects the curve at a third point `R`. Reflect `R` across the x-axis to get `P + Q`.

```
        y
        │        P
        │       ●
        │      / \
        │     /   ● Q
        │    /     \
        │   ●───────●──── x
        │  P+Q
```

Formula:

```
λ = (y₂ − y₁) / (x₂ − x₁)  mod p

x₃ = λ² − x₁ − x₂          mod p
y₃ = λ·(x₁ − x₃) − y₁      mod p
```

### Point doubling (P = Q)

When `P = Q`, the line becomes the tangent at `P`.

```
λ = (3x₁² + a) / (2y₁)     mod p

x₃ = λ² − 2x₁              mod p
y₃ = λ·(x₁ − x₃) − y₁      mod p
```

### Vertical case

```
If x₁ = x₂ and y₁ + y₂ ≡ 0 (mod p):
    P + Q = Θ
```

This includes `P + (−P)` and the doubling of a point with `y = 0`.

---

## Worked Example (p = 97)

Curve: `y² = x³ + 2x + 3 mod 97`

```
P = (0, 87)
Q = (1, 43)
```

Both are on the curve. Add them:

```
λ = (43 − 87) / (1 − 0)
  = −44 / 1
  = −44 mod 97
  = 53

x₃ = 53² − 0 − 1
   = 2809 − 1
   = 2808 mod 97
   = 92

y₃ = 53·(0 − 92) − 87
   = −4876 − 87
   = −4963 mod 97
   = 81
```

```
P + Q = (92, 81)
```

### Doubling `P`

```
λ = (3·0² + 2) / (2·87)
  = 2 / 174
  = 2 · 77⁻¹ mod 97
  = 2 · 63
  = 126 mod 97
  = 29

x₃ = 29² − 0 − 0 = 841 mod 97 = 65
y₃ = 29·(0 − 65) − 87 = −1972 mod 97 = 65
```

```
2P = (65, 65)
```

---

## Scalar Multiplication

ECC computes `k·P` by repeated addition — the same idea as square-and-multiply, but with points.

```
Double-and-add:

result = Θ
addend = P

while k > 0:
    if k is odd:  result = result + addend
    addend = addend + addend
    k = k >> 1

return result
```

```
Complexity: O(log k) point additions instead of O(k).
```

```
k·P is easy to compute.
Given P and Q = k·P, finding k is hard.  ← ECDLP
```

---

## The Hard Problem: ECDLP

**Elliptic Curve Discrete Logarithm Problem:**

```
Given P and Q = k·P on the curve, find k.
```

```
Forward:  P → k·P        easy (double-and-add)
Reverse:  k·P → k        believed hard
```

The best known classical algorithms take about `O(√n)` time (Pollard's rho), where `n` is the order of the point. This is why a 256-bit curve gives roughly 128-bit security.

```
No efficient classical algorithm is known.
Quantum computers would break ECC with Shor's algorithm,
just as they would break RSA.
```

---

## ECDH — Key Exchange

Elliptic Curve Diffie–Hellman lets two parties agree on a shared secret over a public channel.

```
Public parameters: curve, base point G, order n.

Alice:
  private a  (random)
  public  A = a·G

Bob:
  private b  (random)
  public  B = b·G

Exchange A and B.

Alice computes:  S = a·B = a·b·G
Bob computes:    S = b·A = b·a·G

Same point S → derive a symmetric key from S.x (via a KDF).
```

```
An eavesdropper sees G, A, B but cannot compute a·b·G
without solving ECDLP.
```

---

## ECDSA — Signatures

Elliptic Curve Digital Signature Algorithm signs a hash of a message.

```
Key generation:
  private d  (random)
  public  Q = d·G

Sign(message):
  1. z = hash(message)
  2. choose random k
  3. R = k·G
  4. r = R.x mod n
  5. s = k⁻¹·(z + r·d) mod n
  6. signature = (r, s)

Verify(message, signature):
  1. z = hash(message)
  2. w  = s⁻¹ mod n
  3. u₁ = z·w mod n
  4. u₂ = r·w mod n
  5. R' = u₁·G + u₂·Q
  6. valid if R'.x mod n == r
```

```
Critical: k must be random and never reused.
Reusing k (or using a biased RNG) leaks the private key.
The Sony PS3 ECDSA failure is the classic example.
```

---

## The Code (C)

```c
#include <stdio.h>


#define THETA_X 0
#define THETA_Y 1

int mod(int a, int p) {
    int r = a % p;
    if (r < 0) r = r + p;
    return r;
}

int mod_add(int a, int b, int p) {
    if(a >= p - b) return (a + b - p);
    else return (a + b);
}

int mod_sub(int a, int b, int p) {
    if(a >= b) return (a - b);
    else return (p - (b - a));
}

int modInverse(int a, int p) {
    int old_r = mod(a, p), r = p, old_s = 1, s = 0;
    int q, temp;
    while(r != 0){
        q = old_r / r;
        temp = old_r - (q * r);
        old_r = r;
        r = temp;

        temp = old_s - (q * s);
        old_s = s;
        s = temp;
    }
    return mod(old_s, p);
}

int isIdentity(int x, int y) {
    return (x == THETA_X && y == THETA_Y);
}

int isOnCurve(int x, int y, int a, int b, int p) {
    long left, right;
    left  = mod((long)y * y, p);
    right = mod(((long)x * x % p) * x + (long)a * x + b, p);
    return (left == right);
}

int isValidPoint(int x, int y, int a, int b, int p) {
    if (x < 0 || x >= p || y < 0 || y >= p) return 0;
    if (isIdentity(x, y)) return 1;
    return isOnCurve(x, y, a, b, p);
}

int listPoints(int a, int b, int p, int xs[], int ys[]) {
    int count = 0;
    int x, y;
    for (x = 0; x < p; x++) {
        for (y = 0; y < p; y++) {
            if (isOnCurve(x, y, a, b, p)) {
                xs[count] = x;
                ys[count] = y;
                count++;
            }
        }
    }
    return count;
}

void addPoints(int x1, int y1, int x2, int y2, int a, int p, int *x3, int *y3) {
    long ans, numerator, denominator;
    // Case 0: if either point is the identity, return the other one.
    if(isIdentity(x1, y1)) {
        *x3 = x2;
        *y3 = y2;
        return;
    }
    if(isIdentity(x2, y2)) {
        *x3 = x1;
        *y3 = y1;
        return;
    }
    if(x1 == x2) {
        // Same x coordinate.
        if(mod((long)y1 + y2, p) == 0){
            *x3 = THETA_X;
            *y3 = THETA_Y;
            return;
        }
        // Case 2: tangent / doubling (P == Q).
        numerator = mod(3L * x1 * x1 + a, p);
        denominator = mod(2L * y1, p);
    }else {
        // Case 1: chord (x1 != x2).
        numerator = mod((long)y2 - y1, p);
        denominator = mod((long)x2 - x1, p);
    }
    ans = mod(numerator * modInverse(denominator, p), p);
    *x3 = (int)mod(ans * ans - x1 - x2, p);
    *y3 = (int)mod(ans * (x1 - (long)*x3) - y1, p);
}

int main(){
    int a = 2, b = 3;
    int primes[2] = {97, 101};
    int xs[200], ys[200];
    int k, p, count, i;
    int x1, y1, x2, y2, x3, y3;
    for (k = 0; k < 2; k++) {
        p = primes[k];
        count = listPoints(a, b, p, xs, ys);

        printf("Points on the curve:\n");

        for (i = 0; i < count; i++) {
            printf("(%d, %d)  ", xs[i], ys[i]);
            if ((i + 1) % 5 == 0) printf("\n");
        }
        if (count % 5 != 0) printf("\n");

        printf("\nNumber of affine points = %d\n", count);
        printf("Order of the group (including Theta) = %d\n\n", count + 1);
    }
    printf("Enter P as:  x1 y1 : ");
    scanf("%d %d", &x1, &y1);
    printf("Enter Q as:  x2 y2 : ");
    scanf("%d %d", &x2, &y2);
    if (!isValidPoint(x1, y1, a, b, p)) printf("ERROR: P = (%d, %d) is not on the curve.\n", x1, y1);
    if (!isValidPoint(x2, y2, a, b, p)) printf("ERROR: Q = (%d, %d) is not on the curve.\n", x2, y2);
    addPoints(x1, y1, x2, y2, a, p, &x3, &y3);
    if (isIdentity(x3, y3)) printf("P + Q = Theta (the point at infinity)\n");
    else {
        printf("P + Q = (%d, %d)\n", x3, y3);
        printf("Result is on the curve? %s\n", isOnCurve(x3, y3, a, b, p) ? "YES" : "NO");
    }
    return 0;
}
```

---

## Code Walkthrough

| Function | Purpose |
|---|---|
| `mod` | Positive modulo (fixes C's negative remainder) |
| `mod_add` / `mod_sub` | Modular addition/subtraction helpers (unused) |
| `modInverse` | Extended Euclidean inverse modulo `p` |
| `isIdentity` | Checks for the `(0,0)` sentinel used for `Θ` |
| `isOnCurve` | Checks `y² ≡ x³ + ax + b (mod p)` |
| `isValidPoint` | Range check + identity + curve check |
| `listPoints` | Brute-forces all `(x, y)` pairs on the curve |
| `addPoints` | Implements identity, inverse, doubling, and chord cases |
| `main` | Lists points for `p = 97` and `p = 101`, then adds two points |

### `addPoints` cases

```
1. P = Θ       → return Q
2. Q = Θ       → return P
3. x1 = x2 and y1 + y2 ≡ 0 (mod p)
               → return Θ          (P + (−P), or vertical tangent)
4. x1 = x2     → doubling: λ = (3x1² + a) / (2y1)
5. x1 ≠ x2     → chord:    λ = (y2 − y1) / (x2 − x1)
```

Then:

```
x3 = λ² − x1 − x2
y3 = λ(x1 − x3) − y1
```

---

## Sample Run

### Points for both primes

```
Points on the curve:
(0, 10)  (0, 87)  (1, 43)  (1, 54)  (3, 6)
...

Number of affine points = 99
Order of the group (including Theta) = 100

Points on the curve:
(1, 39)  (1, 62)  (3, 6)  (3, 95)  (5, 21)
...

Number of affine points = 95
Order of the group (including Theta) = 96
```

### Adding two valid points (p = 101, the code's final `p`)

```
Enter P as:  x1 y1 : 1 62
Enter Q as:  x2 y2 : 3 6
P + Q = (73, 35)
Result is on the curve? YES
```

### Doubling

```
Enter P as:  x1 y1 : 1 62
Enter Q as:  x2 y2 : 1 62
P + Q = (79, 44)
Result is on the curve? YES
```

### Points valid for p = 97 (demonstrates the bug)

```
Enter P as:  x1 y1 : 0 87
Enter Q as:  x2 y2 : 1 43
ERROR: P = (0, 87) is not on the curve.
ERROR: Q = (1, 43) is not on the curve.
P + Q = ...          ← computed modulo 101, not 97
Result is on the curve? NO
```

```
The exact wrong point depends on the platform's integer sizes.
The important part is the error: these points belong to the
p = 97 curve, but the addition still runs modulo p = 101.

Also note: `mod` takes an `int` parameter, so the `long`
expressions in addPoints are narrowed before the modulo call.
For small p = 101 it happens to work; for larger p it breaks.
```

---

## Known Issues in the Implementation

### 1. Addition always uses `p = 101`

```
The loop ends with p = primes[1] = 101.
Nothing resets p before validating or adding the entered points.

So even though the program prints both curves, any addition is
performed modulo 101. Points that are valid only on the p = 97
curve are rejected, and the result may not even be on the curve.
```

Fix: ask which curve to use, or store the chosen `p` explicitly.

```c
printf("Enter curve prime (97 or 101): ");
scanf("%d", &p);
```

### 2. `(0, 0)` is used as the identity sentinel

```
(0,0) is not on these curves, so it works here.
But in general (0,0) can be a valid point (when b ≡ 0 mod p),
which would collide with the identity representation.

Better: represent Θ with a separate flag, or use a coordinate
outside the field (e.g., x = −1) and validate accordingly.
```

### 3. Fixed-size `int` arithmetic

```
ans * ans, numerator * denominator, etc. can overflow for larger p.
For cryptographic sizes (256 bits), you need big integers and
constant-time modular arithmetic — not int/long.
```

### 4. No primality or curve validation

```
The program assumes p is prime and the curve is nonsingular.
Real ECC implementations validate domain parameters and
check that points are on the curve (to prevent invalid-curve attacks).
```

### 5. No scalar multiplication or protocols

```
The program implements point addition only.
Real ECC needs:
  - scalar multiplication (double-and-add)
  - ECDH key exchange
  - ECDSA signatures
  - secure randomness
  - point compression and serialization
```

### 6. `modInverse` assumes the denominator is invertible

```
For valid points on a prime curve, the denominators used here
are non-zero and invertible (the x1 = x2 vertical case is handled).
Still, a production implementation must check gcd(denominator, p) = 1.
```

### 7. Unused helpers

```
mod_add and mod_sub are defined but never called.
```

---

## What Real ECC Needs

```
1. A standard curve:
     NIST P-256, Curve25519, secp256k1, P-384
2. A large prime-order subgroup:
     cofactor h = 1 is preferred
3. Secure random generation for private keys and ECDSA nonces
4. Point validation and subgroup checks
5. Constant-time scalar multiplication
6. A KDF for ECDH shared secrets
7. A hash function for ECDSA (e.g., SHA-256)
8. Serialization formats (SEC1, compressed/uncompressed points)
```

```
Never invent your own curve or your own point arithmetic for production.
Use a vetted library: OpenSSL, libsodium, BoringSSL, Go crypto/elliptic,
RustCrypto, etc.
```

---

## ECC vs RSA

| | RSA | ECC |
|---|---|---|
| Hard problem | Integer factorisation | Elliptic curve discrete log |
| Key size for ~128-bit security | 3072 bits | 256 bits |
| Key generation | Slow (prime generation) | Faster |
| Encryption/signature speed | Slower | Faster |
| Bandwidth | Larger keys and signatures | Compact keys |
| Implementation risk | Simpler math | Easy to get wrong (invalid curves, nonces) |
| Common curves | — | P-256, Curve25519, secp256k1 |

```
ECC's smaller keys are a big advantage on mobile, IoT,
smart cards, and bandwidth-constrained links.
```

---

## Revision Summary

| Concept | Value |
|---|---|
| Curve equation | y² = x³ + ax + b (mod p) |
| Nonsingular condition | 4a³ + 27b² ≠ 0 (mod p) |
| Identity | Point at infinity Θ |
| Inverse | −P = (x, −y mod p) |
| Addition slope | λ = (y₂ − y₁)/(x₂ − x₁) |
| Doubling slope | λ = (3x₁² + a)/(2y₁) |
| Result | x₃ = λ² − x₁ − x₂, y₃ = λ(x₁ − x₃) − y₁ |
| Scalar multiplication | Double-and-add, O(log k) |
| Hard problem | ECDLP: find k from P and k·P |
| Key exchange | ECDH: a·B = a·b·G = b·A |
| Signatures | ECDSA: (r, s) using a random nonce k |
| 256-bit ECC | ≈ 3072-bit RSA security |
| Common curves | P-256, Curve25519, secp256k1 |

- ECC adds points on a curve; the group law replaces modular multiplication
- `P + Q` uses a chord; `P + P` uses a tangent
- `P + (−P) = Θ`
- `k·P` is fast; recovering `k` is the ECDLP
- ECDH agrees on a shared point; ECDSA signs with a random nonce
- Never reuse an ECDSA nonce
- Custom curves and custom arithmetic are dangerous
- For production, use a vetted library

Previous: [RSA](rsa.md).
