# RIP — Routing Information Protocol

## What It Is

**RIP** is a distance-vector IGP that uses **hop count** as its metric. It is one of the oldest routing protocols still in use, prized for simplicity and small networks.

```
RIP router:
  "I can reach 10.1.0.0/16 in 3 hops via 192.168.1.2"
```

| Property | Value |
|---|---|
| Type | Distance vector |
| Algorithm | Bellman-Ford |
| Metric | Hop count (1–15, 16 = infinity) |
| Transport | UDP port 520 |
| Update | Every 30 seconds (full table) |
| Max diameter | 15 routers |
| Standard | RFC 1058 (v1), RFC 2453 (v2), RFC 2080 (RIPng) |

---

## How It Works

```
1. Each router starts with connected routes (cost 1).
2. Every 30 s, send the full routing table to neighbours.
3. On receiving an update:
     new cost = received cost + 1 (one more hop)
     if new cost < current cost → update route
4. Triggered updates send a change immediately.
5. Routes not refreshed for 180 s are marked invalid.
6. Routes not refreshed for 240 s are flushed.
```

```
   A ──── B ──── C
   
   B advertises: "A = 1 hop"
   C learns:     "A = 2 hops via B"  (received 1 + 1)
```

Hop count is **not** link speed — a route over a 56 kbps link and one over 100 Gbps both count as one hop.

---

## RIP Versions

| Feature | RIPv1 | RIPv2 | RIPng |
|---|---|---|---|
| IPv4 / IPv6 | IPv4 | IPv4 | IPv6 |
| Classful / classless | Classful | Classless (sends mask) | Classless |
| VLSM / CIDR | No | Yes | Yes |
| Update destination | Broadcast 255.255.255.255 | Multicast 224.0.0.9 | Multicast FF02::9 |
| Authentication | None | Plaintext or MD5 | Uses IPsec |
| Next-hop field | No | Yes | Yes |
| Route tags | No | Yes | Yes |
| Transport | UDP 520 | UDP 520 | UDP 521 |
| Auto-summary | Always on | On by default (can disable) | Not applicable |

```
RIPv1: broadcast, no mask → cannot support VLSM or CIDR
RIPv2: multicast, mask included → supports VLSM, CIDR, auth
RIPng: IPv6-native rewrite
```

---

## Message Format (RIPv2)

```
┌──────────┬──────────┬──────────────────────────┐
│ Command  │ Version  │ Reserved / Routing Domain│
│  (1B)    │  (1B)    │         (2B)             │
├──────────┴──────────┴──────────────────────────┤
│ Route entry 1 (20 bytes)                       │
├────────────────────────────────────────────────┤
│ Route entry 2 (20 bytes)                       │
├────────────────────────────────────────────────┤
│ ... up to 25 entries per message ...           │
└────────────────────────────────────────────────┘
```

Each **route entry**:

```
┌──────────┬──────────┬──────────────┬──────────────┐
│  AFI     │ Route    │  IP address  │  Subnet mask │
│  (2B)    │ tag (2B) │    (4B)      │     (4B)     │
├──────────┴──────────┴──────────────┴──────────────┤
│       Next hop (4B)     │      Metric (4B)        │
└───────────────────────────────────────────────────┘
```

| Field | Meaning |
|---|---|
| Command | 1 = request, 2 = response |
| Version | 1 or 2 |
| AFI | Address Family Identifier (2 = IP) |
| Route tag | Carries external/redistribution info |
| Subnet mask | Prefix length (absent in v1) |
| Next hop | Better next hop to use (0.0.0.0 = use sender) |
| Metric | Hop count (1–16) |

```
Max 25 route entries per RIP message:
  4-byte header + 25 × 20-byte entries = 504 bytes
  (fits comfortably in a 512-byte UDP payload)
```

---

## Timers (Cisco Defaults)

| Timer | Default | Meaning |
|---|---|---|
| Update | 30 s | Send full table to neighbours |
| Invalid | 180 s | No update received → mark route unreachable (metric 16) |
| Hold-down | 180 s | After invalid, ignore worse routes |
| Flush | 240 s | Remove the route entirely |

```
t = 0      route learned
t = 30     updates continue...
t = 180    no update → route marked invalid (16)
t = 180+   hold-down: accept only a better route
t = 240    route flushed from table
```

Timers should match across the network; mismatches cause flapping and slow convergence.

---

## Loop Prevention in RIP

RIP uses the standard distance-vector toolbox:

| Mechanism | How |
|---|---|
| Split horizon | Don't advertise a route back out the interface it was learned from |
| Poison reverse | Advertise it back with metric 16 |
| Route poisoning | Failed route advertised as 16 immediately |
| Triggered updates | Send change immediately, not on the 30 s timer |
| Hold-down | Ignore worse routes during convergence |
| Infinity = 16 | Bounds count-to-infinity |

```
Even with all of these, RIP can still converge slowly on large or
asymmetric topologies. Its 15-hop limit keeps the damage small.
```

---

## Configuration (Cisco IOS)

### Basic RIPv2

```
router rip
 version 2
 no auto-summary
 network 10.0.0.0
 network 192.168.1.0
 passive-interface GigabitEthernet0/1
```

- `network` enables RIP on interfaces in that classful range.
- `passive-interface` stops updates going out an interface but still advertises its network.
- `no auto-summary` is required for discontiguous subnets and CIDR.

### Authentication

```
key chain RIP-KEYS
 key 1
  key-string S3cret

interface GigabitEthernet0/0
 ip rip authentication mode md5
 ip rip authentication key-chain RIP-KEYS
```

### RIPng

```
ipv6 unicast-routing
ipv6 router rip RIPNG

interface GigabitEthernet0/0
 ipv6 rip RIPNG enable
```

---

## Verification Commands

```
show ip protocols
show ip route
show ip route rip
show ip rip database
debug ip rip

show ipv6 route rip
show ipv6 rip
```

```
show ip protocols output tells you:
  - routing protocol and timers
  - networks being advertised
  - passive interfaces
  - neighbours sending updates
  - administrative distance (120)
```

---

## Limitations

| Limitation | Impact |
|---|---|
| Max 15 hops | Cannot route across large networks |
| Hop count metric | Ignores bandwidth, delay, load |
| Slow convergence | Minutes in bad cases |
| Periodic full-table updates | Wastes bandwidth on large tables |
| 25 routes per message | Many packets for large tables |
| Count-to-infinity | Still possible despite mitigations |
| No hierarchy | Flat network only |
| RIPv1 classful | No VLSM/CIDR |
| UDP transport | No built-in reliability (RIP retries at app level) |

---

## When to Use RIP

```
Good:
  - small lab or classroom network
  - legacy devices that only support RIP
  - simple stub network with a default route
  - quick temporary setup

Avoid:
  - production enterprise networks
  - networks larger than 15 hops
  - links with very different bandwidths
  - environments needing fast convergence
```

```
Modern default: OSPF or IS-IS instead of RIP.
RIP survives mainly for simplicity and legacy support.
```

---

## Quick Revision

| Concept | Key Point |
|---|---|
| Type | Distance vector |
| Metric | Hop count |
| Max hops | 15 usable, 16 = infinity |
| Transport | UDP 520 (RIPng 521) |
| Update interval | 30 s |
| Invalid / hold-down / flush | 180 / 180 / 240 s |
| RIPv1 | Classful, broadcast, no auth |
| RIPv2 | Classless, 224.0.0.9, MD5 auth, route tags |
| RIPng | IPv6, FF02::9, IPsec |
| Loop prevention | Split horizon, poison reverse, hold-down, triggered updates |
| Admin distance | 120 |
| Best for | Small/legacy networks |

- RIP is simple, old, and limited to 15 hops
- v2 adds classless routing, multicast, and authentication
- Slow convergence and hop-count metric are its core weaknesses
- Still useful for labs and tiny stub networks

---

← [13-intra-domain-routing](13-intra-domain-routing.md) • [↑ Computer Networks](README.md) • [15-ospf](15-ospf.md) →
