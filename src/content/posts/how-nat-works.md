---
author: Vaibhav Jha
pubDatetime: 2026-10-15T10:00:00Z
title: "How NAT Works: The Invisible Gateway Between You and the Internet"
slug: how-nat-works
featured: true
draft: false
tags:
  - networking
  - infrastructure
  - explainer
description: "Your home network has one public IP but dozens of devices. How does NAT juggle all those connections? Let's break it down from first principles."
---

Right now, your laptop, your phone, your smart TV, and maybe a dozen other gadgets are all happily browsing the internet. But your ISP gave you exactly **one** public IP address. How is that even possible?

The answer is **NAT — Network Address Translation** — one of the most important (and most invisible) pieces of internet infrastructure. It's the reason the IPv4 internet didn't collapse in the early 2000s, and it's the reason peer-to-peer connections are so annoyingly difficult.

Let's tear it apart.

## Table of contents

## The IPv4 address exhaustion problem

IPv4 addresses are 32 bits long. That gives us roughly **4.3 billion** unique addresses. Sounds like a lot — until you realize there are over 8 billion people on the planet and each of them might have multiple devices.

The Internet Engineering Task Force (IETF) saw this coming in the early 1990s. IPv4 address space was being handed out in enormous blocks, and at the rate things were going, we'd run out. (Spoiler: we did. IANA allocated the last top-level `/8` blocks in **February 2011**.)

Two big solutions were proposed:

1. **IPv6** — a new protocol with 128-bit addresses, giving us approximately 3.4 × 10³⁸ addresses. Enough for every grain of sand on Earth to have its own IP. But IPv6 required upgrading *everything*.
2. **NAT** — a clever hack that lets multiple devices share a single public IP address. No protocol changes needed on the broader internet.

Guess which one got adopted faster?

NAT was standardized in **RFC 1631 (1994)** and later refined in **RFC 3022 (2001)**. It was meant to be a temporary stopgap until IPv6 took over. Thirty years later, NAT is still everywhere, and IPv6 adoption is hovering around 40-45% globally.

## Private vs. public IP ranges (RFC 1918)

For NAT to work, you need a set of IP addresses that are guaranteed to **never appear on the public internet**. That's what RFC 1918 defines — three blocks of "private" address space:

```
┌─────────────────────┬────────────────────────────┬───────────────┐
│ Range               │ CIDR                       │ # of Addresses│
├─────────────────────┼────────────────────────────┼───────────────┤
│ 10.0.0.0 –          │ 10.0.0.0/8                 │ 16,777,216    │
│ 10.255.255.255      │                            │               │
├─────────────────────┼────────────────────────────┼───────────────┤
│ 172.16.0.0 –        │ 172.16.0.0/12              │ 1,048,576     │
│ 172.31.255.255      │                            │               │
├─────────────────────┼────────────────────────────┼───────────────┤
│ 192.168.0.0 –       │ 192.168.0.0/16             │ 65,536        │
│ 192.168.255.255     │                            │               │
└─────────────────────┴────────────────────────────┴───────────────┘
```

Your home router almost certainly hands out addresses from the `192.168.x.x` range. Corporate networks often use `10.x.x.x`. These addresses mean nothing on the public internet — any router that sees a packet with a `192.168.1.42` destination will simply drop it. That's the whole point: they're only meaningful *inside* your local network.

The magic of NAT is turning those private addresses into the one public IP your ISP gave you, and then undoing the transformation when the response comes back.

## How basic NAT works — the translation table

In its simplest form, NAT performs a **one-to-one mapping** between a private IP address and a public IP address. This is sometimes called **Static NAT** or **Basic NAT**.

```
              Private Network              NAT Router            Public Internet
           ┌─────────────────┐        ┌──────────────┐       ┌──────────────────┐
           │ 192.168.1.10    │──────▶ │ Translates   │─────▶│ 203.0.113.5      │
           │                 │        │ Src IP:      │       │ (web server)     │
           │                 │        │ 192.168.1.10 │       │                  │
           │                 │        │ → 198.51.100.1│      │                  │
           └─────────────────┘        └──────────────┘       └──────────────────┘
```

The router maintains a **translation table** (also called a NAT table):

```
┌──────────────────────┬──────────────────────┐
│ Internal (Private)   │ External (Public)    │
├──────────────────────┼──────────────────────┤
│ 192.168.1.10         │ 198.51.100.1         │
│ 192.168.1.11         │ 198.51.100.2         │
│ 192.168.1.12         │ 198.51.100.3         │
└──────────────────────┴──────────────────────┘
```

When a packet leaves the network, the router rewrites the **source IP** in the IP header from the private address to the mapped public address. When a response comes back, it does the reverse — rewrites the **destination IP** from the public address back to the private one.

But there's a problem. Basic NAT still requires **one public IP per device**. If you have 50 devices, you need 50 public IPs. That defeats the whole purpose of conserving addresses.

Enter the real workhorse.

## NAPT / PAT — port-based multiplexing

**Network Address Port Translation** (NAPT), also called **Port Address Translation** (PAT) or colloquially **NAT overload**, is what your home router actually does. It maps *many* private IP:port pairs to a *single* public IP, using different source ports to distinguish connections.

Here's the key insight: a TCP or UDP connection is identified by a **5-tuple**:

```
(Source IP, Source Port, Destination IP, Destination Port, Protocol)
```

The source port is a 16-bit number (1–65535), and clients usually pick an **ephemeral port** (typically in the range 49152–65535) for outbound connections. NAT exploits this — it can remap the source port to create a unique mapping for each connection.

### A concrete example

Say two devices on your home network both want to visit `93.184.216.34:443` (example.com):

```
Device A: 192.168.1.10:52000 → 93.184.216.34:443
Device B: 192.168.1.11:52000 → 93.184.216.34:443
```

Your router (public IP `198.51.100.1`) creates two NAT table entries:

```
┌──────────────────────────┬──────────────────────────┬──────────────────────────┐
│ Internal (Private)       │ External (Public)        │ Destination              │
├──────────────────────────┼──────────────────────────┼──────────────────────────┤
│ 192.168.1.10:52000       │ 198.51.100.1:40010       │ 93.184.216.34:443        │
│ 192.168.1.11:52000       │ 198.51.100.1:40011       │ 93.184.216.34:443        │
└──────────────────────────┴──────────────────────────┴──────────────────────────┘
```

Both devices share the same public IP (`198.51.100.1`), but they're distinguished by different **mapped source ports** (`40010` vs. `40011`). When the web server sends a response to `198.51.100.1:40010`, the router knows to forward it to `192.168.1.10:52000`. When it responds to port `40011`, that goes to Device B.

> [!NOTE]
> Since port numbers are 16-bit, a single public IP can theoretically support up to ~65,000 simultaneous NAT mappings. In practice, routers reserve some ports and have memory limits, but a typical home router handles thousands of concurrent connections just fine.

## The NAT translation table in detail

Let's zoom into what a real NAT table looks like with multiple active connections:

```
┌────┬──────────────────────────┬──────────────────────────┬──────────────────────────┬──────────┬─────────┐
│ #  │ Internal Src             │ External Src (Mapped)    │ Destination              │ Protocol │ TTL (s) │
├────┼──────────────────────────┼──────────────────────────┼──────────────────────────┼──────────┼─────────┤
│  1 │ 192.168.1.10:52000       │ 198.51.100.1:40010       │ 93.184.216.34:443        │ TCP      │   300   │
│  2 │ 192.168.1.10:52001       │ 198.51.100.1:40011       │ 142.250.80.46:443        │ TCP      │   300   │
│  3 │ 192.168.1.11:48500       │ 198.51.100.1:40012       │ 151.101.1.140:443        │ TCP      │   300   │
│  4 │ 192.168.1.20:33120       │ 198.51.100.1:40013       │ 8.8.8.8:53              │ UDP      │    30   │
│  5 │ 192.168.1.20:33121       │ 198.51.100.1:40014       │ 1.1.1.1:53              │ UDP      │    30   │
└────┴──────────────────────────┴──────────────────────────┴──────────────────────────┴──────────┴─────────┘
```

A few things to notice:

- **The same internal device can have multiple mappings** — entry #1 and #2 are both from `192.168.1.10`, just different connections.
- **TTL (Time to Live)** — NAT entries don't live forever. TCP entries typically last 5 minutes of inactivity (some routers go up to 2 hours for established connections). UDP entries are much shorter, often 30 seconds to 2 minutes. If no packets flow, the entry is evicted.
- **Protocol matters** — TCP and UDP NAT entries are tracked separately.

## How return traffic gets routed back

NAT is fundamentally **stateful**. Unlike a simple router that just looks at the destination IP and forwards packets, a NAT device must remember every active outbound connection so it can reverse the translation for incoming responses.

Let's trace the lifecycle of a single connection:

### Step 1: Outbound packet

```
Original packet (leaving your laptop):
┌────────────────────────────────────────────────────┐
│ IP Header                                          │
│   Src:  192.168.1.10    Dst: 93.184.216.34         │
│ TCP Header                                         │
│   Src Port: 52000       Dst Port: 443              │
│ Payload: [TLS ClientHello...]                      │
└────────────────────────────────────────────────────┘
```

### Step 2: NAT rewrites the packet

The router creates a NAT table entry and rewrites the source:

```
Rewritten packet (leaving the router's WAN interface):
┌────────────────────────────────────────────────────┐
│ IP Header                                          │
│   Src:  198.51.100.1    Dst: 93.184.216.34         │
│ TCP Header                                         │
│   Src Port: 40010       Dst Port: 443              │
│ Payload: [TLS ClientHello...]                      │
└────────────────────────────────────────────────────┘
```

The router also recalculates the **IP header checksum** and the **TCP checksum** since both depend on the source IP and port.

### Step 3: Server responds

The web server at `93.184.216.34` sees the request as coming from `198.51.100.1:40010`. It has no idea about your private IP. It sends the response to that address:

```
Response packet (from the server):
┌────────────────────────────────────────────────────┐
│ IP Header                                          │
│   Src: 93.184.216.34    Dst: 198.51.100.1          │
│ TCP Header                                         │
│   Src Port: 443         Dst Port: 40010            │
│ Payload: [TLS ServerHello...]                      │
└────────────────────────────────────────────────────┘
```

### Step 4: NAT reverse-translates

The router receives this packet on its WAN interface. It looks up `198.51.100.1:40010` in the NAT table, finds it maps to `192.168.1.10:52000`, and rewrites the destination:

```
Rewritten packet (delivered to your laptop):
┌────────────────────────────────────────────────────┐
│ IP Header                                          │
│   Src: 93.184.216.34    Dst: 192.168.1.10          │
│ TCP Header                                         │
│   Src Port: 443         Dst Port: 52000            │
│ Payload: [TLS ServerHello...]                      │
└────────────────────────────────────────────────────┘
```

Your laptop receives the packet as if it came straight from the server. It never knows NAT happened.

> [!IMPORTANT]
> This is why you can't just "connect to" a device behind NAT from the outside. If the NAT table doesn't have an entry for the incoming packet (because no outbound packet created one), the router has no idea which internal device to forward it to. It drops the packet. This is NAT's accidental firewall effect.

## Types of NAT

Not all NATs are created equal. RFC 3489 (and its successor RFC 5389) classified NAT behavior into four types based on how they handle the mapping between internal and external addresses. Understanding these types is crucial for peer-to-peer protocols.

### Full Cone NAT (Endpoint-Independent Mapping)

Once an internal address `192.168.1.10:52000` is mapped to `198.51.100.1:40010`, **any** external host can send packets to `198.51.100.1:40010` and they'll be forwarded to the internal device.

```
                        NAT Table
            192.168.1.10:52000 ↔ 198.51.100.1:40010

    Any external host can reach 198.51.100.1:40010
    ─────────────────────────────────────────────────
    Server A  ──▶ 198.51.100.1:40010  ✅ Forwarded
    Server B  ──▶ 198.51.100.1:40010  ✅ Forwarded
    Random IP ──▶ 198.51.100.1:40010  ✅ Forwarded
```

This is the most permissive type. Easy for peer-to-peer, but rare in modern hardware because it's essentially an open door.

### Restricted Cone NAT (Address-Restricted)

The mapping works the same way (endpoint-independent), but the NAT only forwards incoming packets from an external IP **if the internal host has previously sent a packet to that IP**.

```
    192.168.1.10 has sent packets to Server A (93.184.216.34)
    but NOT to Server B (151.101.1.140)
    ─────────────────────────────────────────────────
    Server A  ──▶ 198.51.100.1:40010  ✅ Forwarded (previously contacted)
    Server B  ──▶ 198.51.100.1:40010  ❌ Dropped   (never contacted)
```

### Port-Restricted Cone NAT

Like restricted cone, but the restriction is on both the **IP and port** of the external host. The internal device must have previously sent a packet to that specific IP:port combination.

```
    192.168.1.10 has sent packets to 93.184.216.34:443
    but NOT to 93.184.216.34:80
    ─────────────────────────────────────────────────
    93.184.216.34:443  ──▶ 198.51.100.1:40010  ✅ Forwarded
    93.184.216.34:80   ──▶ 198.51.100.1:40010  ❌ Dropped
```

### Symmetric NAT (Endpoint-Dependent Mapping)

This is the trickiest one. A **different external mapping** is created for each unique destination. If `192.168.1.10:52000` talks to Server A, it gets mapped to `198.51.100.1:40010`. If the same internal socket talks to Server B, it gets mapped to `198.51.100.1:40011` — a *different* external port.

```
    192.168.1.10:52000 → Server A  maps to  198.51.100.1:40010
    192.168.1.10:52000 → Server B  maps to  198.51.100.1:40011
    ─────────────────────────────────────────────────
    Only Server A can reply to :40010
    Only Server B can reply to :40011
    There is NO single external endpoint that "represents" the internal socket.
```

Symmetric NAT is common in enterprise networks and carrier-grade NAT (CGN/CGNAT). It's the hardest to work with for peer-to-peer connections because a peer can't predict what external port will be used for its connection.

> [!TIP]
> You can check your NAT type using STUN test tools. If you're behind symmetric NAT and trying to use WebRTC or VoIP, you'll almost certainly need a TURN relay as a fallback.

## NAT traversal — why peer-to-peer is hard

NAT creates a fundamental problem: two devices behind different NATs cannot directly connect to each other. Neither one can initiate the connection because neither knows the other's NAT mapping, and neither NAT has a table entry to allow the incoming packet.

This is the **NAT traversal problem**, and it has spawned an entire ecosystem of workarounds.

### STUN (Session Traversal Utilities for NAT)

**RFC 5389.** STUN is a simple protocol that lets a device discover its public IP and port as seen from outside the NAT.

```
    ┌──────────────┐          ┌──────────────┐          ┌──────────────┐
    │ Your Device  │──────▶   │   NAT        │──────▶   │  STUN Server │
    │ 192.168.1.10 │          │              │          │ (public)     │
    │ :52000       │          │ Maps to      │          │              │
    │              │          │ 198.51.100.1 │          │ "Your public │
    │              │◀─────────│ :40010       │◀─────────│  address is  │
    │              │          │              │          │ 198.51.100.1 │
    │              │          │              │          │  :40010"     │
    └──────────────┘          └──────────────┘          └──────────────┘
```

Once both peers know their public IP:port (their "server reflexive candidates"), they can attempt a direct connection — this is called **hole punching**.

**UDP hole punching** works like this:
1. Peer A sends a packet to Peer B's public IP:port (as discovered via STUN).
2. This creates an entry in Peer A's NAT table.
3. Peer B does the same toward Peer A.
4. Once both NATs have entries, bidirectional traffic can flow.

The timing has to be coordinated — both peers need to "punch" roughly simultaneously, which is why a **signaling server** (separate from STUN) is used to coordinate.

> [!WARNING]
> Hole punching doesn't work with symmetric NAT. Since the external port changes per destination, the port Peer A discovers via STUN won't be the same port the NAT assigns when Peer A talks to Peer B. The numbers won't match, and the connection fails.

### TURN (Traversal Using Relays around NAT)

**RFC 5766.** When direct connection is impossible (symmetric NAT, strict firewalls), TURN provides a **relay server** on the public internet. Both peers send their traffic to the TURN server, which forwards it between them.

```
    Peer A ──────▶ TURN Server ◀────── Peer B
              (relays traffic)
```

This always works — but it's expensive. All traffic goes through the relay, adding latency and bandwidth cost. That's why TURN is a **last resort**.

### ICE (Interactive Connectivity Establishment)

**RFC 8445.** ICE is the framework that ties it all together. It's used by WebRTC, SIP, and other protocols. ICE gathers multiple connection **candidates** and tries them in order of preference:

1. **Host candidates** — direct LAN addresses (works if both peers are on the same network)
2. **Server reflexive candidates** — public IP:port discovered via STUN (works with most NATs)
3. **Relay candidates** — TURN server relay (always works, worst performance)

ICE performs **connectivity checks** on all candidate pairs simultaneously and picks the best one that actually works. This is why WebRTC calls sometimes take a second or two to "connect" — ICE is running through its candidate pairs.

## NAT64 and the transition to IPv6

IPv6 was supposed to eliminate the need for NAT entirely. With 2¹²⁸ addresses, every device can have a globally routable address. But the transition is taking decades, and during this transition period, IPv6-only devices still need to reach the IPv4 internet.

**NAT64** (RFC 6146) is a translation mechanism between IPv6 and IPv4:

```
    IPv6-only client                 NAT64 Gateway              IPv4 Server
    ┌──────────────┐            ┌──────────────────┐        ┌──────────────┐
    │ 2001:db8::1  │──────────▶ │ Translates       │──────▶ │ 93.184.216.34│
    │              │            │ IPv6 ↔ IPv4       │        │              │
    │              │◀───────────│ packets           │◀──────│              │
    └──────────────┘            └──────────────────┘        └──────────────┘
```

NAT64 is paired with **DNS64**, which synthesizes `AAAA` (IPv6) DNS records for domains that only have `A` (IPv4) records. The synthesized IPv6 address embeds the IPv4 address, typically using the well-known prefix `64:ff9b::/96`:

```
    IPv4 address: 93.184.216.34
    Embedded in IPv6: 64:ff9b::5db8:d822
                      (93 = 0x5d, 184 = 0xb8, 216 = 0xd8, 34 = 0x22)
```

NAT64 is widely deployed by mobile carriers. If you're on a modern mobile network, you might already be using it without knowing — your phone gets an IPv6 address only, and NAT64 handles the IPv4 internet transparently.

> [!NOTE]
> IPv6 doesn't use NAT in the traditional sense. Every device gets a globally unique address, and firewalling (not address translation) is used for security. However, some networks do deploy NPTv6 (Network Prefix Translation for IPv6, RFC 6296) for multihoming or renumbering purposes. It translates prefixes, not individual addresses, and is stateless — philosophically different from IPv4 NAT.

## Real-world example: tracing a packet end to end

Let's trace an HTTPS request from your laptop to `example.com` and back, through every NAT transformation. This is what actually happens, every single time you load a webpage.

### The setup

```
    Your Laptop                Home Router                    ISP / Internet           Web Server
    ┌──────────────┐       ┌──────────────────┐          ┌────────────────┐      ┌──────────────┐
    │ 192.168.1.42 │───────│ LAN: 192.168.1.1 │──────────│                │──────│ 93.184.216.34│
    │              │       │ WAN: 203.0.113.17 │          │   The Internet │      │ (example.com)│
    └──────────────┘       └──────────────────┘          └────────────────┘      └──────────────┘
```

### Step 1: Your laptop creates a TCP SYN

Your browser resolves `example.com` to `93.184.216.34` and initiates a TCP connection. The OS picks ephemeral port `49812`.

```
    Packet on the LAN:
    ┌─────────────────────────────────────────────────────┐
    │ Ethernet: Src MAC=laptop  Dst MAC=router            │
    │ IP:       Src=192.168.1.42  Dst=93.184.216.34       │
    │ TCP:      Src Port=49812    Dst Port=443   [SYN]    │
    └─────────────────────────────────────────────────────┘
```

### Step 2: Router performs NAT

The router receives the packet on its LAN interface. It creates a NAT table entry:

```
    NAT Table Entry:
    Internal:    192.168.1.42:49812
    External:    203.0.113.17:61024
    Destination: 93.184.216.34:443
    Protocol:    TCP
    State:       SYN_SENT
    Timer:       60s (SYN timeout)
```

It rewrites the packet and sends it out the WAN interface:

```
    Packet on the WAN:
    ┌─────────────────────────────────────────────────────┐
    │ Ethernet: Src MAC=router_wan  Dst MAC=isp_gateway   │
    │ IP:       Src=203.0.113.17   Dst=93.184.216.34      │
    │ TCP:      Src Port=61024     Dst Port=443   [SYN]   │
    └─────────────────────────────────────────────────────┘
```

### Step 3: Packet traverses the internet

The packet hops through multiple routers on the internet. None of them know or care that NAT happened — they just see a packet from `203.0.113.17` heading to `93.184.216.34`. Standard IP routing.

### Step 4: Server responds with SYN-ACK

```
    Response packet:
    ┌─────────────────────────────────────────────────────┐
    │ IP:  Src=93.184.216.34   Dst=203.0.113.17           │
    │ TCP: Src Port=443        Dst Port=61024  [SYN-ACK]  │
    └─────────────────────────────────────────────────────┘
```

### Step 5: Router reverse-translates

The router receives the SYN-ACK on its WAN interface. It looks up destination `203.0.113.17:61024` in the NAT table, finds the entry, and rewrites:

```
    Delivered to your laptop:
    ┌─────────────────────────────────────────────────────┐
    │ IP:  Src=93.184.216.34   Dst=192.168.1.42           │
    │ TCP: Src Port=443        Dst Port=49812  [SYN-ACK]  │
    └─────────────────────────────────────────────────────┘
```

The NAT entry state updates to `ESTABLISHED`, and the timer extends to 300+ seconds.

### Step 6: TCP handshake completes, data flows

Your laptop sends the ACK, completing the three-way handshake. Then the TLS handshake happens, then the HTTP request, then the response — every single packet in both directions goes through this same NAT translation. Dozens or hundreds of packets, each rewritten on the fly.

```
    Full round trip:

    Laptop          Router (NAT)           Internet           Server
      │                  │                    │                  │
      │─── SYN ─────────▶│                    │                  │
      │  192.168.1.42:49812                   │                  │
      │                  │── SYN ────────────────────────────────▶│
      │                  │  203.0.113.17:61024                   │
      │                  │                    │                  │
      │                  │◀──────────────────────── SYN-ACK ─────│
      │                  │           203.0.113.17:61024          │
      │◀─── SYN-ACK ────│                    │                  │
      │  192.168.1.42:49812                   │                  │
      │                  │                    │                  │
      │─── ACK ─────────▶│── ACK ────────────────────────────────▶│
      │─── GET / ────────▶│── GET / ──────────────────────────────▶│
      │◀─── 200 OK ──────│◀──────────────────────── 200 OK ──────│
      │                  │                    │                  │
```

Every arrow crossing the NAT boundary involves rewriting IP headers, recalculating checksums, and updating the NAT table's timers. Your router does this for every device, for every connection, at line speed. Modern routers do it in hardware (ASICs) or in optimized kernel code (like Linux's `nf_conntrack` + `iptables`/`nftables`).

## The hidden costs and quirks of NAT

NAT isn't free. Here are the gotchas:

**Protocols that embed IP addresses in payloads break.** FTP's `PORT` command sends the client's IP address *inside the data stream*. SIP (VoIP) embeds addresses in SDP bodies. NAT devices need **Application Layer Gateways (ALGs)** — protocol-aware modules that inspect and rewrite payloads. These ALGs are notoriously buggy.

**IPsec and NAT don't mix easily.** IPsec's AH (Authentication Header) covers the IP header, and NAT changes the IP header, which breaks the authentication. The solution is **NAT-T (NAT Traversal for IPsec, RFC 3948)**, which encapsulates IPsec ESP packets inside UDP port 4500.

**Connection tracking is expensive at scale.** Carrier-grade NAT (CGNAT, RFC 6598, using the `100.64.0.0/10` address range) translates traffic for thousands of subscribers. The NAT table can hold millions of entries, requiring significant memory and CPU.

**Logging and accountability.** If law enforcement needs to trace an IP address back to a specific user behind CGNAT, the ISP needs logs of every NAT mapping — source IP, source port, translated IP, translated port, timestamp. That's an enormous amount of data.

## Wrapping up

NAT was a brilliant hack that saved IPv4 from collapsing under its own success. It lets billions of devices share a limited pool of public addresses by dynamically rewriting packet headers and maintaining a stateful translation table.

But it comes with costs:

- It breaks the **end-to-end principle** of the internet — the idea that any host should be able to directly communicate with any other host.
- It makes **peer-to-peer connections** painful, requiring STUN, TURN, and ICE.
- It adds **complexity and state** to what should be a stateless packet-forwarding layer.
- It requires **protocol-specific hacks** (ALGs) for applications that embed addresses in payloads.

IPv6 is the proper long-term solution, restoring true end-to-end connectivity. But until the transition is complete, NAT remains one of the most important — and most invisible — pieces of the internet's plumbing.

Next time you open a browser tab, remember: your router just performed microsecond-precision packet surgery to make that connection work. Not bad for a "temporary" hack from 1994.
