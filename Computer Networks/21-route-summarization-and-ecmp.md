# Route Summarization and ECMP

## Route Summarization

**Summarization (aggregation)** replaces many specific prefixes with one covering prefix. It is the main reason routing tables can stay manageable.

```
Without summary:                With summary:
  10.1.0.0/24                     10.1.0.0/22
  10.1.1.0/24         ───►        (covers all four)
  10.1.2.0/24
  10.1.3.0/24
```

Benefits:

```
- Smaller routing tables
- Less CPU/memory for lookups
- Fewer routing updates (one prefix instead of many)
- Hides instability: a flapping /24 does not flap the /22
- Cleaner boundaries between areas/ASes
```

---

## How Summarization Works

Summarization is just **CIDR in reverse**. Find the common leading bits.

```
10.1.0.0/24    third octet: 00000000
10.1.1.0/24    third octet: 00000001
10.1.2.0/24    third octet: 00000010
10.1.3.0/24    third octet: 00000011
                            ^^^^^^
                            first 6 bits are common (000000)
                            last 2 bits vary

Common prefix: 10.1.0.0/22
  /22 = 8 + 8 + 6 = 22 bits
  Covers 10.1.0.0 – 10.1.3.255
```

```
Rule: the summary must be a contiguous block whose size is a power of 2,
aligned on its own boundary.
```

### Valid summary

```
10.1.0.0/24 + 10.1.1.0/24         → 10.1.0.0/23
10.1.0.0/24 ... 10.1.3.0/24       → 10.1.0.0/22
10.1.0.0/24 ... 10.1.7.0/24       → 10.1.0.0/21
```

### Invalid / lossy summary

```
10.1.0.0/24 + 10.1.2.0/24   → cannot be one /23
                              (missing 10.1.1.0/24)
Would need 10.1.0.0/22, which also covers 10.1.1.0/24
```

```
Summarizing a non-contiguous set either:
  - includes addresses you do not own (may blackhole), or
  - requires more than one summary.
```

---

## Longest Prefix Match Still Wins

A summary does not override more specific routes.

```
Routing table:
  10.1.0.0/22    → R1   (summary)
  10.1.2.0/24    → R2   (more specific)

Packet to 10.1.2.5  → /24 wins → R2
Packet to 10.1.3.5  → /22 wins → R1
```

This is how summarization coexists with exceptions and traffic engineering.

---

## Summarization per Protocol

### OSPF

```
Area range (on an ABR):
  area 1 range 10.1.0.0 255.255.252.0

External summary (on an ASBR):
  summary-address 10.2.0.0 255.255.0.0
```

### EIGRP

```
interface GigabitEthernet0/0
 ip summary-address eigrp 10 10.1.0.0 255.255.252.0
```

### BGP

```
aggregate-address 10.1.0.0 255.255.252.0 summary-only
```

```
summary-only → suppress the more specific routes
(without it, both the aggregate and the specifics are advertised)
```

### RIP

```
RIPv2 auto-summarizes at classful boundaries by default.
Disable with `no auto-summary` when using CIDR/VLSM.
Manual summary is interface-based on some platforms.
```

---

## The Null0 / Blackhole Route

When a router advertises a summary but does not have a route to every address inside it, it may install the summary pointing to **Null0** (a discard interface).

```
Summary: 10.1.0.0/22 → Null0
Specific: 10.1.0.0/24, 10.1.1.0/24 → real next hops

Traffic to 10.1.2.5 (not actually allocated):
  matches /22 → Null0 → dropped
  instead of looping back out toward the source
```

```
Null0 prevents routing loops caused by summaries that cover
unallocated space. The router "owns" the summary and drops
anything it cannot route more specifically.
```

---

## Summarization Caveats

| Caveat | Effect |
|---|---|
| Non-contiguous prefixes | Cannot summarize cleanly |
| Summary covers unallocated space | Blackholes (unless Null0 is used correctly) |
| Summary hides reachability | Traffic may be sent to a router that cannot deliver it |
| Suboptimal routing | Traffic may take a longer path to a summary |
| Asymmetric routing | Return path may differ from forward path |
| Over-summarization | Harder to troubleshoot |
| Summaries at wrong boundary | Blackholes and loops |

```
Summarize at the edge of a well-designed hierarchy,
not in the middle of a flat network.
```

---

## Hierarchical Routing Revisited

Summarization only works well when the addressing plan is hierarchical.

```
        ┌──────────── Area 0 ────────────┐
        │                                │
   ┌────┴────┐                     ┌─────┴─────┐
   │ Area 1  │                     │ Area 2    │
   │10.1.0.0 │                     │10.2.0.0   │
   │  /16    │                     │  /16      │
   └─────────┘                     └───────────┘

Area 0 sees:
  10.1.0.0/16 via Area 1 ABR
  10.2.0.0/16 via Area 2 ABR
```

```
Good addressing plan:
  Area 1 → 10.1.0.0/16
  Area 2 → 10.2.0.0/16
  → clean summaries at every boundary

Bad addressing plan:
  Area 1 → 10.1.0.0/24, 10.5.0.0/24, 10.9.0.0/24
  Area 2 → 10.2.0.0/24, 10.6.0.0/24, 10.10.0.0/24
  → no clean summary possible
```

```
Rule of thumb: allocate addresses by topology, not chronologically.
```

---

## ECMP — Equal-Cost Multi-Path

**ECMP** lets a router install and use **multiple paths with the same cost** simultaneously.

```
             ┌── R2 ──┐
   R1 ──────┤         ├────── R4
             └── R3 ──┘

If R1→R2→R4 and R1→R3→R4 have equal cost:
  ECMP installs both next hops and shares traffic.
```

```
Without ECMP: one path is chosen, the other sits idle.
With ECMP:    both paths carry traffic → more bandwidth, better resilience.
```

### Configuration

```
router ospf 1
 maximum-paths 4

router eigrp 10
 maximum-paths 4

router bgp 65001
 maximum-paths 4
```

```
Platforms differ: some support 8, 16, or more paths.
BGP ECMP usually requires identical attributes (or multipath relax).
```

---

## How ECMP Distributes Traffic

ECMP is almost always **per-flow**, not per-packet.

```
Hash inputs (usually):
  source IP
  destination IP
  source port
  destination port
  protocol
  (sometimes ingress interface, flow label)

hash(flow) → one of N next hops
```

```
Per-flow hashing:
  - keeps a flow on one path → packets stay in order
  - different flows spread across paths

Per-packet hashing:
  - better load distribution
  - causes reordering → bad for TCP
  - rarely used
```

```
Hash polarization:
  If many flows share the same hash inputs (e.g., same src/dst),
  they all land on one path → ECMP does not balance.
  Fix: include more entropy (ports, flow label, ingress port).
```

---

## ECMP Issues

| Issue | Why |
|---|---|
| Asymmetric routing | Forward and return paths differ |
| Stateful firewalls | Replies may arrive on the wrong path |
| MTU mismatch | One path may have a smaller MTU |
| Flow stickiness | A single large flow cannot use multiple paths |
| Hash polarization | Poor entropy → uneven load |
| Debugging difficulty | `traceroute` shows only one path |
| Out-of-order packets | If per-packet or path cost changes mid-flow |

```
ECMP is great for many flows, not for one giant flow.
For single-flow throughput, use link aggregation (LAG) or MPTCP.
```

---

## Unequal-Cost Multi-Path

Some protocols support unequal-cost load balancing.

### EIGRP variance

```
router eigrp 10
 variance 2
```

```
Variance = multiplier for the feasible distance.
Paths with metric ≤ (best metric × variance) can be installed,
provided they pass the feasibility condition.
```

### BGP multipath

```
router bgp 65001
 address-family ipv4
  maximum-paths 4
  bgp bestpath as-path multipath-relax
```

```
BGP normally requires identical AS_PATH length (and other attributes)
for multipath. `multipath-relax` allows different AS paths.
```

---

## Anycast

**Anycast** advertises the **same IP prefix from multiple locations**. Routing delivers each user to the topologically nearest instance.

```
             ┌── Site A: 203.0.113.0/24
   User ─────┤
             └── Site B: 203.0.113.0/24

BGP picks the nearest site for each user.
```

Used by:

```
- DNS root servers
- CDNs
- Public resolvers (8.8.8.8, 1.1.1.1)
- DDoS scrubbing centres
```

```
Anycast is not a routing protocol — it is a deployment technique
that relies on normal routing (usually BGP) to pick the nearest copy.
```

---

## Quick Revision

| Concept | Key Point |
|---|---|
| Summarization | Replace many prefixes with one covering prefix |
| CIDR reverse | Find common leading bits |
| Alignment | Summary must be contiguous, power-of-2, aligned |
| Longest prefix match | More specific routes still win |
| Null0 | Drops traffic covered by a summary but not specifically routed |
| OSPF | `area range`, `summary-address` |
| EIGRP | `ip summary-address` |
| BGP | `aggregate-address ... summary-only` |
| RIPv2 | Auto-summary by default; disable for CIDR |
| Hierarchical addressing | Makes summarization possible |
| ECMP | Install multiple equal-cost paths |
| ECMP hashing | Per-flow, usually 5-tuple |
| Hash polarization | Poor entropy causes uneven load |
| EIGRP variance | Unequal-cost multipath |
| BGP multipath | Requires matching attributes, or `multipath-relax` |
| Anycast | Same prefix from many sites; nearest wins |

- Summarize only contiguous, aligned blocks
- Longest prefix match preserves exceptions
- Null0 prevents summary-induced loops
- Good address planning is a prerequisite for summarization
- ECMP adds bandwidth and resilience but complicates stateful paths
- Per-flow hashing keeps TCP in order; per-packet does not
- Anycast turns routing into a geographic load balancer

---

← [20-routing-policy-and-redistribution](20-routing-policy-and-redistribution.md) • [↑ Computer Networks](README.md) • [22-multicast-routing](22-multicast-routing.md) →
