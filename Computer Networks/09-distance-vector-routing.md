# Distance Vector Routing

## What It Is

**Distance vector (DV)** is a routing algorithm where each router knows only:

1. The **distance** (cost) to every destination
2. The **next hop** (direction/vector) to reach it

Routers do not know the full topology. They learn by exchanging their entire routing table with **direct neighbours** — often called **"routing by rumour."**

```
A ──── B ──── C ──── D

B tells A: "I can reach D in 2 hops"
A adds its cost to B: "I can reach D in 3 hops via B"
```

---

## The Bellman-Ford Equation

Distance vector is a distributed implementation of the **Bellman-Ford** shortest-path algorithm.

```
D(x, y) = min over neighbours v { cost(x, v) + D(v, y) }

D(x, y) = least cost from x to y
cost(x,v) = cost of link from x to neighbour v
D(v, y) = neighbour v's cost to y
```

```
To reach destination D:
  Look at every neighbour's advertised distance to D
  Add my cost to that neighbour
  Pick the smallest total

        via B: 1 + 2 = 3  ← best
A ──1── B
        via C: 4 + 3 = 7

A installs: D via B, total cost 3
```

Each router repeats this for every destination, using only neighbour advertisements.

---

## How It Works

```
Step 1: Initialise
  - Connected networks: distance = link cost
  - Everything else: infinity
  - Send full table to direct neighbours

Step 2: Receive
  - For each destination in a neighbour's update:
      candidate = my cost to that neighbour + neighbour's distance
      if candidate < current distance:
          update distance and next hop

Step 3: Advertise
  - Send the updated table to neighbours

Step 4: Repeat
  - On a timer (periodic) or on change (triggered)
```

---

## Worked Example

Topology:

```
A ──1── B ──1── C ──1── D
```

Initial state (each router knows only its own links):

| Router | To A | To B | To C | To D |
|---|---|---|---|---|
| A | 0 (local) | 1 | ∞ | ∞ |
| B | 1 | 0 (local) | 1 | ∞ |
| C | ∞ | 1 | 0 (local) | 1 |
| D | ∞ | ∞ | 1 | 0 (local) |

After exchanging with neighbours:

| Router | To A | To B | To C | To D |
|---|---|---|---|---|
| A | 0 | 1 via B | 2 via B | 3 via B |
| B | 1 via A | 0 | 1 via C | 2 via C |
| C | 2 via B | 1 via B | 0 | 1 via D |
| D | 3 via C | 2 via C | 1 via C | 0 |

```
A learns D at cost 3 via B:
  B advertised D at cost 2
  A's cost to B = 1
  total = 1 + 2 = 3  → install via B
```

Convergence is reached when no router's table changes after an update.

---

## Count-to-Infinity Problem

DV's biggest weakness. When a link fails, **bad news travels slowly** — routers can bounce increasing costs off each other until they reach "infinity."

### Example

```
A ──1── B ──1── C ──1── D
A is directly attached to network X (cost 1)
```

Before failure:

| Router | Route to X |
|---|---|
| A | X, cost 1, direct |
| B | X, cost 2, via A |
| C | X, cost 3, via B |
| D | X, cost 4, via C |

Now **A's link to X fails**:

```
1. A removes X (or sets it to infinity).
2. B still has "X via A, cost 2" and advertises it to A.
3. A hears "X at cost 2 via B" → installs X via B at cost 3.
4. A advertises X at cost 3 to B.
5. B updates: X via A at cost 4.
6. B advertises 4 → A updates to 5 → ...
   A → B → A → B → ... until cost reaches infinity.
```

```
A ──1── B ──1── C ──1── D
✗ X        (B still points at A, A points at B → loop)
```

During this time, packets to X loop between A and B until TTL expires.

---

## Solutions to Count-to-Infinity

### 1. Split Horizon

**Do not advertise a route back out the interface it was learned from.**

```
B learned "X via A" from interface to A.
→ B never advertises "X via A" back to A.

So when A's link fails, A does not hear a bogus route from B.
```

```
         X
         │
         A
         │
         B ──── C

B tells C about X (fine).
B does NOT tell A about X, because B learned X from A.
```

Limitation: fails for loops involving more than two routers.

### 2. Poison Reverse

Advertise the route back to the source, but with **infinite metric** ("don't use me for this").

```
B advertises to A: "X = infinity"
This explicitly tells A: "do not route X through me."
```

Poison reverse is stronger than split horizon (it breaks two-node loops explicitly) but increases update size.

### 3. Route Poisoning

When a route fails, advertise it with an infinite metric instead of removing it. Neighbours immediately mark it unreachable rather than waiting for a timeout.

```
A's link to X fails:
  A advertises "X = 16" (RIP infinity)
  Neighbours immediately purge their routes via A
```

### 4. Hold-Down Timers

After a route is declared unreachable, **ignore any update for that route for a fixed period** (unless the new route has a better metric). This gives the network time to converge before accepting a possibly stale route.

```
Route fails → start hold-down (e.g., 180 s)
During hold-down: accept only a route with a better metric
After hold-down: accept normal updates
```

### 5. Triggered Updates

Send an update **immediately** when a route changes, instead of waiting for the periodic timer.

```
Without triggered updates: bad news waits up to 30 s (RIP)
With triggered updates: failure is announced right away
```

Often combined with a short random delay to avoid update storms.

### 6. Define Infinity

Use a small finite value as "infinity" so the count stops quickly. RIP uses **16** (max valid hop count is 15).

```
15 hops = reachable
16 hops = unreachable/infinity
```

This bounds the count-to-infinity loop to at most 15 iterations, but also limits network diameter.

---

## Pros and Cons

| Pros | Cons |
|---|---|
| Simple to implement | Slow convergence |
| Low CPU/memory | Count-to-infinity loops |
| Easy to understand | Routing by rumour (no topology view) |
| Works on small networks | Periodic full-table updates waste bandwidth |
| No complex database | Hop-count metrics ignore bandwidth |

---

## Protocols That Use Distance Vector

| Protocol | Notes |
|---|---|
| RIP | Classic DV, hop count, max 15 |
| RIPv2 / RIPng | Classless / IPv6 versions |
| IGRP | Cisco legacy, composite metric (obsolete) |
| EIGRP | Advanced DV / hybrid, uses DUAL to avoid loops |
| BGP | Path vector — a DV variant that carries AS paths |

---

## Quick Revision

| Concept | Key Point |
|---|---|
| Distance vector | Distance + next hop, learned from neighbours |
| Bellman-Ford | `D(x,y) = min(cost(x,v) + D(v,y))` |
| Routing by rumour | No router knows the full topology |
| Count-to-infinity | Bad news propagates slowly, costs keep rising |
| Split horizon | Don't advertise a route back where it came from |
| Poison reverse | Advertise it back with infinity |
| Route poisoning | Announce failed route with infinity immediately |
| Hold-down | Ignore worse updates for a fixed time |
| Triggered update | Send changes immediately, not on timer |
| Infinity | Small finite value (RIP = 16) to stop counting |
| Convergence | Slow — the main weakness of DV |

- DV = simple, memory-light, convergence-heavy
- Split horizon + poison reverse + hold-down + triggered updates = the standard fix set
- EIGRP keeps DV's simplicity but uses DUAL for loop-free fast convergence
- BGP is path vector: same idea, but carries the full AS path

---

← [08-routing-fundamentals](08-routing-fundamentals.md) • [↑ Computer Networks](README.md) • [10-link-state-routing](10-link-state-routing.md) →
