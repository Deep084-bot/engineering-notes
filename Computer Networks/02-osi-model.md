# OSI Model

## What It Is

The **OSI (Open Systems Interconnection) model** is a 7-layer reference model that describes how data flows from one application to another across a network.

```
Layer 7: Application      ← your app (HTTP, FTP, SMTP)
Layer 6: Presentation     ← data format, encryption
Layer 5: Session           ← connection management
Layer 4: Transport         ← end-to-end delivery (TCP/UDP)
Layer 3: Network           ← routing, IP addressing
Layer 2: Data Link         ← framing, MAC addressing
Layer 1: Physical          ← bits on the wire
```

---

## Data Flow: Encapsulation & Decapsulation

```
Sending side (top → bottom):
  Data → +Header → +Header → ... → bits on wire

Each layer adds its own header (encapsulation)
Receiving side (bottom → top):
  bits → strip header → strip header → ... → Data (decapsulation)
```

```
Application:  [        DATA        ]
Transport:    [TCP/UDP][        DATA        ]
Network:      [IP][TCP/UDP][        DATA        ]
Data Link:    [Frame H][IP][TCP/UDP][        DATA        ][Frame T]
Physical:     011010010110...
```

---

## Layer-by-Layer Breakdown

### Layer 1 — Physical

Transmits raw **bits** over a physical medium.

| Function | Detail |
|---|---|
| What it handles | Bits (0s and 1s) |
| Medium | Copper, fiber, wireless |
| Concerns | Voltage, timing, cable specs |
| Devices | Hubs, repeaters, cables |

```
Example: Ethernet cable, radio waves, fiber optics
Doesn't care what the bits mean — just pushes them through.
```

---

### Layer 2 — Data Link

Transfers frames between **directly connected** nodes (hop-to-hop).

| Function | Detail |
|---|---|
| What it handles | Frames (packets with header + trailer) |
| Addressing | MAC addresses (48-bit, e.g., `AA:BB:CC:DD:EE:FF`) |
| Error detection | CRC, parity (in the frame trailer) |
| Flow control | Prevents overwhelming the receiver |
| Medium access | Who gets to use the shared link (MAC protocols) |

```
Frame structure:
  ┌─────────┬──────────┬──────────────┬───────┐
  │ Dest MAC │ Src MAC  │   Payload    │  CRC  │
  │  (6B)    │  (6B)    │  (variable)  │ (4B)  │
  └─────────┴──────────┴──────────────┴───────┘
```

**Sub-layers:**
- **LLC (Logical Link Control):** interfaces with the network layer
- **MAC (Media Access Control):** controls access to the physical medium

---

### Layer 3 — Network

Routes packets across **multiple networks** (end-to-end path, hop-by-hop forwarding).

| Function | Detail |
|---|---|
| What it handles | Packets (datagrams) |
| Addressing | IP addresses (32-bit for IPv4, 128-bit for IPv6) |
| Routing | Determines the best path through the network |
| Fragmentation | Breaks large packets to fit link MTU |
| Devices | Routers |

```
IP Header (simplified):
  ┌──────┬───────────────────────┐
  │ Ver  │     Source IP         │
  │ TTL  │    Destination IP     │
  └──────┴───────────────────────┘
```

---

### Layer 4 — Transport

Provides **end-to-end** (process-to-process) delivery and reliability.

| Function | Detail |
|---|---|
| What it handles | Segments (TCP) / Datagrams (UDP) |
| Addressing | Port numbers (16-bit, 0–65535) |
| Reliability | TCP: retransmission, ordering, flow control |
| Multiplexing | Multiple apps share one IP via ports |

**TCP vs UDP:**

| | TCP | UDP |
|---|---|---|
| Connection | Connection-oriented | Connectionless |
| Reliability | Guaranteed delivery | Best-effort |
| Ordering | Ordered | Unordered |
| Speed | Slower (overhead) | Faster (no overhead) |
| Use | Web, email, file transfer | DNS, streaming, gaming |

```
TCP port 80  = HTTP
TCP port 443 = HTTPS
UDP port 53  = DNS
```

---

### Layer 5 — Session

Manages **sessions** (dialogues) between applications.

| Function | Detail |
|---|---|
| Session establishment | Create, maintain, tear down |
| Synchronization | Checkpoints in long transfers |
| Dialog control | Who speaks when (full-duplex, half-duplex) |

```
Example: A video call = a session
  Establish → maintain → handle reconnection → teardown
```

*In practice, this layer is often merged into the application or transport layer (TCP handles sessions in the TCP/IP model).*

---

### Layer 6 — Presentation

Handles **data representation** and transformation.

| Function | Detail |
|---|---|
| Data formatting | ASCII, Unicode, EBCDIC |
| Encryption/Decryption | SSL/TLS (often here conceptually) |
| Compression | Reduce data size |
| Translation | Convert between formats |

```
Example: converting between big-endian and little-endian
        encrypting data before sending
        JPEG/PNG for images
```

*Also often merged into the application layer in practice.*

---

### Layer 7 — Application

The **user-facing** layer — interfaces with the application.

| Function | Detail |
|---|---|
| Protocols | HTTP, FTP, SMTP, DNS, SSH, Telnet |
| Services | File transfer, email, web browsing |
| User interaction | The app you actually use |

```
HTTP (port 80/443)  = web browsing
FTP (port 20/21)    = file transfer
SMTP (port 25)      = sending email
DNS (port 53)       = domain name resolution
SSH (port 22)       = secure remote login
```

---

## The 7 Layers — Quick Reference

```
7  Application    ─── HTTP, FTP, SMTP, DNS       ─── Data
6  Presentation   ─── Encryption, compression     ─── Data
5  Session        ─── Session management           ─── Data
4  Transport      ─── TCP/UDP, port numbers        ─── Segments
3  Network        ─── IP, routing, fragmentation  ─── Packets
2  Data Link      ─── MAC, frames, CRC            ─── Frames
1  Physical       ─── Bits, cables, signals        ─── Bits
```

**Mnemonic (bottom to top):** "Please Do Not Throw Sausage Pizza Away"
(Physical, Data Link, Network, Transport, Session, Presentation, Application)

**Mnemonic (top to bottom):** "All People Seem To Need Data Processing"

---

## OSI vs TCP/IP Model

| OSI (7 layers) | TCP/IP (4 layers) |
|---|---|
| Application | Application |
| Presentation | Application |
| Session | Application |
| Transport | Transport |
| Network | Internet |
| Data Link | Network Access |
| Physical | Network Access |

The internet uses TCP/IP, not OSI. OSI is a reference model — useful for understanding concepts, but TCP/IP is what's actually deployed.

---

## Quick Revision

| Layer | Function | Address | PDU | Device |
|---|---|---|---|---|
| 7 Application | User services | — | Data | — |
| 6 Presentation | Format, encrypt | — | Data | — |
| 5 Session | Dialog control | — | Data | — |
| 4 Transport | End-to-end delivery | Port # | Segment | — |
| 3 Network | Routing | IP Address | Packet | Router |
| 2 Data Link | Framing, error detect | MAC Address | Frame | Switch |
| 1 Physical | Bits on wire | — | Bit | Hub |

---

← [01-switching](01-switching.md) • [↑ Computer Networks](README.md) • [03-framing](03-framing.md) →
