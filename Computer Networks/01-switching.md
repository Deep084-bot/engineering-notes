# Switching, Internet & Performance Metrics

## Overview of the Internet

The internet is a global network of interconnected networks. At its core:

```
End systems (hosts) = computers, phones, servers
Links              = wired (fiber, copper) or wireless
Routers            = forward packets between networks
Protocols           = rules governing communication (TCP, IP, HTTP)
```

**Three things to know:**
- Internet = network of networks (ISPs connect to each other)
- Communication is packet-based (not circuit-based)
- End systems talk via protocols (TCP/IP is the standard)

---

## Circuit Switching vs Packet Switching

### Circuit Switching

A **dedicated end-to-end path** is established before communication begins.

```
Phone call analogy:
  A───────────dedicated path───────────B
  Path is reserved for entire duration
  Even when nobody is talking, the path is held
```

```
Phases:  Setup → Data Transfer → Teardown
         (reserve)  (use it)     (release)
```

| Pros | Cons |
|---|---|
| Guaranteed bandwidth | Wasteful (path idle when not transmitting) |
| Low latency during transfer | Setup delay before data |
| Predictable performance | Doesn't scale well |

**Used in:** traditional telephone networks (PSTN), some VPNs.

### Packet Switching

Data is broken into **packets**, each sent independently through the network. No dedicated path — packets share links with other traffic.

```
A sends 3 packets → Router picks best path for each
  Packet 1 → R1 → R3 → B
  Packet 2 → R1 → R2 → R3 → B  (different path possible)
  Packet 3 → R2 → R3 → B
```

```
No setup phase. Packets are sent on demand.
```

| Pros | Cons |
|---|---|
| Efficient (link shared, no idle reservation) | Variable delay (congestion, queuing) |
| No setup needed | Packets may arrive out of order |
| Fault-tolerant (packets take different paths) | Requires protocols to handle reordering |

**Used in:** the internet, most modern networks.

### Comparison

| | Circuit Switching | Packet Switching |
|---|---|---|
| Path | Dedicated, fixed | Dynamic, per-packet |
| Setup | Required | Not required |
| Resource usage | Reserved (wasted when idle) | Shared (statistical multiplexing) |
| Delay | Predictable | Variable |
| Scalability | Poor | Good |
| Example | Phone call | Internet browsing |

**Statistical multiplexing:** packet switching's key advantage — links are shared among many users, so bandwidth isn't wasted when one user is idle.

---

## Network Performance Metrics

### Types of Delay

Total delay for a packet = sum of four components:

```
d_total = d_proc + d_queue + d_trans + d_prop
```

#### 1. Processing Delay (d_proc)

Time for the router to **examine the packet header** and determine where to forward it.

```
Router receives packet → checks header → looks up routing table → forwards

Typically very small (microseconds) — modern routers are fast.
```

#### 2. Queuing Delay (d_queue)

Time the packet **waits in a buffer** before being transmitted.

```
If the link is busy, the packet queues up:
  Packet arrives → buffer → waits → finally transmitted

This is the most VARIABLE delay:
  - Light traffic: near zero
  - Heavy traffic: can be milliseconds to seconds
  - Buffer overflow → packet DROP
```

**Queuing delay depends on:** traffic intensity, number of packets ahead, link speed.

#### 3. Transmission Delay (d_trans)

Time to **push all bits of the packet onto the link**.

```
d_trans = L / R

L = packet length (bits)
R = link bandwidth (bits/sec)

Example: 10,000-bit packet on a 1 Mbps link
  d_trans = 10,000 / 1,000,000 = 0.01 seconds = 10 ms
```

**Depends on:** packet size and link speed.

#### 4. Propagation Delay (d_prop)

Time for a bit to **travel physically** from one end of the link to the other.

```
d_prop = D / S

D = distance of the link (meters)
S = propagation speed of the medium (≈ 2×10⁸ m/s for copper/wire)

Example: 1000 km link
  d_prop = 1,000,000 / 2×10⁸ = 0.005 seconds = 5 ms
```

**Depends on:** distance and medium speed. NOT on packet size.

### Summary Table

| Delay Type | Depends On | Typical |
|---|---|---|
| Processing | Router speed | Microseconds |
| Queuing | Traffic load | 0 → seconds (variable) |
| Transmission | Packet size, link speed | Milliseconds |
| Propagation | Distance, medium speed | Microseconds–milliseconds |

```
Analogy — mailing a letter:
  Processing  = post office sorting
  Queuing     = waiting at the post office
  Transmission = time for the postman to physically place it on the truck
  Propagation = time the truck drives to the destination
```

### Throughput vs Bandwidth

| Metric | Definition |
|---|---|
| **Bandwidth** | Maximum rate a link can carry (bits/sec) |
| **Throughput** | Actual rate achieved (bits/sec) |

```
Link bandwidth: 100 Mbps
Actual throughput: 80 Mbps  (due to congestion, protocol overhead)

Throughput ≤ Bandwidth always.
```

### Packet Loss

```
When a router's buffer is full:
  New packet arrives → no room → DROPPED

Packet loss rate = packets lost / total packets sent
  Typically < 1% on good networks
  Higher under congestion
```

---

## Quick Revision

| Concept | Key Point |
|---|---|
| Circuit switching | Dedicated path, reserved resources, phone networks |
| Packet switching | Packets share links, statistical multiplexing, internet |
| Processing delay | Header examination (router) |
| Queuing delay | Waiting in buffer (variable, worst under load) |
| Transmission delay | L/R (packet size ÷ link speed) |
| Propagation delay | D/S (distance ÷ speed of light in medium) |
| Bandwidth | Maximum link capacity |
| Throughput | Actual achieved rate |
| Packet loss | Buffer overflow → dropped packets |

---

← [↑ Computer Networks](README.md) • [02-osi-model](02-osi-model.md) →
