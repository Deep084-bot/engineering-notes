# Multicast Routing

## What It Is

**Multicast** delivers one copy of a packet to a **group** of receivers. The network replicates the packet only where paths diverge.

```
Unicast:    one sender → one receiver        (N copies for N receivers)
Broadcast:  one sender → everyone on a LAN   (does not cross routers)
Multicast:  one sender → a group             (replication only where needed)
Anycast:    one sender → nearest of a group  (one receiver)
```

```
              ┌── R2 ── receiver 1
Source ── R1 ─┤
              └── R3 ── R4 ── receiver 2
                          └── receiver 3

One stream from the source; routers replicate at branch points.
```

Why multicast?
- One sender can serve thousands of receivers without N× bandwidth at the source.
- Efficient for live video, market data, software distribution, conferencing.

---

## Multicast Addressing

IPv4 multicast uses **Class D**: `224.0.0.0/4` (224.0.0.0 – 239.255.255.255).

| Range | Use |
|---|---|
| `224.0.0.0/24` | Local network control (never forwarded) |
| `224.0.0.0/4` | Global multicast |
| `232.0.0.0/8` | Source-Specific Multicast (SSM) |
| `239.0.0.0/8` | Administratively scoped (private) |

Common addresses:

| Address | Meaning |
|---|---|
| `224.0.0.1` | All hosts on this subnet |
| `224.0.0.2` | All routers on this subnet |
| `224.0.0.5` | All OSPF routers |
| `224.0.0.6` | OSPF DR/BDR |
| `224.0.0.9` | RIPv2 |
| `224.0.0.13` | PIM routers |
| `224.0.1.1` | NTP (historical) |

```
A multicast address identifies a GROUP, not a host.
Receivers join a group; senders simply send to the group address.
```

---

## Multicast MAC Mapping

IPv4 multicast maps into Ethernet multicast MAC addresses:

```
MAC = 01:00:5E : 0 + lower 23 bits of the IPv4 multicast address

224.0.0.5   → 01:00:5E:00:00:05
239.1.1.1   → 01:00:5E:01:01:01
```

```
Only 23 bits of the 28-bit IPv4 multicast space are used
→ 32 different multicast IPs map to the same MAC address.
```

This is why hosts must filter at IP level — the NIC may receive several groups' frames.

---

## IGMP — Host-to-Router Signalling

**IGMP (Internet Group Management Protocol)** lets hosts tell routers which groups they want.

| Version | Features |
|---|---|
| IGMPv1 | Join only; router queries; no leave |
| IGMPv2 | Join, leave, group-specific queries |
| IGMPv3 | Source-specific joins (SSM), source filtering |

```
IGMP runs between hosts and their local multicast router.
It does NOT route multicast between networks.
```

### IGMPv2 operation

```
Router ── Membership Query (224.0.0.1) ──► hosts
Host   ── Membership Report (join)     ──► router
Host   ── Leave Group                  ──► router
Router ── Group-Specific Query         ──► check if others remain
```

```
Queries are sent periodically (default 125 s).
A host that wants a group sends a report; the router adds the
interface to the group's forwarding state.
```

### IGMP snooping

A Layer 2 switch normally floods multicast frames to all ports. **IGMP snooping** lets the switch listen to IGMP messages and forward multicast only to ports that have interested receivers.

```
Without snooping: multicast floods every port
With snooping:    multicast goes only to subscribed ports
```

---

## Multicast Routing Concepts

### Distribution tree

Multicast uses a tree rooted at the source or at a rendezvous point.

```
Source tree (S,G):
  Root = the source
  One tree per source
  Shortest paths to receivers
  More state, optimal paths

Shared tree (*,G):
  Root = a rendezvous point (RP)
  One tree per group
  Less state, possibly longer paths
```

```
(S,G) = "traffic from source S to group G"
(*,G) = "traffic from any source to group G"
```

### Reverse Path Forwarding (RPF)

Multicast routing cannot use normal destination-based forwarding — the destination is a group, not a host. Instead, routers check the **source**.

```
RPF check:
  Did this packet arrive on the interface that is the
  unicast shortest path BACK to the source?

  Yes → forward (if the outgoing interface list allows)
  No  → drop (prevents loops and duplicates)
```

```
         Source S
            │
           R1
          /  \
        R2    R3
         \    /
           R4

R4 accepts multicast from S only on the interface
that R4 would use to send unicast traffic to S.
Packets arriving on the other interface are dropped.
```

RPF is the foundation of every multicast routing protocol.

---

## Multicast Routing Protocols

| Protocol | Type | How it builds the tree |
|---|---|---|
| DVMRP | Dense | Flood and prune, distance-vector |
| MOSPF | Dense | Link-state, uses OSPF LSDB |
| PIM-DM | Dense | Flood and prune |
| PIM-SM | Sparse | Explicit joins to an RP, then SPT |
| PIM-SSM | Sparse | Source-specific joins, no RP |
| CBT | Sparse | Core-based shared tree |

```
Dense mode: assume everyone wants the traffic, then prune.
Sparse mode: assume nobody wants it, then join explicitly.
```

---

## PIM-DM — Dense Mode

```
1. Multicast traffic arrives → flood out all RPF interfaces
2. Routers with no receivers send Prune upstream
3. Pruned branches are re-flooded periodically (every ~3 min)
4. If a new receiver appears → Graft to add the branch back
```

```
Source ── R1 ── R2 ── no receivers → prune
              └─ R3 ── receiver    → keep
```

| Pros | Cons |
|---|---|
| Simple, no RP | Periodic flood/prune wastes bandwidth |
| Good for dense receiver populations | Poor for sparse groups |
| Fast to build | State is refreshed constantly |

### Assert

On a multi-access LAN, multiple routers may forward the same multicast. **PIM Assert** elects one forwarder (lowest metric to source, then highest IP) and the others stop forwarding.

---

## PIM-SM — Sparse Mode

PIM-SM assumes receivers are sparse. It uses a **Rendezvous Point (RP)**.

```
1. Receiver joins group G:
     IGMP report → local router
     router sends (*,G) Join toward the RP

2. Source starts sending:
     first-hop router encapsulates traffic in a Register
     message to the RP

3. RP forwards down the shared tree (*,G) to receivers

4. Last-hop router may switch to the source-specific tree (S,G)
     → SPT switchover for a shorter path

5. RP stops receiving Register-encapsulated traffic for that source
```

```
                 RP
                /  \
   Source ── R1     R2 ── Receiver
              \    /
               R3

Shared tree (*,G): Source → R1 → RP → R2 → Receiver
Source tree (S,G): Source → R1 → R3 → R2 → Receiver  (after switchover)
```

| Message | Purpose |
|---|---|
| Join / Prune | Build or remove tree branches |
| Register | Encapsulate source traffic to RP |
| Register-Stop | RP tells first-hop router to stop registering |
| Assert | Elect one forwarder on a LAN |
| Bootstrap / Candidate-RP | RP discovery (BSR) |

### RP discovery

```
Static:      manually configure the RP on every router
Auto-RP:     Cisco proprietary, candidate RPs announce via multicast
BSR:         BootStrap Router, standards-based RP election
Anycast RP:  same RP address on multiple routers; MSDP keeps them in sync
```

### PIM-SSM

```
No RP. Receiver signals (S,G) directly via IGMPv3.
Router sends (S,G) Join toward the source.
Used for source-specific applications (e.g., IPTV channels).
```

---

## Inter-Domain Multicast

```
- MBGP (Multicast BGP): carries multicast RPF information
  (AFI/SAFI 2) so RPF checks work across ASes.
- MSDP (Multicast Source Discovery Protocol): connects RPs in
  different domains, shares source-active (SA) messages.
- PIM-SM with MSDP is the traditional inter-domain design.
- PIM-SSM avoids RPs and is simpler across domains.
```

```
Unicast BGP says "where is the prefix?"
MBGP says "where is the source for RPF?"
MSDP says "which sources are active in your domain?"
```

---

## Multicast State

Routers keep per-group (or per-source-group) state:

```
(S,G) state:
  incoming interface (RPF interface)
  outgoing interface list (OIL)
  flags (S, G, etc.)
```

```
Multicast is stateful and receiver-driven.
Unicast is destination-driven; multicast is source-and-group-driven.
```

---

## Use Cases

| Use case | Why multicast |
|---|---|
| Live video / IPTV | One stream to many viewers |
| Financial market data | Same data to many subscribers |
| Video conferencing | One-to-many media distribution |
| OS/software deployment | One image to many machines |
| Routing protocols | OSPF, RIPv2, PIM use multicast |
| Service discovery | mDNS, SSDP |

---

## Quick Revision

| Concept | Key Point |
|---|---|
| Multicast | One packet to a group |
| IPv4 range | 224.0.0.0/4 (Class D) |
| SSM range | 232.0.0.0/8 |
| Admin scoped | 239.0.0.0/8 |
| MAC mapping | 01:00:5E + lower 23 bits |
| IGMP | Host ↔ local router group membership |
| IGMP snooping | L2 switch forwards only to interested ports |
| Distribution tree | Source (S,G) or shared (*,G) |
| RPF | Accept only on the path back to the source |
| PIM-DM | Flood and prune (dense) |
| PIM-SM | RP + explicit joins (sparse) |
| PIM-SSM | Source-specific, no RP |
| RP | Rendezvous point for shared tree |
| MBGP | Multicast RPF info across ASes |
| MSDP | Connects RPs across domains |

- Multicast is receiver-driven and stateful
- RPF is the core loop-prevention mechanism
- Dense mode floods then prunes; sparse mode joins explicitly
- PIM-SM uses an RP shared tree, then optionally switches to SPT
- SSM removes the RP entirely
- IGMP only signals local membership; PIM does the routing

---

← [21-route-summarization-and-ecmp](21-route-summarization-and-ecmp.md) • [↑ Computer Networks](README.md) • [23-mpls-and-label-switching](23-mpls-and-label-switching.md) →
