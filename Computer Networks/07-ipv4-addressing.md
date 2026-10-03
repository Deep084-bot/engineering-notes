# IPv4 Addressing

## What It Is

Every device on the internet needs a unique address. An **IPv4 address** is a 32-bit number that identifies a device's interface on a network.

```
IPv4 address: 32 bits = 4 bytes = 4 octets
  Binary:  11000000 10101000 00000001 00000001
  Decimal: 192.168.1.1
```

---

## Two Representations

### Dotted Decimal

```
192.168.1.1

Each octet: 0–255 (8 bits)
Convert each octet from decimal to binary:
  192 = 11000000
  168 = 10101000
    1 = 00000001
    1 = 00000001

Full binary: 11000000.10101000.00000001.00000001
```

### Binary

```
11000000 10101000 00000001 00000001

Group into octets, convert each to decimal:
  11000000 = 128+64 = 192
  10101000 = 128+32+8 = 168
  00000001 = 1
  00000001 = 1

Dotted decimal: 192.168.1.1
```

---

## Two Parts of Every Address

```
IP Address = [Network Prefix] + [Host Identifier]

192.168.1.1
││││││││││││││││││││││││││││││││
├──── Network ────┤├── Host ──┤
  (which network?)   (which device on that network?)
```

The **subnet mask** tells where the network portion ends and host portion begins.

```
192.168.1.1 / 24
              ↑ 24 = first 24 bits are network, last 8 are host

Mask: 11111111.11111111.11111111.00000000 = 255.255.255.0
Network: 192.168.1.0
Host range: 192.168.1.1 – 192.168.1.254
Broadcast: 192.168.1.255
```

---

## Classful Addressing

The original system divided IP addresses into **5 classes** based on the leading bits.

| Class | First bits | First octet | Network/Host | Default Mask |
|---|---|---|---|---|
| **A** | 0 | 1–126 | N.H.H.H | 255.0.0.0 (/8) |
| **B** | 10 | 128–191 | N.N.H.H | 255.255.0.0 (/16) |
| **C** | 110 | 192–223 | N.N.N.H | 255.255.255.0 (/24) |
| **D** | 1110 | 224–239 | Multicast | — |
| **E** | 1111 | 240–255 | Reserved | — |

```
Class A: 0.H.H.H         → 126 networks, 16M hosts each
Class B: 10.N.N.H         → 16,384 networks, 65K hosts each
Class C: 110.N.N.N        → 2M networks, 254 hosts each
Class D: 1110.X.X.X       → multicast
Class E: 1111.X.X.X       → reserved/experimental
```

### Problem with Classful

```
A company needs 500 hosts:
  Class C (254 hosts) → too small, need 3 Class C blocks
  Class B (65K hosts) → way too big, wastes 65K addresses

Classful is INFLEXIBLE — forced to choose between
  too small and massively wasteful.
```

---

## Classless Addressing — CIDR

**CIDR (Classless Inter-Domain Routing)** replaces classful with flexible prefix lengths.

```
Instead of fixed /8, /16, /24:
  Use ANY prefix length: /13, /20, /27, etc.

192.168.1.0/24  → 254 hosts (like Class C)
10.0.0.0/8      → 16M hosts (like Class A)
172.16.0.0/12   → 1M hosts (between Class A and B)
```

**CIDR notation:** `a.b.c.d/n` where n = number of network bits.

```
192.168.1.0/24
              ↑ 24 bits for network, 8 bits for hosts
```

### CIDR Block Sizes

```
/24 → 2^(32-24) = 256 addresses  (254 usable)
/25 → 2^(32-25) = 128 addresses  (126 usable)
/26 → 2^(32-26) = 64 addresses   (62 usable)
/27 → 2^(32-27) = 32 addresses   (30 usable)
/28 → 2^(32-28) = 16 addresses   (14 usable)
/29 → 2^(32-29) = 8 addresses    (6 usable)
/30 → 2^(32-30) = 4 addresses    (2 usable, for point-to-point)

/20 → 2^(32-20) = 4096 addresses
/16 → 2^(32-16) = 65,536 addresses
/8  → 2^(32-8)  = 16,777,216 addresses
```

**Usable hosts = total − 2** (subtract network address and broadcast address).

### Advantages of CIDR over Classful

| Classful | CIDR |
|---|---|
| Fixed block sizes (/8, /16, /24) | Any prefix length |
| Lots of wasted addresses | Efficient allocation |
| No route aggregation | Route aggregation (supernetting) |
| 256 or 65K or 16M — no in-between | Any size block |
| Routing table entries explode | Fewer entries via aggregation |

**Route aggregation (supernetting):**
```
Instead of advertising 4 separate /24 routes:
  192.168.0.0/24
  192.168.1.0/24
  192.168.2.0/24
  192.168.3.0/24

Advertise ONE route:
  192.168.0.0/22   (covers all four)

/22 = first 22 bits fixed → 192.168.000000xx.xxxxxxxx
  → 192.168.0.0 – 192.168.3.255
```

---

## Subnetting

Subnetting divides a larger network into smaller **sub-networks**.

```
Starting: 192.168.1.0/24 (256 addresses)
Need: 4 subnets of ~60 hosts each

Borrow 2 bits from host portion:
  /24 + 2 = /26

/26 = 64 addresses per subnet (62 usable)

Subnets:
  192.168.1.0/26    → .1 – .62     (broadcast .63)
  192.168.1.64/26   → .65 – .126   (broadcast .127)
  192.168.1.128/26  → .129 – .190  (broadcast .191)
  192.168.1.192/26  → .193 – .254  (broadcast .255)
```

### Subnetting Formula

```
Number of subnets = 2^(bits borrowed)
Hosts per subnet  = 2^(remaining host bits) − 2

/24 with 2 bits borrowed:
  Subnets: 2² = 4
  Hosts per subnet: 2⁶ − 2 = 62
```

### Subnet Mask for Subnetting

```
/24 = 255.255.255.0
/26 = 255.255.255.192

/26 in binary: 11111111.11111111.11111111.11000000
                         ↑ 26 ones = network prefix
```

---

## Special Addresses

| Address | Meaning |
|---|---|
| `0.0.0.0` | This network (default route) |
| `127.x.x.x` | Loopback (localhost) |
| `255.255.255.255` | Limited broadcast |
| `x.x.x.0` | Network address (host bits all 0) |
| `x.x.x.255` | Broadcast address (host bits all 1) |
| `10.0.0.0/8` | Private (Class A) |
| `172.16.0.0/12` | Private (Class B) |
| `192.168.0.0/16` | Private (Class C) |

---

## Quick Revision

| Concept | Key Point |
|---|---|
| IPv4 | 32-bit address, dotted decimal (4 octets) |
| Network + Host | Prefix identifies network, suffix identifies host |
| Classful | A(/8), B(/16), C(/24) — rigid, wasteful |
| CIDR | Flexible prefix (/n), efficient, aggregation |
| Subnet mask | Tells where network bits end |
| Subnetting | Borrow host bits → smaller subnets |
| Usable hosts | 2^(host bits) − 2 (minus network & broadcast) |
| Route aggregation | Supernetting reduces routing table size |

- CIDR > Classful: flexible sizes, aggregation, less waste
- Subnet mask /n → 2^(32-n) addresses, 2^(32-n)−2 hosts
- Private ranges: 10.x, 172.16-31.x, 192.168.x

---

← [06-sliding-window](06-sliding-window.md) • [↑ Computer Networks](README.md)
