# Inter-Domain Routing

## What It Is

**Inter-domain routing** is routing **between Autonomous Systems (ASes)** — independently administered networks such as ISPs, enterprises, cloud providers, and content networks.

```
        AS 100                 AS 200                 AS 300
   ┌───────────────┐      ┌───────────────┐      ┌───────────────┐
   │  enterprise   │──────│     ISP       │──────│   content     │
   └───────────────┘      └───────────────┘      └───────────────┘
            │                     │                     │
            └──────────── BGP (inter-domain) ───────────┘
```

The protocol is **BGP**. It is the only EGP in practical use and the glue that holds the Internet together.

---

## Why an IGP Cannot Do This

| Problem | Why IGPs fail at Internet scale |
|---|---|
| Scale | Millions of prefixes; SPF/LSDB flooding would be enormous |
| Trust | No single authority owns the Internet |
| Policy | ISPs must enforce contracts, not just shortest paths |
| Instability | A flapping link in one ISP must not destabilise the world |
| Autonomy | Each AS must control its own routing decisions |

```
OSPF says: "Here is the topology, compute the shortest path."
BGP says:  "Here are the paths I am willing to use, subject to policy."
```

---

## Autonomous Systems

An **AS** is a set of networks and routers under one administrative control with a consistent routing policy.

```
AS 65001
  - one or more IGPs inside (OSPF, IS-IS, EIGRP)
  - one or more border routers running BGP
  - one or more ASNs
```

### AS numbers

| Range | Use |
|---|---|
| 1 – 64511 | Public 16-bit ASNs |
| 64512 – 65534 | Private 16-bit ASNs |
| 65535 | Reserved |
| 4200000000 – 4294967294 | Private 32-bit ASNs |

- 16-bit ASNs: 65,535 total (RFC 4271 era)
- 32-bit ASNs: RFC 6793, to handle exhaustion
- Private ASNs are stripped before advertising to the Internet.

---

## Business Relationships

Inter-domain routing is driven by **money and contracts**, not distance.

### 1. Customer–Provider (transit)

```
Customer AS 100 ────── Provider AS 200
```

- Customer **pays** the provider for Internet access.
- Provider advertises the customer's prefixes to the rest of the world.
- Provider advertises the whole Internet to the customer.
- Provider's routes to customer are typically preferred (via LOCAL_PREF).

### 2. Peer-to-Peer (settlement-free)

```
AS 100 ────── AS 200
```

- Two networks exchange **their own** traffic and their customers' traffic.
- No money changes hands (usually).
- Typically only routes for each other's customers are exchanged — not transit.
- Peers usually do **not** provide transit to third parties.

### 3. Sibling

```
AS 100 ────── AS 200   (same owner)
```

- Same organisation, different ASNs.
- Usually full trust and all routes exchanged.

```
The hierarchy of preference:
  Customer routes  >  Peer routes  >  Provider routes

Why? Customer routes make money; peer routes are free;
provider routes cost money.
```

---

## Valley-Free Routing

The commercial relationship creates an implicit rule: **do not carry transit traffic for free.**

```
A route that goes:
  provider → peer → provider   ← "valley" — usually invalid

Allowed patterns:
  customer → provider → ...
  peer → peer
  ... → customer
```

This is why BGP uses **policy**, not hop count.

```
Customer  →  Provider  : announce everything (all customers, peers, providers)
Provider  →  Customer  : announce only customer and internal routes
Peer      →  Peer      : announce only your customers and internal routes
```

---

## eBGP vs iBGP

BGP runs in two modes:

| | eBGP | iBGP |
|---|---|---|
| Between | Different ASes | Routers in the same AS |
| AS_PATH | Prepend own AS | Unchanged |
| TTL | 1 by default (directly connected) | Can be multi-hop |
| Next hop | Changes to eBGP peer | Unchanged by default |
| Loop prevention | AS_PATH | Split-horizon rule |
| Admin distance | 20 | 200 |
| Full mesh needed | No | Yes (unless route reflectors) |

```
        AS 100                         AS 200
   R1 ──iBGP── R2 ──eBGP── R3 ──iBGP── R4
```

### eBGP behaviour

```
- Runs between border routers of different ASes
- Sends its own AS number at the front of AS_PATH
- Default TTL = 1 → peers must be directly connected
  (use eBGP multihop for loopback peering)
- Next hop is rewritten to the eBGP peer's IP
```

### iBGP behaviour

```
- Runs between routers inside the same AS
- Does NOT prepend its own AS number
- Next hop is NOT changed by default
  → the IGP must provide reachability to the external next hop
- iBGP split horizon:
    a route learned from one iBGP peer is NOT advertised to another iBGP peer
```

### The iBGP full-mesh problem

Because of the split-horizon rule, every iBGP router must peer with every other iBGP router:

```
With N routers → N(N−1)/2 sessions

5 routers   → 10 sessions
50 routers  → 1,225 sessions
500 routers → 124,750 sessions   ← does not scale
```

Solutions:
- **Route reflectors (RR)** — a hub reflects routes to clients
- **Confederations** — split the AS into sub-ASes

Details: [19-bgp](19-bgp.md).

---

## Internet Exchange Points (IXPs)

An **IXP** is a shared fabric where many networks meet to exchange traffic directly.

```
                ┌──────── IXP fabric ────────┐
                │                            │
   AS 100 ──────┤  ┌──────┐   ┌──────┐       ├────── AS 300
                │  │switch│───│switch│       │
   AS 200 ──────┤  └──────┘   └──────┘       ├────── AS 400
                │                            │
                └────────────────────────────┘
```

Benefits:
- Lower latency (direct path, fewer hops)
- Lower cost (no transit fees for peer traffic)
- More path diversity
- Better resilience

**Route server** at an IXP: one BGP session to the route server can provide many peering sessions (the server re-advertises routes according to policy).

---

## ISP Tiers

```
Tier 1: global backbone, peers settlement-free with all other Tier 1s,
        sells transit to everyone else
Tier 2: regional/national, buys transit from Tier 1, peers with others
Tier 3: local/access ISPs, buy transit from Tier 2/1
Content providers: often peer at IXPs to reach users directly
```

```
Tier 1 does not pay for transit.
Tier 2 pays Tier 1 but peers to reduce cost.
Tier 3 mostly pays for transit.
```

---

## Multihoming and Traffic Engineering

A network with multiple upstreams uses BGP to control inbound and outbound traffic.

### Outbound traffic (which path do I use?)

Controlled by **LOCAL_PREF** (higher wins) and **AS_PATH length**.

```
AS 100 has two providers:
  ISP A: better latency, more expensive
  ISP B: cheaper, higher latency

Set LOCAL_PREF 200 for routes via ISP A → outbound prefers A.
```

### Inbound traffic (which path do others use to reach me?)

Controlled by **AS_PATH prepending** and **MED**.

```
Want traffic to enter via ISP A:
  Advertise the prefix to ISP B with AS_PATH prepended multiple times
  → ISP B's path looks longer → the Internet prefers ISP A
```

```
MED:
  Tell a neighbouring AS which of several links to prefer
  for traffic entering your AS.
  Lower MED = more preferred (usually).
```

```
LOCAL_PREF controls what YOU choose.
MED and AS_PATH prepending influence what OTHERS choose.
```

---

## BGP Route Selection (Preview)

BGP picks the best path using a ranked list of attributes, not a single metric.

```
1. Highest WEIGHT (Cisco, local only)
2. Highest LOCAL_PREF
3. Locally originated route
4. Shortest AS_PATH
5. Lowest ORIGIN (IGP < EGP < incomplete)
6. Lowest MED
7. eBGP over iBGP
8. Lowest IGP metric to next hop
9. Oldest route / lowest router ID / lowest neighbour IP
```

Details: [19-bgp](19-bgp.md).

---

## Common Inter-Domain Designs

### Single-homed

```
AS 100 ──── one provider
```

Simple default route; no BGP strictly required (static default works).

### Dual-homed (same provider)

```
AS 100 ──┬── provider link 1
         └── provider link 2
```

BGP for failover and load sharing; one provider means limited policy.

### Multi-homed (multiple providers)

```
AS 100 ──┬── ISP A
         ├── ISP B
         └── ISP C
```

Full BGP table or partial routes + default; needs policy for inbound/outbound.

### Internet exchange peering

```
AS 100 ──── IXP ──── many peers
        └── transit provider (for the rest of the Internet)
```

Peering reduces transit cost; transit covers the rest.

---

## Quick Revision

| Concept | Key Point |
|---|---|
| Inter-domain routing | Routing between ASes using BGP |
| AS | Network under one administrative control |
| ASN | 16-bit or 32-bit identifier |
| Customer–provider | Paid transit; provider carries your routes |
| Peering | Settlement-free exchange of own/customer routes |
| Valley-free | Don't carry transit for free |
| eBGP | Between ASes; AS_PATH prepend; TTL 1 |
| iBGP | Within an AS; no prepend; full mesh or RR |
| iBGP split horizon | Don't re-advertise iBGP routes to iBGP peers |
| IXP | Shared peering fabric |
| Route server | One session, many peering relationships |
| Multihoming | Multiple upstreams for resilience/policy |
| LOCAL_PREF | Controls outbound path choice |
| MED / prepending | Influences inbound path choice |

- Inter-domain routing is policy-driven, not shortest-path-driven
- Customer > peer > provider is the standard preference
- eBGP carries routes between ASes; iBGP distributes them inside
- iBGP needs full mesh, route reflectors, or confederations
- IXPs and peering are how the Internet avoids unnecessary transit

---

← [15-ospf](15-ospf.md) • [↑ Computer Networks](README.md) • [19-bgp](19-bgp.md) →
