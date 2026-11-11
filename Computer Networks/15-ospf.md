# OSPF — Open Shortest Path First

## What It Is

**OSPF** is a link-state IGP. Every router builds a complete map of its area, then runs **Dijkstra's SPF algorithm** to compute the shortest path to every destination.

```
OSPF router:
  1. Discover neighbours (Hello)
  2. Exchange link states (LSAs)
  3. Build the same LSDB as everyone else in the area
  4. Run SPF → shortest path tree → routing table
```

| Property | Value |
|---|---|
| Type | Link state |
| Algorithm | Dijkstra / SPF |
| Metric | Cost (bandwidth-based) |
| Transport | IP protocol 89 |
| Multicast | 224.0.0.5 (all), 224.0.0.6 (DR/BDR) |
| Standard | RFC 2328 (v2), RFC 5340 (v3) |
| Admin distance | 110 |

---

## OSPF Operation

```
Step 1: Hello          → discover neighbours, elect DR/BDR
Step 2: Database sync  → exchange DBDs, then LSAs
Step 3: LSDB           → identical database across the area
Step 4: SPF            → Dijkstra computes shortest paths
Step 5: Routing table  → best routes installed
Step 6: Maintenance    → Hellos refresh, changes trigger updates
```

---

## OSPF Packet Types

| Type | Name | Purpose |
|---|---|---|
| 1 | Hello | Discover/maintain neighbours, elect DR/BDR |
| 2 | Database Description (DBD) | Summarise LSDB contents |
| 3 | Link State Request (LSR) | Ask for a specific LSA |
| 4 | Link State Update (LSU) | Send one or more LSAs |
| 5 | Link State Acknowledgment (LSAck) | Confirm receipt of LSAs |

```
Neighbour discovery:
  R1 ── Hello ──► R2
  R1 ◄─ Hello ─── R2
  ↓
  DBD exchange (what do you have?)
  ↓
  LSR / LSU (send me the missing LSAs)
  ↓
  LSAck (got it)
  ↓
  Full adjacency
```

---

## Hello Protocol

The Hello packet carries:

| Field | Purpose |
|---|---|
| Router ID | Unique 32-bit identifier |
| Hello interval | How often Hellos are sent |
| Dead interval | How long before declaring neighbour dead |
| Area ID | Must match to form adjacency |
| Authentication | Type and data |
| Stub flags | Area type information |
| Priority | DR/BDR election |
| DR / BDR | Current designated routers |
| Neighbour list | Routers seen on the segment |

**These must match** for an adjacency to form:

```
- Area ID
- Hello and dead intervals
- Authentication
- Stub area flag
- MTU (for DBD exchange)
- Network type compatibility
```

Default intervals:

| Network type | Hello | Dead |
|---|---|---|
| Broadcast | 10 s | 40 s |
| Point-to-point | 10 s | 40 s |
| NBMA | 30 s | 120 s |
| Point-to-multipoint | 30 s | 120 s |

Dead interval is normally **4× Hello**.

---

## Neighbour States

```
Down
  │  Hello received
  ▼
Init
  │  own router ID seen in neighbour's Hello
  ▼
2-Way ────────────────► (DROther ↔ DROther stays here)
  │  DR/BDR election, start database exchange
  ▼
ExStart
  │  negotiate master/slave, initial DBD
  ▼
Exchange
  │  exchange DBDs
  ▼
Loading
  │  request missing LSAs (LSR/LSU)
  ▼
Full  ← adjacency complete
```

| State | Meaning |
|---|---|
| Down | No Hellos received |
| Init | Hello received, but we are not in the neighbour's list |
| 2-Way | Bidirectional communication established |
| ExStart | Master/slave negotiation for DBD exchange |
| Exchange | DBDs being exchanged |
| Loading | Requesting full LSAs |
| Full | LSDBs synchronised |

```
On a broadcast segment, DROther-to-DROther stays in 2-Way.
That is normal — full adjacency is only needed with the DR/BDR.
```

---

## DR and BDR

On a **broadcast** or **NBMA** segment, OSPF elects a **Designated Router (DR)** and **Backup DR (BDR)** to reduce the number of adjacencies.

```
Without DR:
  5 routers → 5×4/2 = 10 adjacencies

With DR/BDR:
  5 routers → all form adjacency with DR/BDR → far fewer LSA exchanges
```

### Election

```
1. Highest OSPF interface priority wins (default 1, range 0–255)
2. Tie → highest router ID
3. Priority 0 → never DR/BDR

The election is NOT preemptive:
  A new router with a higher priority does not take over
  until the current DR fails.
```

```
DR  = 224.0.0.6 listener + 224.0.0.5 sender
BDR = 224.0.0.6 listener + 224.0.0.5 sender
DROther = 224.0.0.5 listener only
```

If the DR fails, the BDR takes over and a new BDR is elected.

---

## OSPF Areas

```
                 ┌──────────────┐
                 │   Area 0     │
                 │  (backbone)  │
                 └───┬──────┬───┘
                     │      │
              ┌──────┴─┐  ┌─┴──────┐
              │ Area 1 │  │ Area 2 │
              └────────┘  └────────┘
```

| Router type | Role |
|---|---|
| Internal router | All interfaces in one area |
| Backbone router | At least one interface in Area 0 |
| ABR (Area Border Router) | Connects an area to Area 0 |
| ASBR (AS Boundary Router) | Connects OSPF to another AS (BGP, static, etc.) |

### Area types

| Area type | Allows | Blocks |
|---|---|---|
| Normal | All LSAs | — |
| Stub | Type 1, 2, 3 + default | Type 5 (external) |
| Totally stubby | Type 1, 2 + default | Type 3, 4, 5 |
| NSSA | Type 1, 2, 3, 7 | Type 5 |
| Totally NSSA | Type 1, 2, 7 + default | Type 3, 4, 5 |

```
Area 0 is the backbone — all other areas must connect to it.
If they cannot physically, use a virtual link.
```

Why areas?
- Smaller LSDB per router
- Less SPF computation
- Topology changes stay local
- Summarisation at area boundaries

---

## LSA Types

| Type | Name | Originated by | Scope |
|---|---|---|---|
| 1 | Router LSA | Every router | Area |
| 2 | Network LSA | DR | Area |
| 3 | Summary LSA | ABR | Area |
| 4 | ASBR Summary LSA | ABR | Area |
| 5 | AS External LSA | ASBR | Whole AS |
| 6 | Group Membership LSA | MOSPF (unused) | Area |
| 7 | NSSA External LSA | ASBR in NSSA | NSSA |

```
Type 1: "Here are my links and their costs."
Type 2: "I am the DR on this broadcast segment; these routers are attached."
Type 3: "I can reach 10.2.0.0/16 through me (from another area)."
Type 4: "The ASBR is reachable at this router."
Type 5: "Here is an external route (from BGP/static/redistribution)."
Type 7: "Here is an external route inside an NSSA."
```

OSPFv3 adds:
- Type 8: Link LSA
- Type 9: Intra-Area Prefix LSA

---

## OSPF Cost

```
cost = reference bandwidth / interface bandwidth

Default reference bandwidth = 100 Mbps

  10 Mbps   → 100/10  = 10
  100 Mbps  → 100/100 = 1
  1 Gbps    → 100/1000 = 1 (rounded up to minimum 1)
  10 Gbps   → 1
```

```
Problem: all links ≥ 100 Mbps look the same by default.
Fix: auto-cost reference-bandwidth 10000   (10 Gbps)
     or set interface cost manually.
```

Path cost = sum of the **outgoing interface costs** along the path.

```
A ──10── B ──10── C
A → C cost = 20
```

Cost is a 16-bit value (1–65535). The lowest total cost wins; equal costs give **ECMP**.

---

## Network Types

| Type | DR/BDR | Hello/Dead | Use |
|---|---|---|---|
| Broadcast | Yes | 10/40 | Ethernet |
| Point-to-point | No | 10/40 | Serial, GRE tunnels |
| NBMA | Yes | 30/120 | Frame Relay, ATM |
| Point-to-multipoint | No | 30/120 | Hub-and-spoke |
| Virtual link | No | — | Area 0 extension |

On point-to-point links, OSPF skips DR/BDR election because there are only two routers.

---

## Adjacency Requirements

Two routers become neighbours only if these match:

```
✓ Area ID
✓ Hello and dead intervals
✓ Authentication
✓ Stub area flag
✓ MTU (for DBD exchange)
✓ Compatible network types
✓ Unique router IDs
```

Common reasons an adjacency stays down:

```
- mismatched area ID
- mismatched hello/dead timers
- authentication mismatch
- MTU mismatch (stuck in ExStart/Exchange)
- duplicate router ID
- ACL blocking IP protocol 89 or multicast 224.0.0.5
- passive interface misconfiguration
```

---

## Configuration (Cisco IOS)

### Basic OSPFv2

```
router ospf 1
 router-id 1.1.1.1
 network 10.0.0.0 0.0.0.255 area 0
 network 192.168.1.0 0.0.0.255 area 1
 passive-interface GigabitEthernet0/1
```

### Interface tuning

```
interface GigabitEthernet0/0
 ip ospf cost 10
 ip ospf priority 100
 ip ospf hello-interval 5
 ip ospf dead-interval 20
```

### Reference bandwidth

```
router ospf 1
 auto-cost reference-bandwidth 10000
```

### Area types

```
router ospf 1
 area 1 stub
 area 1 nssa
```

### OSPFv3 (IPv6)

```
ipv6 unicast-routing
ipv6 router ospf 1
 router-id 1.1.1.1

interface GigabitEthernet0/0
 ipv6 ospf 1 area 0
 ipv6 ospf cost 10
```

---

## Verification Commands

```
show ip ospf neighbor
show ip ospf database
show ip ospf interface
show ip ospf
show ip route ospf
show ip protocols
debug ip ospf adj
debug ip ospf hello
```

`show ip ospf neighbor` states worth noting:

```
FULL/DR      full adjacency, this neighbour is DR
FULL/BDR     full adjacency, this neighbour is BDR
FULL/DROTHER full adjacency on a point-to-point link
2WAY/DROTHER normal on broadcast segments for DROther peers
```

---

## OSPFv3 vs OSPFv2

| Feature | OSPFv2 | OSPFv3 |
|---|---|---|
| Address family | IPv4 | IPv6 (and address families via AF support) |
| Router ID | 32-bit | 32-bit (still IPv4-style) |
| Link addressing | Per subnet | Per link |
| Next hop | IPv4 address | Link-local address |
| Authentication | Built-in (MD5/SHA) | IPsec (protocol-independent) |
| LSA types | 1–7 | 1–7 plus 8 (Link), 9 (Intra-Area Prefix) |
| Multicast | 224.0.0.5 / .6 | FF02::5 / FF02::6 |
| Transport | IP 89 | IP 89 |

```
OSPFv3 runs per link, not per subnet — multiple IPv6 prefixes
can share one adjacency.
```

---

## Troubleshooting Checklist

```
1. Are interfaces up/up and in the right area?
2. Do hello/dead timers match?
3. Does authentication match?
4. Is MTU consistent?
5. Are router IDs unique?
6. Is IP protocol 89 allowed by ACLs/firewalls?
7. Is the interface passive when it should be?
8. Does `show ip ospf neighbor` show FULL?
9. Does `show ip ospf database` contain the expected LSAs?
10. Does `show ip route ospf` contain the expected prefixes?
```

---

## Quick Revision

| Concept | Key Point |
|---|---|
| Type | Link state, Dijkstra |
| Metric | Cost = ref BW / interface BW |
| Transport | IP protocol 89 |
| Multicast | 224.0.0.5, 224.0.0.6 |
| Packet types | Hello, DBD, LSR, LSU, LSAck |
| Neighbour states | Down → Init → 2-Way → ExStart → Exchange → Loading → Full |
| DR/BDR | Highest priority, then router ID; non-preemptive |
| Area 0 | Backbone; all areas connect to it |
| ABR / ASBR | Area boundary / AS boundary |
| LSA 1/2 | Router / network (intra-area) |
| LSA 3/4 | Summary (inter-area) |
| LSA 5/7 | External / NSSA external |
| Admin distance | 110 |
| Convergence | Fast, loop-free |

- OSPF = link state with areas and SPF
- DR/BDR cuts adjacency count on broadcast segments
- Costs are bandwidth-based but flat above 100 Mbps unless tuned
- Hello/dead, area, auth, and MTU must match for adjacency
- LSA types tell you what information is being flooded where

---

← [14-rip](14-rip.md) • [↑ Computer Networks](README.md) • [18-inter-domain-routing](18-inter-domain-routing.md) →
