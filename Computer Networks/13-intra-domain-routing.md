# Intra-Domain Routing (IGP)

## What It Is

**Intra-domain routing** is routing **inside a single Autonomous System (AS)**. The protocols that do this are called **Interior Gateway Protocols (IGPs)**.

```
        AS 100 (one organisation)
   ┌──────────────────────────────┐
   │   R1 ── R2 ── R3             │
   │    │     │     │             │
   │   R4 ── R5 ── R6             │  ← IGPs run here
   └──────────┬───────────────────┘
              │ BGP (inter-domain)
        ┌─────┴─────┐
        │   AS 200  │
        └───────────┘
```

The IGP's job: give every router inside the AS a loop-free, converged path to every internal prefix — and to the border routers that lead outside.

---

## The IGP Family

| Protocol | Type | Algorithm | Metric | Standard |
|---|---|---|---|---|
| RIP | Distance vector | Bellman-Ford | Hop count (max 15) | RFC 2453 (v2), RFC 2080 (ng) |
| OSPF | Link state | Dijkstra (SPF) | Cost (bandwidth-based) | RFC 2328 (v2), RFC 5340 (v3) |
| EIGRP | Advanced distance vector | DUAL | Composite (bandwidth, delay, load, reliability) | Cisco (RFC 7868) |
| IS-IS | Link state | Dijkstra (SPF) | Cost (default 10 per link) | ISO 10589 / RFC 1195 |

```
RIP   → small, simple, legacy
OSPF  → the standard open enterprise IGP
EIGRP → Cisco-centric, fast convergence, easy design
IS-IS → service provider favourite, very scalable
```

---

## RIP — Routing Information Protocol

```
Type:    distance vector
Metric:  hop count (1–15, 16 = infinity)
Update:  full table every 30 s to neighbours
Transport: UDP port 520 (RIPng: UDP 521)
```

**Strengths:** trivial to configure, low resource use.
**Weaknesses:** slow convergence, 15-hop limit, ignores bandwidth, count-to-infinity.

**Use when:** small lab or legacy network, or as a teaching protocol.

Details: [14-rip](14-rip.md).

---

## OSPF — Open Shortest Path First

```
Type:    link state
Metric:  cost = reference bandwidth / interface bandwidth
         (default reference = 100 Mbps)
Update:  LSAs flooded within an area; triggered by change
Transport: IP protocol 89
```

**Strengths:** open standard, fast convergence, loop-free, area hierarchy, vendor-neutral.
**Weaknesses:** complex, CPU/memory heavy, area design matters, SPF churn on unstable links.

**Use when:** multi-vendor enterprise networks, large campuses, any network needing a solid open IGP.

Details: [15-ospf](15-ospf.md).

---

## EIGRP — Enhanced Interior Gateway Routing Protocol

```
Type:    advanced distance vector ("hybrid")
Metric:  composite — bandwidth, delay (default), optionally load, reliability
Update:  partial, bounded, triggered; neighbours only
Transport: IP protocol 88 (RTP)
```

Key ideas:
- **DUAL** (Diffusing Update Algorithm) computes loop-free paths without waiting for timers.
- **Successor** = best next hop; **feasible successor** = backup next hop already known to be loop-free.
- **Feasible condition:** a neighbour's reported distance to a destination must be less than the current feasible distance.
- **Unequal-cost load balancing** via `variance`.

**Strengths:** very fast convergence, low overhead, easy to design, supports unequal-cost multipath.
**Weaknesses:** historically Cisco-proprietary (now partially open), less common in pure multi-vendor environments.

---

## IS-IS — Intermediate System to Intermediate System

```
Type:    link state
Metric:  cost (default 10 per link, wide metrics supported)
Transport: directly over Layer 2 (not IP)
Structure: Level 1 (intra-area), Level 2 (backbone)
```

**Strengths:** highly scalable, stable, TLV-based extensibility, common in service providers.
**Weaknesses:** less familiar to enterprise engineers, configuration can be cryptic, no built-in IP-only assumptions (runs over CLNS).

---

## Head-to-Head Comparison

| Feature | RIP | OSPF | EIGRP | IS-IS |
|---|---|---|---|---|
| Algorithm | Bellman-Ford | Dijkstra | DUAL | Dijkstra |
| Type | DV | LS | Advanced DV | LS |
| Metric | Hops | Cost | Composite | Cost |
| Max diameter | 15 hops | No hard limit | No hard limit | No hard limit |
| Convergence | Slow | Fast | Very fast | Fast |
| Hierarchy | None | Areas | None (flat) | Levels |
| Transport | UDP 520 | IP 89 | IP 88 | L2 |
| Multicast | 224.0.0.9 (v2) | 224.0.0.5/6 | 224.0.0.10 | L2 multicast |
| Standard | Open | Open | Cisco (RFC 7868) | Open |
| Config complexity | Low | High | Medium | High |
| Scalability | Small | Large | Large | Very large |
| Typical use | Legacy/small | Enterprise | Cisco enterprise | Service provider |

---

## How an IGP Fits With BGP

```
        ┌──────────── AS 100 ────────────┐
        │                                │
        │  OSPF/IS-IS/EIGRP inside       │
        │                                │
        │  R1 ── R2 ── R3                │
        │         │                      │
        │        R4 (border)             │
        └─────────┬──────────────────────┘
                  │ eBGP
              ┌───┴───┐
              │ AS 200│
              └───────┘
```

- The **IGP** provides reachability to internal prefixes and to the **next hop** of BGP routes.
- **BGP** provides external prefixes and policy.
- If the IGP cannot reach a BGP next hop, the BGP route is unusable.

```
BGP route: 203.0.113.0/24 via 10.0.0.2
IGP must know how to reach 10.0.0.2 — otherwise the route is invalid.
```

This is why BGP and an IGP are almost always deployed together at an AS border.

---

## Choosing an IGP

```
Small network, simple, legacy gear        → RIP (rarely new deployments)
Multi-vendor enterprise, open standard    → OSPF
Cisco-heavy enterprise, fast convergence  → EIGRP
Service provider, very large scale        → IS-IS
```

Practical rules:

- **OSPF** is the safe default for new enterprise networks.
- **IS-IS** scales best and is common in ISPs and large data centres.
- **EIGRP** is excellent if the network is Cisco-only and design simplicity matters.
- **RIP** is mostly historical or for labs.

---

## Metrics in Practice

```
RIP:    every link = 1 hop. A 10 Mbps link and a 10 Gbps link cost the same.
OSPF:   cost = 100 Mbps / bandwidth.
          10 Mbps  → 10
          100 Mbps → 1
          1 Gbps   → 1   (all high-speed links look equal unless reference changes)
          Fix: set reference bandwidth to 10 Gbps or 100 Gbps, or set cost manually.
EIGRP:  metric = 256 × (10^7 / min bandwidth + cumulative delay)
          Bandwidth and delay are the defaults; load/reliability optional.
IS-IS:  default cost 10 per link; wide metrics allow fine-grained costs.
```

---

## Redistribution Between IGPs

When two IGPs meet (e.g., during migration or after a merger), routes must be **redistributed**.

```
OSPF domain ── R ── EIGRP domain

R redistributes:
  OSPF routes → EIGRP
  EIGRP routes → OSPF
```

Problems:
- Metrics are not comparable → must set a **seed metric**
- Loops can form if redistribution is mutual
- Route feedback can re-inject routes with worse attributes

Fixes: route tags, distribute lists, one-way redistribution, careful metric design.

Details: [20-routing-policy-and-redistribution](20-routing-policy-and-redistribution.md).

---

## Quick Revision

| Protocol | Type | Metric | Convergence | Best For |
|---|---|---|---|---|
| RIP | DV | Hops (max 15) | Slow | Small/legacy |
| OSPF | LS | Cost | Fast | Multi-vendor enterprise |
| EIGRP | Advanced DV | Composite | Very fast | Cisco enterprise |
| IS-IS | LS | Cost | Fast | Service providers |

- IGP = routing inside an AS; EGP = between ASes
- OSPF is the open enterprise default
- EIGRP's DUAL gives fast loop-free convergence
- IS-IS is the scalability king
- RIP is simple but limited to 15 hops and slow convergence
- The IGP must always reach BGP next hops

---

← [12-routing-algorithms-comparison](12-routing-algorithms-comparison.md) • [↑ Computer Networks](README.md) • [14-rip](14-rip.md) →
