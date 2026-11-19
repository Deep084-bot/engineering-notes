# Routing Policy and Redistribution

## What It Is

**Routing policy** controls which routes are accepted, preferred, modified, and advertised. **Redistribution** moves routes from one routing protocol into another.

```
Routing policy answers:
  - Which routes should I trust?
  - Which routes should I prefer?
  - Which routes should I advertise, and to whom?
  - With what attributes?
```

```
Redistribution answers:
  - How do OSPF routes get into BGP?
  - How do RIP routes get into OSPF?
  - What metric/attributes should they carry?
```

---

## Why Policy Matters

```
Without policy:
  - Every router trusts every route
  - Transit traffic may be carried for free
  - A misconfigured router can blackhole the network
  - Migrations and mergers cause routing loops

With policy:
  - Prefixes are filtered at boundaries
  - Business relationships are enforced
  - Traffic follows engineered paths
  - Protocol migrations are controlled
```

---

## Policy Tools

| Tool | Purpose |
|---|---|
| Access list (ACL) | Match addresses/prefixes (older, less precise) |
| Prefix list | Match prefixes and prefix lengths efficiently |
| Distribute list | Apply a filter to routing updates |
| Route map | Match + set attributes, sequenced rules |
| Offset list | Increase a distance-vector metric |
| Route tag | Mark routes to prevent loops / identify origin |
| Administrative distance | Choose between protocols |
| BGP attributes | LOCAL_PREF, MED, AS_PATH, communities, weight |

---

## Prefix Lists

A prefix list matches a **prefix and its length**. It is the preferred tool for route filtering.

```
ip prefix-list PL-EXAMPLE permit 10.0.0.0/8 le 24
```

```
10.0.0.0/8 le 24
  → matches 10.0.0.0/8 through 10.255.255.0/24
  → any prefix whose first 8 bits are 10 and length ≤ 24

ip prefix-list PL permit 10.1.0.0/16 ge 17 le 24
  → matches subnets of 10.1.0.0/16 with length 17–24
  → does NOT match 10.1.0.0/16 itself
```

| Keyword | Meaning |
|---|---|
| `ge` | Greater than or equal to this length |
| `le` | Less than or equal to this length |
| no `ge`/`le` | Exact prefix length only |
| `ge` and `le` together | Range of lengths |

```
Exact:      10.1.0.0/16
Range:      10.1.0.0/16 ge 17 le 24
Default:    0.0.0.0/0 le 32   (matches everything)
```

Prefix lists are faster and safer than ACLs because they match on prefix length, not just address bits.

---

## Route Maps

A route map is an ordered list of rules. Each rule has a sequence number, a permit/deny action, `match` conditions, and `set` actions.

```
route-map FROM-CUSTOMER permit 10
 match ip address prefix-list CUSTOMER-PREFIXES
 set local-preference 200
 set community 65001:100

route-map FROM-CUSTOMER deny 20
 match ip address prefix-list BOGONS
```

Rules:

```
- Evaluated in sequence order (10, 20, 30, ...)
- First match wins
- If a rule matches but is deny → route is denied, stop
- If a rule matches and permits → apply set actions, stop
- If no rule matches → implicit deny at the end
- An empty route-map with only "permit 10" matches everything
```

```
match conditions:  ip address, prefix-list, community, as-path,
                   metric, tag, interface, next-hop
set actions:       metric, tag, local-preference, med, weight,
                   community, as-path prepend, next-hop
```

---

## Distribute Lists

A distribute list applies an ACL or prefix list to routing updates.

```
router ospf 1
 distribute-list prefix-list ALLOWED in

router rip
 distribute-list prefix-list ALLOWED out GigabitEthernet0/0
```

```
in  → filter routes being received
out → filter routes being advertised
```

```
Distribute lists filter ROUTES, not DATA.
They do not block ping or user traffic — only routing information.
```

---

## Offset Lists

An offset list increases the metric of matching routes — useful for traffic engineering in distance-vector protocols.

```
router rip
 offset-list 1 in 2 GigabitEthernet0/0

access-list 1 permit 10.2.0.0 0.0.255.255
```

```
Adds 2 hops to matching routes learned on that interface,
making them less preferred.
```

---

## Route Tags

A tag is a numeric label attached to a route. It survives redistribution and can be matched later.

```
route-map TAG-OSPF permit 10
 set tag 100

route-map DENY-TAGGED deny 10
 match tag 100
```

```
Use cases:
  - prevent a route from being redistributed back into its source protocol
  - identify routes learned from a specific domain
  - drive policy without relying on prefix lists
```

---

## Administrative Distance as Policy

You can force a router to prefer one protocol over another for the same prefix by changing AD.

```
ip route 0.0.0.0 0.0.0.0 192.0.2.1 250
```

```
Lower AD wins.
Useful for:
  - floating static routes (backup path)
  - preferring one routing domain over another
Risk:
  - inconsistent AD across routers → loops and blackholes
```

```
Floating static:
  Primary: dynamic route (OSPF, AD 110)
  Backup:  static route with AD 130
  → static is used only if the OSPF route disappears
```

---

## Redistribution

**Redistribution** takes routes from one source and injects them into another routing protocol.

```
        OSPF domain                 EIGRP domain
   ┌──────────────────┐       ┌──────────────────┐
   │   R1 ── R2       │       │   R4 ── R5       │
   │         │        │       │         │        │
   │        R3 ───────┼───────┼──────── R6       │
   └──────────────────┘   ↑   └──────────────────┘
                     redistribution point

R3 learns OSPF routes and injects them into EIGRP.
R3 learns EIGRP routes and injects them into OSPF.
```

### Why redistribution is tricky

```
Each protocol has a different metric:
  RIP     = hop count
  OSPF    = cost
  EIGRP   = composite bandwidth/delay
  BGP     = policy attributes

A route cannot carry an OSPF cost into RIP directly.
You must set a SEED METRIC.
```

### Seed metrics

| Into | Default seed | Notes |
|---|---|---|
| RIP | Infinity (unusable) | Must set a metric explicitly |
| OSPF | 20 | External metric; type E2 by default |
| EIGRP | Infinity | Must set bandwidth/delay/etc. |
| BGP | MED 0, origin incomplete | Route must exist in the table |

```
router ospf 1
 redistribute eigrp 10 subnets metric 100 metric-type 1

router eigrp 10
 redistribute ospf 1 metric 10000 100 255 1 1500

router rip
 redistribute ospf 1 metric 3
```

### OSPF external metric types

```
E1: external cost + internal cost to the ASBR
E2: external cost only (default)
    → E2 ignores the internal distance to the ASBR
```

### BGP redistribution

```
BGP does not automatically redistribute IGP routes.
You can either:
  - use `network` for specific prefixes (preferred, explicit)
  - use `redistribute` to inject a class of routes

network 203.0.113.0 mask 255.255.255.0
redistribute connected
redistribute static
```

```
`network` requires the prefix to exist in the routing table.
`redistribute` injects routes from the protocol directly.
Prefer `network` for control and predictability.
```

---

## Redistribution Loops

Mutual redistribution can feed routes back and forth.

```
OSPF ──► EIGRP ──► OSPF ──► EIGRP ...

A route redistributed from OSPF into EIGRP can be redistributed
back into OSPF with a different metric — sometimes preferred over
the original.
```

Fixes:

```
1. Route tags
   - tag routes on the way out
   - deny tagged routes on the way back in

2. Distribute lists / prefix lists
   - explicitly allow only the intended prefixes

3. One-way redistribution
   - redistribute only in one direction, use defaults the other way

4. Administrative distance tuning
   - make the redistributed copy less preferred

5. Filter at the boundary
   - never redistribute the same prefix in both directions
```

### One-way redistribution

```
        OSPF ──────────────► EIGRP
              (redistribute OSPF into EIGRP)

        EIGRP ◄────────────── OSPF
              (do NOT redistribute EIGRP into OSPF)
              Instead, inject a default route into OSPF
```

This is often cleaner for stub domains.

---

## Policy in BGP

BGP policy is the most important policy in real networks.

### Inbound policy (what I accept and how I prefer it)

```
route-map FROM-ISP permit 10
 match ip address prefix-list ISP-ROUTES
 set local-preference 150
 set community 65001:200

neighbor 192.0.2.2 route-map FROM-ISP in
```

### Outbound policy (what I advertise)

```
ip prefix-list MY-PREFIXES permit 203.0.113.0/24

route-map TO-ISP permit 10
 match ip address prefix-list MY-PREFIXES

neighbor 192.0.2.2 route-map TO-ISP out
```

### Common BGP policy actions

| Goal | Tool |
|---|---|
| Prefer one upstream for outbound | `LOCAL_PREF` |
| Influence inbound path | `AS_PATH` prepend, `MED` |
| Prevent transit | Only advertise customer prefixes |
| Tag route origin | Communities |
| Limit prefix count | `maximum-prefix` |
| Block bogons/martians | Prefix filters |
| Prevent loops | AS_PATH, route tags |

---

## BGP Prefix Filtering Best Practices

```
Always filter:
  - your own prefixes (never accept them from outside)
  - bogon prefixes (private, reserved, unallocated)
  - too-specific prefixes (e.g., /25+ in the global table)
  - default route from peers unless intended

Use:
  - prefix lists with explicit ge/le bounds
  - maximum-prefix limits per neighbour
  - RPKI / IRR data for origin validation where possible
```

```
RFC 7454 (BGP operations and security) recommends:
  - explicit inbound/outbound filters
  - max-prefix limits
  - bogon filtering
  - no next-hop-self on eBGP
  - TTL security (GTSM) where supported
```

---

## Verification Commands

```
show ip protocols
show ip route
show ip route <prefix>
show route-map
show ip prefix-list
show ip bgp
show ip bgp neighbors <ip> routes
show ip bgp neighbors <ip> advertised-routes
show ip ospf database external
show ip eigrp topology
```

---

## Quick Revision

| Concept | Key Point |
|---|---|
| Routing policy | Control accept/prefer/advertise |
| Prefix list | Matches prefix + length (`ge`/`le`) |
| Route map | Sequenced match/set rules; implicit deny |
| Distribute list | Filters routing updates in/out |
| Offset list | Increases DV metric |
| Route tag | Marks routes across redistribution |
| Administrative distance | Chooses between protocols |
| Redistribution | Inject routes from one protocol into another |
| Seed metric | Required when metrics are incompatible |
| OSPF E1/E2 | External cost + internal vs external only |
| Mutual redistribution | Can loop; use tags/filters/one-way |
| BGP `network` | Originates a prefix already in the table |
| BGP `redistribute` | Injects a class of routes |
| LOCAL_PREF | Outbound BGP preference |
| AS_PATH prepend / MED | Inbound BGP influence |

- Policy = trust boundaries, business relationships, traffic engineering
- Prefix lists are better than ACLs for route filtering
- Redistribution needs seed metrics and loop protection
- Route tags + filters are the standard anti-loop tools
- BGP policy is applied inbound and outbound per neighbour
- Always filter your own prefixes, bogons, and overly specific routes

---

← [19-bgp](19-bgp.md) • [↑ Computer Networks](README.md) • [21-route-summarization-and-ecmp](21-route-summarization-and-ecmp.md) →
