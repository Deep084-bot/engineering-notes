# RSA (Rivest–Shamir–Adleman)

## What It Is

**RSA** is an asymmetric (public-key) cryptosystem. It uses two mathematically linked keys:

```
Public key  (e, n)  → given to everyone; used to encrypt and verify
Private key (d, n)  → kept secret; used to decrypt and sign
```

```
Plaintext m ──► c = m^e mod n ──► Ciphertext c
Ciphertext c ──► m = c^d mod n ──► Plaintext m
```

Its security rests on the **integer factorisation problem**: multiplying two large primes is easy, but factoring their product back into the primes is believed to be infeasible for large `n`.

```
p × q = n        easy
n → p, q         hard
```

RSA is used for:
- Key exchange in TLS (encrypting a session key)
- Digital signatures (RSA-PSS, PKCS#1 v1.5)
- Certificates (X.509)
- SSH authentication
- Code signing

---

## The Math Behind It

### Euler's totient

For `n = p × q` where `p` and `q` are distinct primes:

```
φ(n) = (p − 1)(q − 1)
```

`φ(n)` counts the integers from `1` to `n` that are coprime with `n`.

### Euler's theorem

If `gcd(a, n) = 1`:

```
a^φ(n) ≡ 1 (mod n)
```

### The key relationship

Choose `e` such that `gcd(e, φ(n)) = 1`. Then `e` has a modular inverse `d`:

```
e · d ≡ 1 (mod φ(n))
```

So `e · d = 1 + k · φ(n)` for some integer `k`.

### Why decryption works

```
c^d = (m^e)^d = m^(e·d) = m^(1 + k·φ(n))
    = m · (m^φ(n))^k
    ≡ m · 1^k        (Euler's theorem)
    ≡ m (mod n)
```

For messages not coprime with `n`, the same result follows from the Chinese Remainder Theorem applied modulo `p` and `q`.

---

## Key Generation

```
1. Pick two large distinct primes p and q.
2. Compute n = p × q.                      ← public modulus
3. Compute φ(n) = (p − 1)(q − 1).
4. Choose e such that 1 < e < φ(n) and gcd(e, φ(n)) = 1.
   Common choice: e = 65537 (0x10001).
5. Compute d = e^−1 mod φ(n).              ← private exponent
6. Public key  = (e, n).
7. Private key = (d, n).
8. Destroy p and q (and φ(n)) securely.
```

```
Security relies on:
  - n being hard to factor
  - d being hard to derive without φ(n)
  - p and q remaining secret
```

---

## Encryption and Decryption

```
Encrypt:  c = m^e mod n
Decrypt:  m = c^d mod n

where 0 ≤ m < n
```

```
The message m must be smaller than n.
This is why real RSA does not encrypt a whole file directly —
it encrypts a symmetric session key (hybrid encryption).
```

### Digital signatures

RSA can also sign. The operations are reversed:

```
Sign:    s = H(m)^d mod n      (using the private key)
Verify:  H(m) ?= s^e mod n    (using the public key)
```

```
Encryption: public key locks, private key unlocks
Signature:  private key signs, public key verifies
```

In practice, signatures use **RSA-PSS** with a hash function, not raw RSA.

---

## Worked Example

Use small primes to see the whole process.

```
p = 61
q = 53

n   = 61 × 53 = 3233
φ(n) = 60 × 52 = 3120

Choose e = 17   (gcd(17, 3120) = 1)

d = 17^−1 mod 3120 = 2753
Check: 17 × 2753 = 46801 = 15 × 3120 + 1  ✓

Public key  = (17, 3233)
Private key = (2753, 3233)
```

### Encrypt `m = 65`

```
c = 65^17 mod 3233 = 2790
```

### Decrypt

```
m = 2790^2753 mod 3233 = 65   ✓
```

### ASCII example (the C program)

```
'H' = 72  →  72^17  mod 3233 = 3000
'I' = 73  →  73^17  mod 3233 = 1486

Ciphertext: 3000 1486

Decrypt:
  3000^2753 mod 3233 = 72 = 'H'
  1486^2753 mod 3233 = 73 = 'I'
```

---

## Why `e` Must Be Coprime With `φ(n)`

```
d = e^−1 mod φ(n) exists  ⇔  gcd(e, φ(n)) = 1
```

If `gcd(e, φ(n)) = g > 1`, then `e·d ≡ 1 (mod φ(n))` has no solution, so there is no private exponent that can undo the encryption.

```
φ(3233) = 3120 = 2^4 × 3 × 5 × 13

Valid e:   3, 7, 11, 17, 19, ...   (coprime with 3120)
Invalid e: 2, 4, 5, 6, 10, 12, ... (share a factor)
```

The C program checks this and refuses to continue with an invalid `e`.

---

## Square-and-Multiply (Fast Modular Exponentiation)

Computing `m^e mod n` by multiplying `m` exactly `e` times is far too slow — `e` can be thousands of bits.

**Square-and-multiply** uses the binary representation of the exponent.

```
result = 1
base   = m mod n

while e > 0:
    if e is odd:  result = (result × base) mod n
    base = (base × base) mod n
    e = e >> 1

return result
```

```
Complexity: O(log e) multiplications instead of O(e).
```

### Example: `e = 17 = 10001₂`

```
Start: result = 1, base = m

bit 1 (LSB): odd  → result = result × base
             base = base²
bit 0:       even → base = base²
bit 0:       even → base = base²
bit 0:       even → base = base²
bit 1 (MSB): odd  → result = result × base

Only 5 squarings and 2 multiplications for e = 17.
```

---

## Modular Inverse (Extended Euclidean Algorithm)

To compute `d = e^−1 mod φ(n)`, find integers `x, y` such that:

```
e·x + φ(n)·y = 1
```

Then `x mod φ(n)` is the inverse.

The C program uses an iterative form of the extended Euclidean algorithm:

```
m0 = m
x0 = 0, x1 = 1

while a > 1:
    q  = a / m
    (a, m) = (m, a mod m)
    (x0, x1) = (x1, x0 − q·x1)

if x1 < 0: x1 += m0
return x1
```

For `e = 17`, `φ = 3120`:

```
17 × 2753 ≡ 1 (mod 3120)
→ d = 2753
```

---

## The Code (C)

```c
#include <stdio.h>
#define MAX 1024

long long squareAndMultiply(long long base, long long exp, long long mod){
    long long result = 1;
    base %= mod;
    while (exp > 0) {
        if (exp & 1) result = (result * base) % mod;
        base = (base * base) % mod;
        exp >>= 1;
    }
    return result;
}

long long gcd(long long a, long long b){
    while (b != 0) {
        long long t = b;
        b = a % b;
        a = t;
    }
    return a;
}

long long modInverse(long long a, long long m){
    long long m0 = m, x0 = 0, x1 = 1;
    if (m == 1) return 0;
    while (a > 1) {
        long long q = a / m;
        long long t = m;
        m = a % m;
        a = t;
        t = x0;
        x0 = x1 - q * x0;
        x1 = t;
    }
    if (x1 < 0) x1 += m0;
    return x1;
}

int main(){
    long long p, q, n, phi, e, d;
    printf("Enter two prime numbers: ");
    scanf("%lld %lld", &p, &q);
    n = p * q;
    phi = (p - 1) * (q - 1);
    printf("Enter public exponent e (1 < e < %lld, coprime with %lld): ", phi, phi);
    scanf("%lld", &e);
    while (e <= 1 || e >= phi || gcd(e, phi) != 1) {
        if (e <= 1 || e >= phi) printf("e must satisfy 1 < e < %lld.\n", phi);
        else printf("gcd(%lld, %lld) = %lld, so e is not coprime with the totient.\n", e, phi, gcd(e, phi));
        printf("Invalid e, enter again: ");
        scanf("%lld", &e);
    }
    printf("e = %lld is coprime with totient %lld.\n", e, phi);
    d = modInverse(e, phi);
    printf("Public key  (e, n) = (%lld, %lld)\n", e, n);
    printf("Private key (d, n) = (%lld, %lld)\n", d, n);
    getchar();
    char msg[MAX];
    printf("Enter message: ");
    int len = 0;
    char ch;
    while (len < MAX - 1 && (ch = getchar()) != '\n' && ch != EOF) msg[len++] = ch;
    msg[len] = '\0';
    long long cipher[MAX];
    printf("Encrypted: ");
    for (int i = 0; i < len; i++) {
        cipher[i] = squareAndMultiply((unsigned char)msg[i], e, n);
        printf("%lld ", cipher[i]);
    }
    printf("\n");

    printf("Decrypted: ");
    for (int i = 0; i < len; i++) printf("%c", (char)squareAndMultiply(cipher[i], d, n));
    printf("\n");

    return 0;
}
```

---

## Code Walkthrough

| Function / Block | Purpose |
|---|---|
| `squareAndMultiply` | Computes `base^exp mod mod` in O(log exp) |
| `gcd` | Euclidean algorithm; validates that `e` is coprime with `φ(n)` |
| `modInverse` | Extended Euclidean; computes `d = e^−1 mod φ(n)` |
| Key generation block | Computes `n`, `φ(n)`, validates `e`, computes `d` |
| Input validation loop | Rejects `e ≤ 1`, `e ≥ φ(n)`, or `gcd(e, φ(n)) ≠ 1` |
| `getchar()` before message | Consumes the newline left by `scanf` |
| Encryption loop | Encrypts each byte of the message separately |
| Decryption loop | Raises each ciphertext value to `d` and casts back to `char` |

### Why each character is encrypted separately

```
The program treats the message as a sequence of bytes.
Each byte becomes one RSA block.

This is textbook RSA and is NOT how real systems work.
Real RSA encrypts one random symmetric key, then uses AES for the data.
```

---

## Sample Run

```
Enter two prime numbers: 61 53
Enter public exponent e (1 < e < 3120, coprime with 3120): 17
e = 17 is coprime with totient 3120.
Public key  (e, n) = (17, 3233)
Private key (d, n) = (2753, 3233)
Enter message: HI
Encrypted: 3000 1486
Decrypted: HI
```

```
Note: n = 3233 must be larger than every byte value being encrypted.
For plain ASCII (0–127) this works. For arbitrary bytes (0–255),
n must be greater than 255, and for real security n must be
hundreds of bits long.
```

---

## Security Notes and Limitations

### This implementation is educational only

| Issue | Why it matters |
|---|---|
| Textbook RSA (no padding) | Deterministic: same message → same ciphertext |
| Per-character encryption | Leaks message length and patterns |
| No OAEP | Vulnerable to chosen-ciphertext attacks |
| No primality testing | `p` and `q` are trusted blindly |
| 64-bit arithmetic | Intermediate products overflow for large `n` |
| No key-size guidance | Small `n` is trivially factorable |
| No constant-time code | Timing side channels are possible |
| No signature support | Only encryption is implemented |

### Deterministic encryption

```
Textbook RSA is deterministic:
  Encrypt("A") always gives the same ciphertext.

An attacker can build a dictionary of likely plaintexts and
compare ciphertexts. Real RSA uses randomised OAEP padding.
```

### Malleability

```
RSA is multiplicatively homomorphic:
  E(m1) × E(m2) = E(m1 × m2)

This is useful for some protocols but dangerous without padding:
an attacker can manipulate ciphertexts in predictable ways.
```

### Key sizes

| Security level | RSA modulus |
|---|---|
| Deprecated | 1024 bits |
| Minimum today | 2048 bits |
| Recommended | 3072 bits or more |
| Long-term / high assurance | 4096 bits |

```
The primes p and q should be roughly the same size,
random, and far apart. Real libraries generate them for you.
```

### Other real-world pitfalls

```
- Small e with no padding → Coppersmith / Hastad attacks
- Reusing the same n across users → common modulus attacks
- Leaking p and q → n can be factored instantly
- Timing / power analysis → side-channel key recovery
- Bleichenbacher attacks on PKCS#1 v1.5 padding
- Using a faulty random number generator for p, q, or padding
```

### Do not roll your own crypto

```
Use a vetted library:
  OpenSSL, libsodium, BoringSSL, mbedTLS, Go crypto/rsa, ...

Use high-level protocols:
  TLS, Signal, age, GPG, ...

The C program here is for understanding the math,
not for protecting real data.
```

---

## Hybrid Encryption (How RSA Is Actually Used)

RSA is slow and size-limited, so real systems combine it with symmetric encryption.

```
1. Generate a random AES key K.
2. Encrypt the message with AES-GCM using K.
3. Encrypt K with the recipient's RSA public key.
4. Send: RSA-encrypted K + AES ciphertext.

Recipient:
1. Decrypt K with the RSA private key.
2. Decrypt the message with K.
```

```
RSA protects the key; AES protects the data.
This is how TLS, S/MIME, and PGP work in practice.
```

---

## RSA vs ECC

| | RSA | ECC |
|---|---|---|
| Hard problem | Integer factorisation | Elliptic curve discrete log |
| Typical key size | 2048–4096 bits | 256–384 bits |
| Speed | Slower key generation | Faster, smaller keys |
| Bandwidth | Larger keys/signatures | Compact |
| Maturity | Very mature | Mature, widely deployed |
| Common use | Certificates, signatures, legacy | TLS, mobile, IoT, Bitcoin |

```
256-bit ECC ≈ 3072-bit RSA in security strength,
with much smaller keys and faster operations.
```

Next: [Elliptic Curve Cryptography](ellipticCurve.md).

---

## Revision Summary

| Concept | Value |
|---|---|
| Type | Asymmetric / public-key |
| Hard problem | Integer factorisation |
| Modulus | n = p × q |
| Totient | φ(n) = (p − 1)(q − 1) |
| Public exponent | e, with gcd(e, φ(n)) = 1 |
| Private exponent | d = e^−1 mod φ(n) |
| Encrypt | c = m^e mod n |
| Decrypt | m = c^d mod n |
| Common e | 65537 |
| Fast exponentiation | Square-and-multiply, O(log e) |
| Inverse | Extended Euclidean algorithm |
| Real-world padding | OAEP (encryption), PSS (signatures) |
| Real-world usage | Hybrid: RSA for key, AES for data |
| Minimum key size | 2048 bits today |

- RSA security = factoring `n` is hard
- `e` and `d` are modular inverses modulo `φ(n)`
- Square-and-multiply makes exponentiation fast
- Textbook RSA is deterministic and insecure
- Always use padding and a vetted library
- Real systems use RSA only to protect a symmetric key
- ECC offers equivalent security with much smaller keys
