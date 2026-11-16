# Computer Networks

> Notes from internet basics to routing protocols, inter-domain routing, and the modern control plane.

## Reading Order

```
01 → 02 → 03 → 04 → 05 → 06 → 07
                                  ↓
08 → 09 → 10 → 11 → 12 → 13 → 14 → 15
                                  ↓
18 → 19 → 20 → 21 → 22 → 23 → 24 → 25 → 26
```

> 16–17 are reserved for EIGRP and IS-IS notes. Both protocols are compared with the other IGPs in [13-intra-domain-routing](13-intra-domain-routing.md).

---

## Foundations

| # | File | Topics |
|---|---|---|
| 01 | [Switching & Delays](01-switching.md) | Circuit vs packet switching, 4 types of delay, throughput |
| 02 | [OSI Model](02-osi-model.md) | 7 layers, encapsulation, TCP/IP comparison |
| 03 | [Framing](03-framing.md) | Byte count, byte stuffing, bit stuffing, coding violation |
| 04 | [Error Control](04-error-control.md) | Parity, 2D parity, checksum, CRC |
| 05 | [Multiple Access](05-multiple-access.md) | TDMA/FDMA/CDMA, ALOHA, CSMA/CD/CA, hidden/exposed terminal |
| 06 | [Sliding Window](06-sliding-window.md) | Stop-and-wait, Go-Back-N, Selective Repeat |
| 07 | [IPv4 Addressing](07-ipv4-addressing.md) | Dotted decimal, classful, CIDR, subnetting |

## Routing Algorithms

| # | File | Topics |
|---|---|---|
| 08 | [Routing Fundamentals](08-routing-fundamentals.md) | Forwarding vs routing, control/data plane, FIB vs RIB, metrics, AS, IGP vs EGP, convergence |
| 09 | [Distance Vector Routing](09-distance-vector-routing.md) | Bellman-Ford, count-to-infinity, split horizon, poison reverse, hold-down timers |
| 10 | [Link State Routing](10-link-state-routing.md) | LSA flooding, LSDB, Dijkstra/SPF, areas, DV vs LS |
| 11 | [Path Vector Routing](11-path-vector-routing.md) | AS paths, loop prevention, attributes, policy over shortest path |
| 12 | [Routing Algorithms — Comparison](12-routing-algorithms-comparison.md) | Dijkstra vs Bellman-Ford, DV/LS/PV, flooding, broadcast, hierarchical routing |

## Intra-Domain Routing (IGP)

| # | File | Topics |
|---|---|---|
| 13 | [Intra-Domain Routing](13-intra-domain-routing.md) | IGP landscape, RIP vs OSPF vs EIGRP vs IS-IS, choosing an IGP |
| 14 | [RIP](14-rip.md) | RIPv1/v2/RIPng, timers, message format, auth, limitations |
| 15 | [OSPF](15-ospf.md) | Areas, LSA types, DR/BDR, neighbour states, packet types, cost, OSPFv3 |

## Inter-Domain Routing (EGP)

| # | File | Topics |
|---|---|---|
| 18 | [Inter-Domain Routing](18-inter-domain-routing.md) | Autonomous systems, customer/peer/provider, eBGP vs iBGP, IXPs, multihoming |
| 19 | [BGP](19-bgp.md) | Messages, FSM, path attributes, best-path selection, route reflectors, communities |
| 20 | [Routing Policy & Redistribution](20-routing-policy-and-redistribution.md) | Prefix lists, route maps, seed metrics, redistribution loops, BGP policy |

## Advanced Topics

| # | File | Topics |
|---|---|---|
| 21 | [Route Summarization & ECMP](21-route-summarization-and-ecmp.md) | CIDR aggregation, Null0, longest prefix match, ECMP hashing, anycast |
| 22 | [Multicast Routing](22-multicast-routing.md) | IGMP, PIM-DM/SM/SSM, RPF, distribution trees, MBGP |
| 23 | [MPLS & Label Switching](23-mpls-and-label-switching.md) | Labels, LDP, RSVP-TE, L3VPN, segment routing |
| 24 | [SDN & Control Plane](24-sdn-and-control-plane.md) | Control/data plane separation, OpenFlow, controllers, SD-WAN, intent |
| 25 | [IPv6 Routing](25-ipv6-routing.md) | IPv6 addressing, NDP, RIPng, OSPFv3, MP-BGP, transition mechanisms |
| 26 | [Routing Troubleshooting](26-routing-troubleshooting.md) | Methodology, commands, OSPF/BGP failures, loops, blackholes, MTU |
