# MPLS and Label Switching

## What It Is

**MPLS (Multiprotocol Label Switching)** forwards packets using short fixed-length **labels** instead of doing an IP lookup at every hop. It sits between Layer 2 and Layer 3 — often called **Layer 2.5**.

```
IP routing:      every router looks up the destination IP
MPLS forwarding: ingress assigns a label; each hop swaps it;
                 egress removes it
```

```
        ┌──────────── LSP ────────────┐
Ingress ──► LSR ──► LSR ──► LSR ──► Egress
  push       swap    swap    swap    pop
```

| Term | Meaning |
|---|---|
| LSR | Label Switch Router |
| LER | Label Edge Router (ingress/egress) |
| LSP | Label Switched Path |
| FEC | Forwarding Equivalence Class — a set of packets treated the same |
| Label | 20-bit identifier used for forwarding |

---

## Why MPLS

```
1. Faster forwarding       → fixed-length label lookup vs longest-prefix match
2. Traffic engineering     → explicit paths, bandwidth reservation
3. VPNs                    → L3VPN, L2VPN, VPLS over one infrastructure
4. Protocol independence   → carries IPv4, IPv6, Ethernet, etc.
5. Fast reroute            → pre-computed backup paths
6. Service provider scale  → one core for many customers
```

```
Modern hardware makes plain IP lookup fast, so raw speed is no longer
the main reason. Traffic engineering and VPN services are.
```

---

## The MPLS Label

A label is 32 bits:

```
┌──────────────────────┬─────────┬───────┬──────────┐
│   Label (20 bits)    │ TC (3)  │ S (1) │ TTL (8)  │
└──────────────────────┴─────────┴───────┴──────────┘
```

| Field | Meaning |
|---|---|
| Label | 20-bit forwarding identifier (0–1,048,575) |
| TC / EXP | Traffic class (QoS) |
| S | Bottom-of-stack bit (1 = last label) |
| TTL | Time to live (loop prevention) |

Reserved labels:

| Label | Meaning |
|---|---|
| 0 | IPv4 explicit null |
| 2 | IPv6 explicit null |
| 3 | Implicit null (penultimate hop popping) |
| 14 | OAM alert |

---

## Label Stack

MPLS supports a **stack** of labels. Each label is pushed/popped independently.

```
┌──────────────┬──────────────┬──────────────┐
│  Top label   │  Inner label │  Payload     │
│  (transport) │  (VPN)       │  (IP packet) │
└──────────────┴──────────────┴──────────────┘
        S=0            S=1
```

```
Outer label: gets the packet across the provider core
Inner label: identifies the customer/VPN at the egress PE
```

Label operations:

| Operation | Where | What |
|---|---|---|
| Push | Ingress LER | Add a label to the stack |
| Swap | Core LSR | Replace top label with a new one |
| Pop | Egress LER / PHP router | Remove the top label |

```
Ingress: push(label 100)
Core:    swap(100 → 200) → swap(200 → 300)
Egress:  pop → forward IP packet
```

---

## Label Distribution

Routers must agree on what label means what. **Label distribution protocols** do this.

### LDP — Label Distribution Protocol

```
- Runs over TCP port 646 (UDP 646 for discovery)
- Distributes labels for IGP-learned prefixes
- Follows the IGP shortest path
- Builds LSPs automatically
```

```
LDP operation:
  1. Discover neighbours (UDP hello)
  2. Establish TCP session
  3. Exchange label mappings
  4. Install label forwarding entries
```

LDP modes:

| Mode | Meaning |
|---|---|
| Downstream unsolicited (DU) | Advertise labels without a request (default) |
| Downstream on demand (DoD) | Send a label only when asked |
| Liberal retention | Keep labels from all neighbours |
| Conservative retention | Keep only the best next hop's label |
| Independent control | Each router decides labels locally |
| Ordered control | Label assigned only after downstream label exists |

### RSVP-TE — Resource Reservation Protocol, Traffic Engineering

```
- Explicitly signals an LSP along a chosen path
- Can reserve bandwidth
- Supports Fast Reroute (FRR)
- Used for traffic engineering, not just label distribution
```

### BGP

```
- MP-BGP distributes labels for VPN routes (VPNv4/VPNv6)
- Used in MPLS L3VPNs
```

---

## Label Switched Path (LSP)

An LSP is the path a labelled packet takes from ingress to egress.

```
        label 100      label 200      label 300
Ingress ────────► LSR1 ────────► LSR2 ────────► Egress
   push            swap            swap           pop
```

```
The LSP is unidirectional.
Return traffic uses a different LSP.
```

### Penultimate Hop Popping (PHP)

```
The second-to-last router pops the label and sends the packet
unlabelled (or with implicit null) to the egress.

Benefit: egress avoids one label lookup.
```

```
LSR2 (penultimate) ── pop ──► Egress (plain IP)
```

---

## MPLS L3VPN

Service providers use MPLS to give each customer a private routed network over shared infrastructure.

```
        ┌──────── Provider core (MPLS) ────────┐
 CE1 ── PE1 ── P1 ── P2 ── PE2 ── CE2
        │                            │
   Customer A                    Customer A

CE = Customer Edge
PE = Provider Edge
P  = Provider core router
```

Key concepts:

| Concept | Meaning |
|---|---|
| VRF | Virtual Routing and Forwarding — per-customer routing table |
| RD | Route Distinguisher — makes identical prefixes unique |
| RT | Route Target — controls which VRFs import/export routes |
| MP-BGP | Carries VPN routes between PEs |
| VPN label | Inner label identifying the customer at egress |

```
Customer A and Customer B can both use 10.0.0.0/24 because:
  RD makes the routes unique in BGP:  65000:1:10.0.0.0/24
  RT controls which PEs import them
  the inner VPN label separates traffic in the core
```

```
Packet at PE1:
  push VPN label for Customer A
  push transport label toward PE2
  → two-label stack across the core
```

---

## MPLS Traffic Engineering

Without TE, MPLS follows the IGP shortest path — which can congest one link while another is idle.

With **RSVP-TE**:

```
- Explicit path: choose the route manually or via constraints
- Bandwidth reservation: reserve capacity per LSP
- Fast Reroute: pre-signalled backup path for sub-50 ms recovery
- Affinity: include/exclude links by colour
- Auto-route: route IGP traffic onto TE tunnels
```

```
        ┌──── Link A (congested) ────┐
   PE1 ─┤                            ├─ PE2
        └──── Link B (idle) ─────────┘

RSVP-TE LSP explicitly uses Link B.
```

---

## Segment Routing (Modern Alternative)

**SR-MPLS** replaces per-LSP signalling with **source routing** using **segment IDs (SIDs)**.

```
- The ingress encodes the path as a list of SIDs in the packet
- No LDP/RSVP state in the core (stateless forwarding)
- SIDs are distributed via IGP extensions (OSPF/IS-IS) or BGP
- Supports traffic engineering and TI-LFA fast reroute
```

```
Classic MPLS:  every router keeps per-LSP state
SR-MPLS:       path is in the packet; core routers just forward SIDs
```

---

## MPLS vs IP Routing

| | IP Routing | MPLS |
|---|---|---|
| Forwarding | Longest prefix match | Label lookup |
| Path | Destination-based | Label / explicit path |
| Traffic engineering | Limited (IGP metrics) | Strong (RSVP-TE, SR) |
| VPNs | Overlay (IPsec, GRE) | Native L3VPN/L2VPN |
| State | Per-prefix | Per-LSP (or per-SID) |
| Protocol | OSPF, BGP | LDP, RSVP-TE, BGP, SR |
| Complexity | Lower | Higher |

---

## Pros and Cons

| Pros | Cons |
|---|---|
| Fast, simple forwarding | Operational complexity |
| Traffic engineering | Additional protocols (LDP/RSVP/BGP) |
| VPN services at scale | Label management |
| Transport independence | Troubleshooting is harder |
| Fast reroute | Vendor interoperability quirks |
| Clear separation of customer/provider | Overhead per packet |

---

## Quick Revision

| Concept | Key Point |
|---|---|
| MPLS | Label-based forwarding, Layer 2.5 |
| Label | 20-bit label + TC + S + TTL |
| Label stack | Outer transport, inner VPN |
| Push / swap / pop | Ingress / core / egress operations |
| LSR / LER | Label switch router / edge router |
| LSP | Label Switched Path |
| FEC | Class of packets treated the same |
| LDP | Distributes labels, TCP 646 |
| RSVP-TE | Explicit paths + bandwidth reservation |
| PHP | Penultimate hop pops the label |
| VRF | Per-customer routing table |
| RD / RT | Unique prefix / import-export policy |
| MP-BGP | Carries VPN routes between PEs |
| SR-MPLS | Source routing with segment IDs |

- MPLS forwards on labels, not IP prefixes
- LDP builds LSPs along the IGP shortest path
- RSVP-TE enables traffic engineering and fast reroute
- L3VPN uses VRFs, RDs, RTs, and MP-BGP
- Two labels are typical in VPNs: transport + VPN
- Segment routing is the modern, state-light evolution

---

← [22-multicast-routing](22-multicast-routing.md) • [↑ Computer Networks](README.md) • [24-sdn-and-control-plane](24-sdn-and-control-plane.md) →
