# Path Vector Routing

## What It Is

**Path vector** routing is a distance-vector variant where each route advertisement carries the **entire path** (usually a list of Autonomous Systems) instead of just a distance and next hop.

```
Distance vector:   "I can reach 203.0.113.0/24 in 3 hops"
Path vector:       "I can reach 203.0.113.0/24 via AS 65001 → AS 65002 → AS 65003"
```

BGP — the routing protocol of the Internet — is a path vector protocol.

---

## Why Path Vector Exists

Distance vector had two problems at Internet scale:

1. **Count-to-infinity** — slow convergence and loops
2. **No policy** — a metric alone can't express business relationships

Path vector solves both:

```
Loop detection:
  If a router sees its own AS number in the AS path → reject the route

Policy:
  Each AS can decide which paths to accept/advertise based on
  AS path, origin, communities, and local preference
```

---

## The AS Path

```
AS 100 ──── AS 200 ──── AS 300 ──── 203.0.113.0/24

AS 100 receives from AS 200:
  prefix: 203.0.113.0/24
  AS path: 200 300
  next hop: 200's border router

AS 100 adds itself when advertising further:
  AS path: 100 200 300
```

Rules:

- Each eBGP speaker **prepends its own AS number** when advertising to another AS.
- The receiver prepends before passing it on.
- If a router sees its own AS in the path → **loop detected, discard**.
- AS path length is a **tie-breaker** in route selection, not the primary goal.

---

## Path Attributes

Unlike DV (which advertises a single number), path vector carries **attributes** alongside the prefix.

| Attribute | Meaning |
|---|---|
| `AS_PATH` | ASes the route has traversed |
| `NEXT_HOP` | IP to send traffic to |
| `ORIGIN` | How the route was originated (IGP, EGP, incomplete) |
| `LOCAL_PREF` | Local preference inside an AS (higher = preferred) |
| `MED` | Multi-exit discriminator — hint to neighbouring AS |
| `COMMUNITIES` | Tags for policy (e.g., no-export, customer) |

These attributes let operators express policy:

```
"Prefer this path because it is a customer link"        → LOCAL_PREF
"Do not advertise this prefix to peers"                 → COMMUNITY
"Do not send traffic to me for this prefix"             → MED
```

Details: [19-bgp](19-bgp.md).

---

## How Path Vector Works

```
1. Establish a session with a neighbour (usually over TCP).
2. Exchange full routing tables once.
3. Send incremental updates as routes change.
4. For each received route:
     - Check AS path for loops.
     - Apply import policy (filtering, attributes).
     - Run best-path selection.
     - Install the best route.
5. Advertise selected routes to neighbours after applying export policy.
```

```
        ┌──────────────┐
        │  Inbound     │  filter, set LOCAL_PREF
        │  policy      │
        └──────┬───────┘
               ▼
        ┌──────────────┐
        │  Best path   │  compare attributes
        │  selection   │
        └──────┬───────┘
               ▼
        ┌──────────────┐
        │  Outbound    │  filter, prepend AS, set MED
        │  policy      │
        └──────────────┘
```

---

## Path Vector vs Distance Vector

| | Distance Vector | Path Vector |
|---|---|---|
| Advertises | Distance + next hop | Full path (AS list) + attributes |
| Loop prevention | Timers, split horizon, infinity | AS path check |
| Policy support | Very limited | Rich (attributes) |
| Convergence | Slow | Slow (but stable and controlled) |
| Metric | Numeric cost | AS path length + attribute ranking |
| Scale | Small networks | Global Internet |
| Example | RIP | BGP |

```
Path vector = distance vector + full path + attributes + policy
```

---

## Path Vector vs Link State

| | Link State | Path Vector |
|---|---|---|
| Scope | Inside one AS (IGP) | Between ASes (EGP) |
| Knowledge | Full topology | Paths advertised by neighbours |
| Algorithm | Dijkstra / SPF | Best-path selection over attributes |
| Objective | Shortest path | Policy satisfaction |
| Convergence | Fast | Slower, deliberately damped |
| Example | OSPF, IS-IS | BGP |

An IGP asks **"what is the shortest path?"**
BGP asks **"which path best matches my business policy?"**

---

## Why BGP Is Not a Pure Shortest-Path Protocol

The Internet is not run by a single owner, so "shortest" is not the objective.

```
AS 100 can reach a prefix via:
  Path A: AS 200 → AS 300       (2 AS hops, customer link)
  Path B: AS 400 → AS 500 → AS 600 (3 AS hops, transit paid link)

Shortest = Path A.
But policy may prefer Path B if Path A is a peer that should not carry transit,
or if Path A is a paid link and Path B is cheaper.
```

BGP decisions are driven by:

1. Business relationships (customer > peer > provider)
2. Local preference
3. AS path length
4. Origin type
5. MED
6. eBGP vs iBGP
7. IGP cost to next hop

```
IGP: "get the packet there fast"
BGP: "get the packet there according to contract and policy"
```

---

## Where Path Vector Is Used

| Protocol | Use |
|---|---|
| BGP-4 | Inter-domain routing on the Internet |
| MP-BGP | IPv6, VPNv4, EVPN, multicast |
| BGP in data centres | Route reflection, EVPN control plane |

---

## Quick Revision

| Concept | Key Point |
|---|---|
| Path vector | Route carries the full AS path |
| AS_PATH | List of ASes traversed; loop detection |
| Loop prevention | Reject route if own AS appears in path |
| Attributes | AS_PATH, NEXT_HOP, LOCAL_PREF, MED, communities |
| Policy | Import/export filtering and attribute setting |
| Algorithm | Best-path selection, not shortest path |
| Scale | Designed for the global Internet |
| Protocol | BGP-4 |

- Path vector = DV + full path + attributes
- AS path gives loop-free routing without timers
- Policy beats shortest path between organisations
- BGP is the only EGP in practical use
- IGP optimises; BGP negotiates

---

← [10-link-state-routing](10-link-state-routing.md) • [↑ Computer Networks](README.md) • [12-routing-algorithms-comparison](12-routing-algorithms-comparison.md) →
