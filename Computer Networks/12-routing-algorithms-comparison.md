# Routing Algorithms — Comparison

## The Landscape

```
Routing algorithms
├── Shortest path
│   ├── Dijkstra (link state)
│   ├── Bellman-Ford (distance vector)
│   └── Floyd-Warshall (all pairs, centralised)
├── Path vector (BGP)
├── Flooding
├── Broadcast
├── Multicast
└── Hierarchical routing
```

No single algorithm is "best" — each makes a different trade-off between **knowledge, convergence, resources, and policy**.

---

## Dijkstra vs Bellman-Ford

| | Dijkstra | Bellman-Ford |
|---|---|---|
| Used by | Link state (OSPF, IS-IS) | Distance vector (RIP) |
| Knowledge | Full topology | Neighbour advertisements only |
| Runs where | Every router, independently | Every router, independently |
| Handles negative weights | No | Yes |
| Complexity | O((V+E) log V) with heap | O(V·E) |
| Loop-free | Yes (with consistent LSDB) | Not inherently |
| Convergence | Fast | Slow |
| Failure mode | LSDB mismatch | Count-to-infinity |
| Output | Shortest path tree | Distance + next hop per destination |

```
Dijkstra: "I know the whole map, let me compute the best tree."
Bellman-Ford: "My neighbours tell me their costs; I relax edges until stable."
```

### Bellman-Ford in one line

```
repeat V−1 times:
  for every edge (u, v):
    if dist[u] + w(u,v) < dist[v]:
        dist[v] = dist[u] + w(u,v)
```

### Dijkstra in one line

```
repeatedly pick the unvisited node with the smallest distance,
then relax all of its edges.
```

Both produce shortest paths; they differ in **what information is available** and **how loops are prevented**.

---

## Other Shortest-Path Algorithms

### Floyd-Warshall

Computes **all-pairs** shortest paths in O(V³).

```
for k in nodes:
  for i in nodes:
    for j in nodes:
      dist[i][j] = min(dist[i][j], dist[i][k] + dist[k][j])
```

- Needs a centralised view of the whole graph.
- Useful for analysis, traffic matrices, and small networks.
- Not used as a distributed routing protocol.

### BFS / DFS

- **BFS** finds the shortest path by **hop count** in an unweighted graph — O(V+E).
- **DFS** finds *a* path, not the shortest — used for loop detection and graph traversal.

### Link-state vs distance-vector as algorithm families

| | Distance vector | Link state |
|---|---|---|
| Algorithm | Bellman-Ford | Dijkstra |
| Information | Local + neighbour distances | Global topology |
| Update | Full table to neighbours | LSAs flooded to all |
| Loop prevention | Split horizon, poison reverse, timers | SPF on consistent LSDB |
| Convergence | Slow | Fast |
| Memory | Low | High |
| CPU | Low | High |

Details: [09-distance-vector-routing](09-distance-vector-routing.md), [10-link-state-routing](10-link-state-routing.md).

---

## Distance Vector vs Link State vs Path Vector

| Dimension | Distance Vector | Link State | Path Vector |
|---|---|---|---|
| What is exchanged | Distances + next hops | Link states (LSAs) | Full AS paths + attributes |
| Topology knowledge | None (rumours) | Complete | Paths only |
| Loop prevention | Timers, split horizon, infinity | SPF | AS path check |
| Metric | Hop count / composite | Cost | Policy + attributes |
| Convergence | Slow | Fast | Slow but controlled |
| Scalability | Small | Large (with areas) | Global Internet |
| Policy | Minimal | Minimal (IGP) | Rich |
| Example | RIP | OSPF, IS-IS | BGP |

```
DV  = simple, cheap, fragile
LS  = complex, expensive, fast and loop-free
PV  = policy-driven, Internet-scale, deliberately conservative
```

---

## Flooding

Every incoming packet is sent out **every interface except the one it arrived on**.

```
        A
       / \
      B   C
     / \ /
    D   E

Packet arrives at B from A → B sends to D and E
E sends to C → C sends to A (but A already has it)
```

| Pros | Cons |
|---|---|
| Guaranteed delivery if a path exists | Exponential duplicate traffic |
| No routing table needed | Wastes bandwidth and CPU |
| Fastest possible propagation | Can congest the network |
| Simple | Duplicates consume buffers |

### Controlled flooding

| Technique | How it limits duplicates |
|---|---|
| TTL | Drop packet after N hops |
| Sequence numbers | Drop packets already seen |
| Source-based tracking | Remember (source, seq) pairs |
| Spanning tree | Forward only on tree edges |

Flooding is used for:
- Link-state LSA distribution
- ARP/DHCP discovery (broadcast)
- PIM-DM initial multicast flooding
- Network-wide alarms and discovery

---

## Broadcast Routing

Send one packet to **all nodes** in a network.

```
Methods:
1. Naive flooding         → every node, many duplicates
2. Controlled flooding    → sequence numbers / TTL
3. Spanning tree          → one copy per node, no loops
4. Reverse path forwarding (RPF) → forward only if packet
                                   arrived on the best path to the source
```

### Reverse Path Forwarding (RPF)

```
Router receives a broadcast from source S on interface I.
If I is the router's best path back to S → forward out all other interfaces.
Otherwise → drop.

Result: each packet follows the shortest-path tree rooted at S.
```

RPF is the basis of multicast routing (see below).

---

## Multicast Routing (Overview)

Send one packet to a **group** of receivers.

```
Source ──► R1 ──► R2 ──► receivers
              └──► R3 ──► receivers

One copy on each link where receivers exist.
```

Key concepts:

- **Group address:** IPv4 `224.0.0.0/4`
- **Distribution tree:** source tree `(S,G)` or shared tree `(*,G)`
- **RPF check:** accept only on the path back to the source/RP
- **Protocols:** IGMP (host signalling), PIM-DM (dense), PIM-SM (sparse), DVMRP, MOSPF

Details: [22-multicast-routing](22-multicast-routing.md).

---

## Hierarchical Routing

Flat routing does not scale — every router would need an entry for every destination.

```
Flat:
  Every router knows all N destinations → table size O(N)
  Link-state flooding → O(N) updates everywhere

Hierarchical:
  Divide into regions/areas/ASes
  Routers know their area in detail
  Other areas are summarised
  → table size and update scope shrink dramatically
```

### Example

```
        ┌──────────── Area 0 ────────────┐
        │                                │
   ┌────┴────┐                     ┌─────┴─────┐
   │ Area 1  │                     │ Area 2    │
   │ 20 nets │                     │ 30 nets   │
   └─────────┘                     └───────────┘

Flat: every router needs 50+ entries
Hierarchical: Area 1 routers need ~20 local + a few summaries
```

### Hierarchical levels on the Internet

```
Internet
└── AS (BGP)
    └── Area (OSPF)
        └── Router
            └── Link
```

Benefits:
- Smaller routing tables
- Faster convergence (changes stay local)
- Easier policy and administration
- Summarisation hides instability

Costs:
- Suboptimal paths (traffic may cross a hierarchy boundary unnecessarily)
- More complex design

---

## Quick Revision

| Algorithm | Used By | Strength | Weakness |
|---|---|---|---|
| Dijkstra | OSPF, IS-IS | Fast, loop-free | Needs full topology |
| Bellman-Ford | RIP | Simple, low resource | Slow, count-to-infinity |
| Path vector | BGP | Policy, loop-free via AS path | Slow convergence |
| Floyd-Warshall | Analysis | All-pairs shortest paths | Centralised, O(V³) |
| Flooding | LSA distribution | Guaranteed delivery | Duplicates |
| Spanning tree | Broadcast | One copy per node | Tree maintenance |
| RPF | Broadcast/multicast | Loop-free distribution | Needs unicast route to source |
| Hierarchical | OSPF areas, BGP ASes | Scales, summarises | Some suboptimality |

- Dijkstra needs the map; Bellman-Ford needs only neighbours
- DV is cheap and fragile; LS is expensive and robust
- Path vector trades optimality for policy and stability
- Flooding is a building block, not a complete routing solution
- Hierarchy is what makes routing scale to the Internet

---

← [11-path-vector-routing](11-path-vector-routing.md) • [↑ Computer Networks](README.md) • [13-intra-domain-routing](13-intra-domain-routing.md) →
