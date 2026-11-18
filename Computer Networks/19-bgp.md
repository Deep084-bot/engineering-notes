# BGP — Border Gateway Protocol

## What It Is

**BGP** is the path-vector protocol that routes traffic **between Autonomous Systems**. It is the routing protocol of the Internet.

```
BGP route:
  prefix:   203.0.113.0/24
  AS path:  65001 65002 65003
  next hop: 192.0.2.1
  attributes: LOCAL_PREF, MED, communities, ORIGIN ...
```

| Property | Value |
|---|---|
| Type | Path vector / EGP |
| Transport | TCP port 179 |
| Current version | BGP-4 (RFC 4271) |
| Metric | Policy + path attributes |
| Loop prevention | AS_PATH |
| Admin distance | eBGP 20, iBGP 200 |
| Default timers | Keepalive 60 s, hold 180 s |

---

## Why BGP Uses TCP

```
✓ Reliable delivery of updates (no need to reinvent retransmission)
✓ Ordered delivery
✓ Incremental updates (only changes, not full table)
✓ Keepalives to detect dead peers
✓ Can run over any IP path, including multi-hop
```

```
BGP session = TCP connection + BGP messages
If TCP breaks → BGP session resets → routes withdrawn
```

---

## BGP Messages

All BGP messages share a 19-byte header:

```
┌────────────────────┬──────────┬──────────┐
│   Marker (16 B)    │ Length   │  Type    │
│   (all ones)       │  (2 B)   │  (1 B)   │
└────────────────────┴──────────┴──────────┘
```

| Type | Name | Purpose |
|---|---|---|
| 1 | OPEN | Start session; exchange ASN, hold time, capabilities |
| 2 | UPDATE | Advertise/withdraw routes and attributes |
| 3 | KEEPALIVE | Keep the session alive (19 bytes, no payload) |
| 4 | NOTIFICATION | Report an error and close the session |
| 5 | ROUTE-REFRESH | Ask a peer to resend routes (capability) |

### OPEN

```
- BGP version (4)
- My AS number
- Hold time (default 180 s)
- BGP identifier (router ID)
- Optional parameters / capabilities
  (route refresh, 4-byte ASN, multiprotocol, graceful restart)
```

### UPDATE

```
- Withdrawn routes (prefixes no longer reachable)
- Path attributes (AS_PATH, NEXT_HOP, LOCAL_PREF, MED, ...)
- NLRI (Network Layer Reachability Information) = new prefixes
```

One UPDATE can advertise multiple prefixes that share the same attributes.

### NOTIFICATION

```
Error codes include:
  Message Header Error
  OPEN Message Error
  UPDATE Message Error
  Hold Timer Expired
  FSM Error
  Cease
```

---

## BGP Finite State Machine

```
Idle
  │  start / retry timer
  ▼
Connect ──TCP fails──► Active
  │                      │
  │ TCP established      │ TCP established
  ▼                      ▼
OpenSent ◄───────────────┘
  │  OPEN received and valid
  ▼
OpenConfirm
  │  KEEPALIVE received
  ▼
Established  ← routes can be exchanged
```

| State | Meaning |
|---|---|
| Idle | No session; waiting to start |
| Connect | TCP connection in progress |
| Active | TCP failed; retrying (despite the name, not connected) |
| OpenSent | OPEN sent, waiting for peer's OPEN |
| OpenConfirm | OPEN accepted, waiting for KEEPALIVE |
| Established | Session up; UPDATEs flowing |

```
"Active" is usually a bad sign in troubleshooting —
it means the router is trying and failing to establish TCP.
```

---

## Sessions: eBGP and iBGP

```
        AS 65001                     AS 65002
   R1 ──iBGP── R2 ────eBGP──── R3 ──iBGP── R4
```

### eBGP

```
- Peers in different ASes
- Prepends own AS to AS_PATH
- Default TTL 1 → directly connected (or multihop config)
- Rewrites NEXT_HOP to its own address
- Admin distance 20
```

### iBGP

```
- Peers in the same AS
- Does NOT prepend its own AS
- NEXT_HOP unchanged by default
- iBGP split horizon: routes learned from one iBGP peer
  are not advertised to another iBGP peer
- Admin distance 200
- Needs full mesh, route reflectors, or confederations
```

### Why iBGP next hop is unchanged

```
R2 learns 203.0.113.0/24 from eBGP peer R3 with NEXT_HOP 192.0.2.1.
R2 advertises it to iBGP peer R1 with the SAME next hop 192.0.2.1.
R1 must reach 192.0.2.1 through the IGP — not through BGP.
```

If the next hop is unreachable via the IGP, the BGP route is not usable.

```
Fix on the border router:
  neighbor <iBGP-peer> next-hop-self
```

---

## Route Advertisement

BGP only advertises routes that are:

1. In the BGP table (learned or originated)
2. Selected as best
3. Permitted by outbound policy

```
Originating a prefix:
  network 203.0.113.0 mask 255.255.255.0

The prefix must exist in the routing table (connected/static/IGP),
otherwise the network statement does nothing.
```

```
Withdrawing:
  Remove the route, or apply an outbound filter.
```

---

## Path Attributes

Attributes describe a path and drive best-path selection.

### Well-known mandatory

| Attribute | Meaning |
|---|---|
| `ORIGIN` | How the route entered BGP: IGP (0), EGP (1), INCOMPLETE (2) |
| `AS_PATH` | ASes traversed, most recent first |
| `NEXT_HOP` | IP to use to reach the prefix |

### Well-known discretionary

| Attribute | Meaning |
|---|---|
| `LOCAL_PREF` | Preference inside the local AS (higher wins) |
| `ATOMIC_AGGREGATE` | Route was aggregated; do not de-aggregate |

### Optional transitive

| Attribute | Meaning |
|---|---|
| `AGGREGATOR` | Router/AS that performed aggregation |
| `COMMUNITIES` | Tags for policy (e.g., no-export) |

### Optional non-transitive

| Attribute | Meaning |
|---|---|
| `MED` | Multi-exit discriminator; hint to a neighbouring AS |
| `ORIGINATOR_ID` | Loop prevention for route reflectors |
| `CLUSTER_LIST` | Loop prevention for route reflectors |

```
Transitive attributes are passed on to other ASes.
Non-transitive attributes are not propagated beyond the next AS.
```

---

## AS_PATH and Loop Prevention

```
AS 65001 advertises 203.0.113.0/24 to AS 65002:
  AS_PATH = 65001

AS 65002 advertises it to AS 65003:
  AS_PATH = 65002 65001

If AS 65001 ever receives a route containing 65001 in AS_PATH:
  → loop detected → discard
```

```
AS_PATH is also used for:
  - path length comparison (shorter usually preferred)
  - inbound traffic engineering (prepending makes a path look longer)
```

**AS_PATH prepending**

```
Want traffic to prefer ISP A over ISP B for inbound:
  advertise to ISP B with AS_PATH = 65001 65001 65001
  → ISP B's path looks longer to the rest of the Internet
  → traffic prefers ISP A
```

---

## BGP Best-Path Selection

BGP evaluates routes in a fixed order. The first attribute that differs decides the winner.

### Cisco order

```
1. Highest WEIGHT                          (Cisco proprietary, local)
2. Highest LOCAL_PREF
3. Locally originated (network/aggregate/redistribute/static)
4. Shortest AS_PATH
5. Lowest ORIGIN (IGP < EGP < INCOMPLETE)
6. Lowest MED
7. eBGP over iBGP
8. Lowest IGP metric to the next hop
9. (If multipath enabled, install multiple equal paths)
10. Oldest route (for eBGP)
11. Lowest router ID
12. Lowest neighbour IP address
```

### Standard (RFC) order

```
1. Highest LOCAL_PREF
2. Locally originated
3. Shortest AS_PATH
4. Lowest ORIGIN
5. Lowest MED
6. eBGP over iBGP
7. Lowest IGP metric to next hop
8. Oldest route
9. Lowest router ID
10. Lowest neighbour address
```

```
WEIGHT is Cisco-specific and only matters locally.
LOCAL_PREF is the main outbound policy knob.
AS_PATH length is the main global preference signal.
MED is a hint to the neighbouring AS, not a local decision.
```

---

## Route Reflectors and Confederations

### Route reflector (RR)

```
Without RR: every iBGP router peers with every other (full mesh)

With RR:
        ┌──────── RR ────────┐
        │                    │
     client              client
        │                    │
     client              client

RR reflects routes from one client to other clients.
```

Rules:

```
- Route learned from a client → reflect to all other clients and non-clients
- Route learned from a non-client → reflect to clients only
- Route learned from an eBGP peer → reflect normally
```

Loop prevention:

```
ORIGINATOR_ID = router ID of the original advertiser
CLUSTER_LIST  = list of cluster IDs traversed
If a router sees its own ID/cluster in these → discard
```

### Confederation

```
AS 65000 split into sub-ASes:
  65001, 65002, 65003

Inside the confederation: eBGP-like sessions between sub-ASes
Outside: the whole confederation looks like AS 65000
```

Benefits: reduces full-mesh requirements, hides internal structure.
Drawbacks: complexity, policy interactions, less common today than RR.

---

## Communities

A **community** is a tag attached to a route to simplify policy.

```
Format: ASN:value   (e.g., 65001:100)

Well-known communities:
  NO_EXPORT            0xFFFFFF01  → don't advertise outside the AS
  NO_ADVERTISE         0xFFFFFF02  → don't advertise to any peer
  NO_EXPORT_SUBCONFED  0xFFFFFF03  → don't advertise outside sub-AS
  INTERNET             0x00000000  → advertise to everyone
```

```
Example policy:
  Tag routes learned from customers with 65001:100
  Outbound to peers: advertise only routes tagged 65001:100

Result: peers get customer routes, not transit routes.
```

---

## Timers and Stability

| Timer | Default | Meaning |
|---|---|---|
| Keepalive | 60 s | Send KEEPALIVE every 60 s |
| Hold time | 180 s | Declare peer dead after 180 s without messages |
| Connect retry | 120 s | Retry TCP connection |
| Keepalive/hold ratio | 1:3 | Hold time is 3× keepalive |

```
Hold time is negotiated: the lower of the two peers' values is used.
Keepalive should be ≤ 1/3 of hold time.
```

### Route dampening

```
Repeatedly flapping routes can be penalised:
  - each flap adds a penalty
  - penalty decays over time
  - above a threshold → route suppressed
  - below reuse threshold → route restored
```

This protects the global table from unstable prefixes. Modern practice often relies on max-prefix limits and prefix filters instead.

---

## Configuration (Cisco IOS)

### eBGP session

```
router bgp 65001
 bgp router-id 1.1.1.1
 neighbor 192.0.2.2 remote-as 65002
 neighbor 192.0.2.2 description ISP-A

 network 203.0.113.0 mask 255.255.255.0
```

### iBGP session

```
router bgp 65001
 neighbor 10.0.0.3 remote-as 65001
 neighbor 10.0.0.3 update-source Loopback0
 neighbor 10.0.0.3 next-hop-self
```

### eBGP multihop (loopback peering)

```
router bgp 65001
 neighbor 192.0.2.2 remote-as 65002
 neighbor 192.0.2.2 ebgp-multihop 2
 neighbor 192.0.2.2 update-source Loopback0
```

### Filtering and policy

```
ip prefix-list ALLOW-ISP permit 203.0.113.0/24
route-map FROM-ISP permit 10
 match ip address prefix-list ALLOW-ISP
 set local-preference 200

router bgp 65001
 neighbor 192.0.2.2 route-map FROM-ISP in
 neighbor 192.0.2.2 maximum-prefix 1000
```

### IPv6 / multiprotocol

```
router bgp 65001
 address-family ipv6 unicast
  neighbor 2001:db8::2 activate
  network 2001:db8:100::/48
```

---

## Verification Commands

```
show ip bgp summary
show ip bgp
show ip bgp neighbors
show ip bgp neighbors 192.0.2.2 advertised-routes
show ip bgp neighbors 192.0.2.2 routes
show ip bgp 203.0.113.0/24
show ip route bgp
show ip bgp community
```

```
show ip bgp summary tells you:
  - neighbour state (Established?)
  - prefixes received
  - uptime
  - messages sent/received
```

```
show ip bgp 203.0.113.0/24 shows:
  - all paths
  - the best path (marked >)
  - AS_PATH, next hop, LOCAL_PREF, MED, origin
```

---

## Common Problems

| Symptom | Likely cause |
|---|---|
| Session stuck in Active | TCP 179 blocked, wrong IP, no route to peer |
| Session idle | No neighbour config, wrong AS |
| Session Established, no routes | Inbound filter, next hop unreachable |
| Route not advertised | Outbound filter, network statement missing, not best path |
| Flapping | Unstable link, hold timer expiry, max-prefix |
| Suboptimal inbound | AS_PATH prepending / MED not tuned |
| Suboptimal outbound | LOCAL_PREF not tuned |

```
BGP troubleshooting order:
  TCP reachability → session state → prefixes received →
  best-path selection → filters → next-hop reachability
```

---

## Quick Revision

| Concept | Key Point |
|---|---|
| Type | Path vector EGP |
| Transport | TCP 179 |
| Messages | OPEN, UPDATE, KEEPALIVE, NOTIFICATION, ROUTE-REFRESH |
| States | Idle → Connect → OpenSent → OpenConfirm → Established |
| eBGP | Between ASes, AS_PATH prepend, TTL 1, AD 20 |
| iBGP | Within AS, no prepend, full mesh/RR, AD 200 |
| Loop prevention | AS_PATH (eBGP), ORIGINATOR_ID/CLUSTER_LIST (RR) |
| LOCAL_PREF | Outbound path choice (higher wins) |
| MED | Inbound hint to neighbour AS (lower preferred) |
| AS_PATH prepend | Influence inbound traffic |
| Communities | Tag routes for policy |
| Route reflector | Scales iBGP beyond full mesh |
| Best path | Weight → LOCAL_PREF → AS_PATH → ORIGIN → MED → eBGP → IGP metric |

- BGP is policy-first, shortest-path-second
- eBGP exchanges routes between ASes; iBGP distributes them internally
- iBGP next hop must be reachable via the IGP
- AS_PATH both prevents loops and influences inbound traffic
- LOCAL_PREF decides outbound; MED/prepending influence inbound
- Route reflectors remove the full-mesh requirement

---

← [18-inter-domain-routing](18-inter-domain-routing.md) • [↑ Computer Networks](README.md) • [20-routing-policy-and-redistribution](20-routing-policy-and-redistribution.md) →
