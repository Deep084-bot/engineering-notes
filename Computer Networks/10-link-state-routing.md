# Link State Routing

## What It Is

**Link state (LS)** routing gives every router a **complete map of the topology**. Each router:

1. Discovers its directly connected neighbours and links
2. Builds a **Link State Advertisement (LSA)** describing those links
3. **Floods** the LSA to every other router in the area
4. Builds an identical **Link State Database (LSDB)** from all LSAs
5. Runs **Dijkstra's algorithm** on the LSDB to compute shortest paths
6. Installs the resulting best routes into the routing table

```
DV: "my neighbour says the destination is 3 hops away"
LS: "I have the full map, I compute the shortest path myself"
```

---

## The Five Steps

```
1. DISCOVER neighbours        → Hello packets
2. MEASURE link cost          → bandwidth, delay, configured cost
3. BUILD LSA                  → "I am R1; my link to R2 costs 10"
4. FLOOD LSA                  → every router gets every LSA
5. COMPUTE SPF                → Dijkstra → shortest path tree → routes
```

---

## Step 1–3: Discover, Measure, Advertise

Each router periodically sends **Hello** packets on each interface to find neighbours and verify they are alive.

```
R1 ────── R2
   Hello →
   ← Hello
   (neighbours discovered)

R1 builds an LSA:
  Router ID: 1.1.1.1
  Link 1: to 2.2.2.2, cost 10
  Link 2: to 3.3.3.3, cost 5
  Sequence: 0000A1
  Age: 0
```

An LSA contains:
- Originating router ID
- List of neighbours and link costs
- **Sequence number** (newer wins)
- **Age** (to expire stale information)

---

## Step 4: Flooding

When a router receives a new LSA, it:

1. Checks the sequence number — is it newer?
2. If yes: stores it, then **forwards it out every interface except the one it arrived on**
3. If no: discards it

```
R1 ── R2 ── R3
      │
      R4

R1's LSA reaches R2 → R2 floods to R3 and R4 → R3 floods to R4 ...
Every router eventually has the same LSDB.
```

Flooding is reliable because each LSA is acknowledged and retransmitted until all neighbours have it. Duplicates are suppressed using sequence numbers.

```
An LSA is replaced only when:
  sequence number is higher
  OR sequence is equal but age is younger (fresher)
```

---

## Step 5: Dijkstra's Algorithm (SPF)

Each router runs **Dijkstra's shortest path first (SPF)** using itself as the root. The result is a **shortest path tree** — the best path to every destination.

### Algorithm

```
1. Mark all nodes unvisited.
2. Set distance to self = 0, all others = ∞.
3. Pick the unvisited node with the smallest distance.
4. For each neighbour of that node:
     candidate = current distance + link cost
     if candidate < neighbour's distance: update it
5. Mark the node visited.
6. Repeat 3–5 until all nodes are visited.
```

### Worked Example

Topology:

```
        A
     1 / \ 4
      /   \
     B─────C
     | \2  | \
    5|  \  |1 \6
     |   \ |   \
     D─────┴────E
        3
```

Edges:
```
A–B = 1    A–C = 4
B–C = 2    B–D = 5
C–D = 1    C–E = 6
D–E = 3
```

Run Dijkstra from **A**:

| Step | Selected | B | C | D | E |
|---|---|---|---|---|---|
| Init | — | ∞ | ∞ | ∞ | ∞ |
| 1 | A (0) | 1 via A | 4 via A | ∞ | ∞ |
| 2 | B (1) | 1 via A | 3 via B | 6 via B | ∞ |
| 3 | C (3) | 1 via A | 3 via B | 4 via C | 9 via C |
| 4 | D (4) | 1 via A | 3 via B | 4 via C | 7 via D |
| 5 | E (7) | 1 via A | 3 via B | 4 via C | 7 via D |

Final shortest path tree from A:

```
A ──1── B ──2── C ──1── D ──3── E
                        (total A→E = 1+2+1+3 = 7)
```

```
A's routing table:
  B via link A–B, cost 1
  C via B,       cost 3
  D via B→C,     cost 4
  E via B→C→D,   cost 7
```

Because every router runs the same algorithm on the same LSDB, all routers compute **consistent, loop-free** paths.

---

## Link State Database (LSDB)

```
LSDB = all LSAs from all routers in the area

  R1's LSA: links R1–R2 (10), R1–R3 (5)
  R2's LSA: links R2–R1 (10), R2–R4 (2)
  R3's LSA: links R3–R1 (5),  R3–R4 (8)
  R4's LSA: links R4–R2 (2),  R4–R3 (8)

Every router in the area has this same database.
Different areas have different LSDBs (see OSPF areas).
```

If the LSDBs differ, routers can compute inconsistent paths → routing loops. LS protocols use reliable flooding and checksums to prevent this.

---

## LSAs, Sequence Numbers, and Aging

| Mechanism | Purpose |
|---|---|
| Sequence number | Newer LSA replaces older one |
| Age | Stops stale LSAs living forever |
| Checksum | Detects corruption |
| Acknowledgment | Makes flooding reliable |
| Refresh timer | Re-floods LSA before it expires |

```
Router reboots → sequence number resets
→ neighbours may ignore its LSAs (they have higher sequence)
→ protocol handles this with a "max age" flush and re-origin
```

---

## Areas and Hierarchy

Running SPF over a huge topology is expensive. Link state protocols split the network into **areas**:

```
        ┌──────────── Area 0 (backbone) ────────────┐
        │                                            │
   ┌────┴────┐                              ┌────────┴────┐
   │ Area 1  │                              │ Area 2      │
   │         │                              │             │
   └─────────┘                              └─────────────┘

Area 1 routers know Area 1's topology in detail.
They learn Area 2/0 destinations as summaries, not full LSAs.
```

- **ABR (Area Border Router):** connects an area to the backbone.
- **ASBR (AS Autonomous System Boundary Router):** connects the AS to another AS.
- Areas reduce LSDB size, SPF churn, and blast radius of topology changes.

Details: [15-ospf](15-ospf.md).

---

## Pros and Cons

| Pros | Cons |
|---|---|
| Fast convergence | More CPU and memory (full LSDB) |
| Loop-free by design | Complex flooding and database sync |
| Every router has full topology | SPF computation can be heavy |
| Scales with areas/hierarchy | More complex to configure/troubleshoot |
| Supports traffic engineering (with extensions) | Bandwidth overhead for flooding |

---

## Distance Vector vs Link State

| | Distance Vector | Link State |
|---|---|---|
| Knowledge | Distances + next hops | Full topology |
| Updates | Full table to neighbours | LSAs flooded to all |
| Trigger | Periodic timer | Change-driven + refresh |
| Algorithm | Bellman-Ford | Dijkstra |
| Convergence | Slow | Fast |
| Loops | Count-to-infinity risk | Loop-free by design |
| Resources | Low CPU/memory | Higher CPU/memory |
| Scalability | Small networks | Large networks (with areas) |
| Examples | RIP, EIGRP (hybrid) | OSPF, IS-IS |

---

## Protocols That Use Link State

| Protocol | Notes |
|---|---|
| OSPF | IP protocol 89, areas, LSA flooding |
| OSPFv3 | OSPF for IPv6 |
| IS-IS | ISO/IEC, levels instead of areas, TLV-based |
| MOSPF | Multicast extension of OSPF (rare) |
| PNNI | ATM (historical) |

---

## Quick Revision

| Concept | Key Point |
|---|---|
| Link state | Full topology map on every router |
| LSA | Router's description of its links and costs |
| Flooding | Reliable dissemination of LSAs |
| LSDB | Identical database of all LSAs in an area |
| SPF / Dijkstra | Computes shortest path tree from LSDB |
| Sequence number | Newer LSA wins |
| Age | Expires stale LSAs |
| Area | Limits LSDB size and SPF scope |
| ABR / ASBR | Area boundary / AS boundary routers |
| Convergence | Fast, loop-free |
| Cost | More CPU/memory than DV |

- LS = "map + Dijkstra", DV = "rumours + Bellman-Ford"
- Same LSDB → consistent, loop-free paths
- Flooding must be reliable, or databases diverge
- Areas are how link state scales
- OSPF and IS-IS are the main LS protocols in use

---

← [09-distance-vector-routing](09-distance-vector-routing.md) • [↑ Computer Networks](README.md) • [11-path-vector-routing](11-path-vector-routing.md) →
