# Routing Fundamentals

## What It Is

**Routing** is the process of choosing a path for a packet across one or more networks. Routers use routing information to decide which outgoing interface (and next hop) a packet should take.

```
Host A ── R1 ── R2 ── R3 ── Host B
          ↑      ↑      ↑
       each router independently decides: "where do I send this next?"
```

Routing works **hop by hop**. No single device knows the entire path — each router only knows the next hop toward the destination.

---

## Forwarding vs Routing

These two terms are often confused, but they happen at different speeds and in different planes.

| | Forwarding | Routing |
|---|---|---|
| What | Move a packet from input port to output port | Learn and choose the best paths |
| Scope | One packet at a time | Whole network, continuous |
| Speed | Very fast (hardware) | Slow (software/protocol) |
| Plane | Data plane | Control plane |
| Decision | Lookup destination in FIB | Build/update the routing table |
| Example | ASIC matches `10.1.1.0/24` → port 3 | OSPF computes shortest path tree |

```
Routing (control plane):  builds the map
Forwarding (data plane):  follows the map, one packet at a time
```

---

## Control Plane vs Data Plane

```
┌─────────────────────────────────────────┐
│  CONTROL PLANE                          │
│  - runs routing protocols (OSPF, BGP)   │
│  - builds RIB (routing table)           │
│  - CPU, software, milliseconds          │
├─────────────────────────────────────────┤
│  DATA PLANE                             │
│  - forwards packets                     │
│  - uses FIB (forwarding table)          │
│  - ASIC/TCAM, hardware, nanoseconds     │
└─────────────────────────────────────────┘
```

- **Control plane** = "brain" — decides paths.
- **Data plane** = "muscle" — moves packets.
- SDN moves the control plane to a central controller (see [24-sdn-and-control-plane](24-sdn-and-control-plane.md)).

---

## Router Architecture

```
        ┌──────────────────────────────────────────────┐
        │              ROUTING PROCESSOR               │
        │   routing protocol ↔ RIB → FIB (control)     │
        └──────────────────────────────────────────────┘
                 │                          │
   ┌─────────────▼─────────┐    ┌───────────▼──────────┐
   │      INPUT PORT       │    │      OUTPUT PORT     │
   │  - receive frame      │    │  - queue packets     │
   │  - decapsulate        │    │  - schedule          │
   │  - lookup FIB         │    │  - transmit          │
   └─────────────┬─────────┘    └───────────▲──────────┘
                 │                          │
                 └──── SWITCHING FABRIC ────┘
```

| Component | Role |
|---|---|
| Input port | Receive, check FCS, lookup destination |
| Switching fabric | Move packet from input to correct output |
| Output port | Queue and transmit |
| Routing processor | Runs protocols, builds RIB/FIB |

**Queues form when input rate > output rate** → delay, jitter, and eventually drops. Head-of-line blocking happens when a packet at the front of a queue blocks packets behind it that could go to a different output.

---

## Routing Table vs FIB

```
RIB (Routing Information Base)
  - built by routing protocols
  - may contain multiple routes per prefix
  - includes metrics, administrative distance, next hops
            │
            │  best route selection
            ▼
FIB (Forwarding Information Base)
  - optimised for lookup (often in TCAM)
  - one active next hop per prefix (or ECMP set)
  - used by the data plane
```

`show ip route` displays the RIB (on most vendors, the best routes).
The FIB is what the hardware actually uses to forward.

---

## How a Router Forwards a Packet

The router performs a **longest prefix match (LPM)** on the destination IP.

```
Routing table:
  10.0.0.0/8      → R1
  10.1.0.0/16     → R2
  10.1.1.0/24     → R3
  0.0.0.0/0       → ISP

Packet to 10.1.1.5:
  matches /8, /16, /24  → /24 wins (most specific) → R3

Packet to 10.2.3.4:
  matches /8 only → R1

Packet to 8.8.8.8:
  matches default only → ISP
```

```
Rule: MOST SPECIFIC PREFIX WINS — not the lowest metric.
```

Tie-breakers (Cisco order):
1. Longest prefix match
2. Lowest administrative distance
3. Lowest metric
4. Additional vendor-specific tie-breakers (e.g., ECMP installs multiple)

---

## Routing Metrics

A metric is the cost of a route. Different protocols use different metrics.

| Metric | Meaning | Used by |
|---|---|---|
| Hop count | Number of routers crossed | RIP |
| Bandwidth | Link capacity (higher = better) | OSPF (cost = ref/bw), EIGRP |
| Delay | Time to traverse link | EIGRP, OSPF (via cost) |
| Load | Current utilisation | EIGRP, IGRP |
| Reliability | Error rate | EIGRP, IGRP |
| Cost | Arbitrary admin value | OSPF, IS-IS |
| MTU | Maximum transmission unit | EIGRP |

```
RIP metric = hop count          → ignores link speed
OSPF metric = cost              → 10/100/1000 Mbps links can all look equal by default
EIGRP metric = bandwidth+delay  → more nuanced
BGP metric = policy (AS path etc.)
```

---

## Administrative Distance

When multiple protocols offer a route to the same prefix, the router picks the one with the **lowest administrative distance (AD)**. AD measures **trustworthiness**, not path quality.

| Route source | AD (Cisco) |
|---|---|
| Connected interface | 0 |
| Static route | 1 |
| eBGP | 20 |
| EIGRP (internal) | 90 |
| OSPF | 110 |
| IS-IS | 115 |
| RIP | 120 |
| EIGRP (external) | 170 |
| iBGP | 200 |
| Unknown | 255 (never used) |

```
Same prefix from OSPF (110) and RIP (120) → OSPF wins, even if RIP has fewer hops.
Static route (1) beats every dynamic protocol.
```

Note: AD is vendor terminology; other vendors call it *preference*.

---

## Static vs Dynamic Routing

### Static Routing

Routes are manually configured and do not change.

```
ip route 192.168.2.0 255.255.255.0 10.0.0.2

"To reach 192.168.2.0/24, send to 10.0.0.2"
```

| Pros | Cons |
|---|---|
| Simple, predictable | No automatic failover |
| No protocol overhead | Manual updates on every change |
| Secure (nothing advertised) | Does not scale |
| Good for stubs and defaults | Easy to misconfigure |

### Dynamic Routing

Routers exchange information and adapt automatically.

| Pros | Cons |
|---|---|
| Automatic convergence | Protocol overhead (CPU, bandwidth) |
| Scales to large networks | More complex to configure/troubleshoot |
| Reacts to failures | Can be a security surface |
| Policy and traffic engineering | Misconfiguration can cause outages |

```
Static is fine for: default routes, stub networks, small labs, edge links
Dynamic is needed for: multi-path networks, large enterprises, the Internet
```

---

## Autonomous Systems (AS)

An **Autonomous System** is a group of IP networks and routers under a single administrative control with a common routing policy.

```
AS 100                    AS 200
┌─────────────────┐      ┌─────────────────┐
│  R1 ── R2 ── R3 │──────│  R4 ── R5       │
└─────────────────┘      └─────────────────┘
        IGP inside AS            IGP inside AS
              └──── BGP between ASes ────┘
```

- Each AS gets an **AS number (ASN)**.
- 16-bit ASN: 0 – 65535 (public `1–64511`, private `64512–65534`, reserved `0`, `65535`).
- 32-bit ASN: up to 4,294,967,294 (RFC 6793) for exhaustion relief.
- The Internet is a network of ASes connected by BGP.

---

## IGP vs EGP

| | IGP (Interior Gateway Protocol) | EGP (Exterior Gateway Protocol) |
|---|---|---|
| Scope | Inside one AS | Between ASes |
| Goal | Find shortest/fastest path | Enforce policy between organisations |
| Metric | Cost, bandwidth, hops | AS path length, attributes, policy |
| Convergence | Fast (seconds) | Slower (tens of seconds to minutes) |
| Examples | RIP, OSPF, EIGRP, IS-IS | BGP |

```
IGP = "how do I get across my own network?"
EGP = "which neighbour should I trust to reach someone else's network?"
```

---

## Convergence

**Convergence** is the time it takes for every router to agree on the topology after a change.

```
Link fails
   ↓
Neighbour detects failure
   ↓
Routing updates propagate
   ↓
Every router recomputes its best path
   ↓
Traffic flows on the new path  ← CONVERGED
```

Until convergence:
- Packets may loop (TTL expires)
- Packets may be dropped (blackhole)
- Traffic may use a suboptimal path

```
Fast convergence needs:
  - quick failure detection (BFD, carrier loss)
  - fast update propagation (triggered updates)
  - fast computation (SPF tuning)
```

---

## Classes of Routing Protocols

```
Routing protocols
├── Distance Vector      → RIP, IGRP (advanced: EIGRP)
├── Link State           → OSPF, IS-IS
└── Path Vector          → BGP
```

| Class | Knows | Shares | Loops prevented by |
|---|---|---|---|
| Distance vector | Distance + next hop | Full table to neighbours | Split horizon, poison reverse, timers |
| Link state | Full topology | Link states flooded to all | SPF computation |
| Path vector | Full AS path | Paths with attributes | AS path loop check |

Details: [09-distance-vector-routing](09-distance-vector-routing.md), [10-link-state-routing](10-link-state-routing.md), [11-path-vector-routing](11-path-vector-routing.md).

---

## Quick Revision

| Concept | Key Point |
|---|---|
| Routing vs forwarding | Routing builds the map; forwarding follows it |
| Control plane | Protocols, RIB, CPU |
| Data plane | FIB, hardware, per-packet |
| Longest prefix match | Most specific prefix always wins |
| Metric | Cost of a path (hops, bandwidth, etc.) |
| Administrative distance | Trust between route sources; lower wins |
| Static route | Manual, no failover, predictable |
| Dynamic route | Learned, converges, scales |
| AS | Network under one administrative control |
| IGP | Routing inside an AS (OSPF, RIP, EIGRP, IS-IS) |
| EGP | Routing between ASes (BGP) |
| Convergence | All routers agree on topology after a change |

- Forwarding is fast and local; routing is slow and distributed
- Prefix length beats metric; AD beats metric across protocols
- IGPs optimise paths; BGP enforces business policy
- Convergence time = detection + propagation + computation

---

← [07-ipv4-addressing](07-ipv4-addressing.md) • [↑ Computer Networks](README.md) • [09-distance-vector-routing](09-distance-vector-routing.md) →
