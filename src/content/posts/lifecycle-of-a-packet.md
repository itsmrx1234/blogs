---
author: Vaibhav Jha
pubDatetime: 2026-09-28T10:00:00Z
title: "Lifecycle of a Packet: From Keystroke to Server and Back"
slug: lifecycle-of-a-packet
featured: true
draft: false
tags:
  - networking
  - deep-dive
  - explainer
description: "What actually happens when you type a URL and hit Enter? Follow a single packet's journey through DNS, TCP handshakes, routing, and back — layer by layer."
---

Every time you type `https://example.com` into your browser and hit Enter, an absurdly complex chain of events unfolds in milliseconds. Dozens of protocols coordinate, packets fly across continents through fiber optic cables, and the answer materializes on your screen as if by magic.

But it's not magic. It's engineering. Let's trace the entire lifecycle of a network packet — from the moment your finger lifts off the Enter key to the moment pixels render on your display.

## Table of contents

## The Starting Point: You Hit Enter

Your browser now has a job: fetch `https://example.com`. But there's an immediate problem — computers don't speak in domain names. They speak in IP addresses. Before a single byte of your HTTP request can leave your machine, the browser needs to figure out *where* `example.com` actually lives.

The browser first checks a cascade of caches:

1. **Browser DNS cache** — Chrome, Firefox, etc. maintain their own in-memory DNS cache.
2. **OS resolver cache** — The operating system (via `systemd-resolved`, `dnsmasq`, or the Windows DNS Client service) keeps its own cache.
3. **`/etc/hosts`** — The classic static lookup file. Still checked on every resolution on most systems.

If any of these have a fresh (non-expired) answer, we skip ahead. But let's assume this is a cold start — no cache hits.

## DNS Resolution: Finding the Address

Your OS sends a DNS query to the **recursive resolver** configured in your system (often your ISP's resolver, or a public one like `8.8.8.8` or `1.1.1.1`). This query is a UDP datagram, typically on port 53, asking: "What is the A (or AAAA) record for `example.com`?"

The recursive resolver now does the heavy lifting. If *it* doesn't have a cached answer, it walks the DNS hierarchy:

```
Your Machine          Recursive Resolver       Root Server (.com TLD)     Authoritative Server
     |                       |                        |                          |
     |--- "A example.com?" ->|                        |                          |
     |                       |--- "Where is .com?" -->|                          |
     |                       |<-- "Try 192.5.6.30" ---|                          |
     |                       |                                                   |
     |                       |--- "A example.com?" ------------------------------>|
     |                       |<-- "93.184.216.34" --------------------------------|
     |<-- "93.184.216.34" ---|
```

Here's what happened:

1. The resolver asks a **root name server** (there are 13 logical root server clusters, labeled A through M) about `example.com`. The root doesn't know the final answer, but it knows who manages `.com` — it returns a referral to the **TLD (Top-Level Domain) name servers**.
2. The resolver follows the referral and asks the `.com` TLD server. The TLD server doesn't know the IP either, but it knows the **authoritative name server** for `example.com` and returns another referral.
3. The resolver asks the authoritative name server, which *does* know the answer: `93.184.216.34`.
4. The resolver caches this answer (respecting the TTL — Time To Live — set by the authoritative server) and returns it to your machine.

> [!NOTE]
> In practice, recursive resolvers almost always have `.com` TLD server addresses cached, so steps 1-2 are usually a single hop. Full walks from the root are rare.

Your browser now knows: `example.com` → `93.184.216.34`. Time to connect.

## The TCP Three-Way Handshake

HTTPS runs over TCP, and TCP is a connection-oriented protocol — before any data flows, both sides must agree to communicate. This is the **three-way handshake**:

```
Client (you)                              Server (93.184.216.34)
     |                                          |
     |----------- SYN (seq=1000) -------------->|  "Hey, I want to talk"
     |                                          |
     |<----- SYN-ACK (seq=4000, ack=1001) -----|  "Sure, I'm here too"
     |                                          |
     |----------- ACK (ack=4001) -------------->|  "Great, let's go"
     |                                          |
     |          CONNECTION ESTABLISHED          |
```

Let's break this down:

1. **SYN** — Your machine sends a TCP segment with the SYN (synchronize) flag set and a random **initial sequence number** (ISN), say `1000`. This is your machine saying "I'd like to open a connection, and I'll be counting my bytes starting from 1000."
2. **SYN-ACK** — The server responds with both SYN and ACK flags set. It acknowledges your sequence number (`ack=1001`, meaning "I've received everything up to 1000, send me 1001 next") and provides its own ISN (`seq=4000`).
3. **ACK** — Your machine acknowledges the server's sequence number (`ack=4001`). The connection is now established.

> [!TIP]
> The random ISNs aren't just counters — they're a security measure. Predictable ISNs made early TCP implementations vulnerable to **session hijacking** attacks. Modern stacks use cryptographically random ISNs.

The entire handshake costs **one round-trip time (RTT)**. For a server across the Atlantic, that's roughly 70-120ms before a single byte of useful data has been exchanged.

## The TLS Handshake: Securing the Channel

Since we're connecting to `https://`, a TLS handshake immediately follows the TCP handshake. TLS 1.3 (the current standard) looks like this:

```
Client                                         Server
  |                                               |
  |--- ClientHello (supported ciphers, random) -->|
  |                                               |
  |<-- ServerHello (chosen cipher, random) -------|
  |<-- Certificate (server's public cert) --------|
  |<-- CertificateVerify (signature proof) -------|
  |<-- Finished ----------------------------------|
  |                                               |
  |--- Finished --------------------------------->|
  |                                               |
  |       ENCRYPTED APPLICATION DATA BEGINS       |
```

The key steps:

1. **ClientHello** — Your browser sends its supported cipher suites, a random nonce, and (crucially in TLS 1.3) its **key share** — a public key for key exchange, speculatively generated.
2. **ServerHello** — The server picks a cipher suite and sends its own random nonce and key share.
3. **Certificate** — The server presents its X.509 certificate chain. Your browser validates this against its trusted Certificate Authority (CA) store.
4. **Key Exchange** — Both sides independently compute the same shared secret using their respective private keys and the other's public key share (typically via **ECDHE** — Elliptic Curve Diffie-Hellman Ephemeral). This gives us **forward secrecy** — even if the server's long-term private key is later compromised, past sessions remain secure.
5. **Finished** — Both sides confirm the handshake integrity with a MAC over the entire transcript.

TLS 1.3 completes in **one RTT** (down from two in TLS 1.2). With **0-RTT resumption**, returning clients can send application data in the very first message, though this comes with replay attack trade-offs.

We now have an encrypted, authenticated channel. Time to use it.

## HTTP Request Construction

Your browser constructs an HTTP/2 (or HTTP/3) request. At its core, it's conceptually similar to the classic HTTP/1.1 text format:

```
GET / HTTP/1.1
Host: example.com
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:130.0)
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br
Connection: keep-alive
```

The key components:

- **Request line**: `GET / HTTP/1.1` — the method (`GET`), the path (`/`), and the protocol version.
- **Host header**: Essential — this tells the server *which* website you want (servers often host many sites on one IP via virtual hosting).
- **Accept-Encoding**: `br` (Brotli), `gzip`, `deflate` — telling the server what compression your browser can handle.
- **User-Agent**: Your browser's identity string — increasingly being deprecated in favor of Client Hints for privacy.

In HTTP/2 and HTTP/3, these headers are sent as compressed binary frames (via HPACK or QPACK), not plaintext — but the semantics are identical.

This HTTP payload is now handed down to the TLS layer for encryption, which passes it further down. And this is where encapsulation begins.

## Encapsulation: Wrapping the Packet Layer by Layer

This is the heart of networking. As data moves down the stack, each layer wraps the previous layer's output in its own header (and sometimes trailer). Think of it like Russian nesting dolls — except each doll adds routing and control information.

```
┌─────────────────────────────────────────────────────────────────────────┐
│ APPLICATION LAYER                                                       │
│ ┌─────────────────────────────────────────────────────────────────────┐ │
│ │  HTTP Request: "GET / HTTP/1.1\r\nHost: example.com\r\n..."        │ │
│ └─────────────────────────────────────────────────────────────────────┘ │
│                              │ encrypted by TLS                         │
│                              ▼                                          │
│ ┌─────────────────────────────────────────────────────────────────────┐ │
│ │  TLS Record: [Content Type | Version | Length | Encrypted Payload] │ │
│ └─────────────────────────────────────────────────────────────────────┘ │
└────────────────────────────────────┬────────────────────────────────────┘
                                     │ passed to TCP
                                     ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ TRANSPORT LAYER (TCP)                                                   │
│ ┌──────────┬──────────┬─────┬─────┬────────┬───────────────────────┐   │
│ │ Src Port │ Dst Port │ Seq │ Ack │ Flags  │   TLS Record Data     │   │
│ │  (49152) │   (443)  │     │     │ [PSH]  │                       │   │
│ └──────────┴──────────┴─────┴─────┴────────┴───────────────────────┘   │
└────────────────────────────────────┬────────────────────────────────────┘
                                     │ passed to IP
                                     ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ NETWORK LAYER (IP)                                                      │
│ ┌─────┬─────┬──────────────┬───────────────┬───────────────────────┐   │
│ │ Ver │ TTL │   Src IP     │    Dst IP     │   TCP Segment         │   │
│ │ (4) │(64) │ 192.168.1.42 │ 93.184.216.34 │                       │   │
│ └─────┴─────┴──────────────┴───────────────┴───────────────────────┘   │
└────────────────────────────────────┬────────────────────────────────────┘
                                     │ passed to NIC driver
                                     ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ DATA LINK LAYER (Ethernet)                                              │
│ ┌──────────────┬──────────────┬──────┬─────────────────┬─────────┐     │
│ │ Dst MAC Addr │ Src MAC Addr │ Type │   IP Datagram   │   FCS   │     │
│ │ aa:bb:cc:...  │ 11:22:33:... │ 0800 │                 │ (CRC)   │     │
│ └──────────────┴──────────────┴──────┴─────────────────┴─────────┘     │
└────────────────────────────────────┬────────────────────────────────────┘
                                     │ transmitted as
                                     ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ PHYSICAL LAYER                                                          │
│  Electrical signals (copper), light pulses (fiber), or radio waves      │
│  (Wi-Fi) — encoding the raw bits of the Ethernet frame                  │
└─────────────────────────────────────────────────────────────────────────┘
```

Each layer only cares about its own header. A router examines the IP header but never peeks into the TCP segment. A switch examines the Ethernet header but ignores everything above it. This **separation of concerns** is what makes the internet composable and scalable.

## Leaving the Host: ARP and the Default Gateway

Your machine has the IP datagram ready, wrapped in an Ethernet frame. But wait — what MAC address does the Ethernet frame use for the destination?

Your machine's IP is `192.168.1.42`, and the server is `93.184.216.34`. They're clearly not on the same local network. So your machine consults its **routing table**:

```bash
$ ip route
default via 192.168.1.1 dev eth0
192.168.1.0/24 dev eth0 proto kernel scope link
```

The destination `93.184.216.34` doesn't match any specific route, so it matches the **default route** — all traffic for unknown destinations goes to the **default gateway** at `192.168.1.1` (your router).

Now your machine needs the **MAC address** of `192.168.1.1`. Enter **ARP (Address Resolution Protocol)**:

```
Your Machine                         Router (192.168.1.1)
     |                                      |
     |-- ARP Request (broadcast) ---------->|  "Who has 192.168.1.1?
     |   "ff:ff:ff:ff:ff:ff"                |   Tell 192.168.1.42"
     |                                      |
     |<-- ARP Reply (unicast) --------------|  "192.168.1.1 is at
     |    "aa:bb:cc:dd:ee:ff"               |   aa:bb:cc:dd:ee:ff"
```

Your machine caches this ARP entry and sets the Ethernet frame's destination MAC to `aa:bb:cc:dd:ee:ff` — the router's MAC address. The **IP destination** is still `93.184.216.34`, but the **Ethernet destination** is the router. This distinction is critical:

> [!IMPORTANT]
> The IP addresses (Layer 3) stay constant end-to-end (modulo NAT). The MAC addresses (Layer 2) change at every hop. Each router strips the old Ethernet frame and wraps the IP datagram in a new one with updated source/destination MACs.

The frame hits the wire (or the airwaves, if you're on Wi-Fi — which adds an extra 802.11 frame layer).

## Traversing the Network: Routers, TTL, and BGP

Your packet has left your home network and entered the broader internet. Its journey might look something like this:

```
Your PC ──> Home Router ──> ISP Router ──> Regional Hub ──> IXP ──> ...
   ... ──> Transit Provider ──> Destination's Edge Router ──> Server

         hop 1       hop 2        hop 3       hop 4     ...   hop N
         TTL=64      TTL=63       TTL=62      TTL=61          TTL=?
```

At every hop, the following happens:

1. The router receives the Ethernet frame on an ingress interface.
2. It strips the Ethernet header, examines the **IP destination address**.
3. It consults its **routing table** (populated by routing protocols) to determine the **next hop** and **egress interface**.
4. It **decrements the TTL** (Time To Live) by 1. If TTL reaches 0, the packet is dropped and an ICMP "Time Exceeded" message is sent back to the source. This prevents routing loops from circulating packets forever — and is exactly how `traceroute` works.
5. It wraps the IP datagram in a new Ethernet frame (or whatever Layer 2 protocol the egress link uses) with the next hop's MAC address.
6. It forwards the frame out the egress interface.

### How Do Routers Know Where to Send Things?

Home routers have simple routing tables — basically "everything goes to my ISP." But core internet routers maintain enormous tables with 900,000+ prefixes, built using **BGP (Border Gateway Protocol)**.

BGP is the protocol that glues the internet together. Every major network (called an **Autonomous System**, or AS) uses BGP to announce which IP prefixes it owns and to learn paths to every other prefix. When you hear about an "internet outage caused by a BGP misconfiguration," this is the system that broke.

BGP is a **path-vector** protocol — routers exchange not just destinations but the full path of AS numbers to reach them. This lets routers make policy-based decisions: "I'd rather route through AS 3356 (Lumen) than AS 174 (Cogent) for this prefix, because of our peering agreement."

> [!NOTE]
> You can see the real hops your packets take using `traceroute` (Linux/macOS) or `tracert` (Windows). Each line is a router that decremented your TTL to zero and sent back an ICMP message.

## NAT: The Address Translation Detour

There's a wrinkle we glossed over. Your machine's IP address is `192.168.1.42` — an **RFC 1918 private address**. It's not routable on the public internet. So how does your packet actually get there?

**NAT (Network Address Translation)** — performed by your home router. When your packet passes through the router, it:

1. **Replaces the source IP** (`192.168.1.42`) with the router's public IP (say `203.0.113.55`).
2. **Replaces the source port** (`49152`) with a port from its own pool (say `23847`).
3. **Records the mapping** in its NAT table:

```
NAT Translation Table:
┌──────────────────────────┬─────────────────────────┐
│ Internal                 │ External                │
├──────────────────────────┼─────────────────────────┤
│ 192.168.1.42:49152       │ 203.0.113.55:23847      │
│ 192.168.1.105:51200      │ 203.0.113.55:23848      │
│ ...                      │ ...                     │
└──────────────────────────┴─────────────────────────┘
```

When the response comes back addressed to `203.0.113.55:23847`, the router looks up the mapping and translates the destination back to `192.168.1.42:49152`, forwarding it to your machine.

NAT is technically a hack — it was invented to delay IPv4 address exhaustion (we only have ~4.3 billion IPv4 addresses for a world with tens of billions of devices). But it's become so entrenched that it now serves as an accidental firewall, blocking unsolicited inbound connections.

> [!TIP]
> With IPv6, every device gets a globally unique address, making NAT unnecessary. The transition is ongoing — as of 2026, roughly 45-50% of internet traffic uses IPv6.

## Arriving at the Destination Server

After traversing perhaps 10-20 router hops across ISPs, internet exchanges, and undersea cables, the packet arrives at the server's network. The final steps mirror the first steps in reverse:

1. The edge router for `example.com`'s network receives the packet and routes it to the correct server (or load balancer) based on the destination IP.
2. A final ARP lookup (or NDP for IPv6) resolves the server's IP to its MAC address on the local network.
3. The Ethernet frame arrives at the server's NIC (Network Interface Card).

Now **decapsulation** begins — the reverse of encapsulation:

```
Physical Layer:   Electrical/optical signals → raw bits
                           │
                           ▼
Data Link Layer:  Strip Ethernet header/trailer
                  Validate FCS (Frame Check Sequence / CRC)
                  Hand IP datagram up
                           │
                           ▼
Network Layer:    Strip IP header
                  Verify destination IP matches
                  Check header checksum
                  Hand TCP segment up
                           │
                           ▼
Transport Layer:  Strip TCP header
                  Reassemble stream (handle ordering, retransmissions)
                  Deliver to socket on port 443
                           │
                           ▼
TLS Layer:        Decrypt payload using session keys
                  Verify record MAC / AEAD authentication tag
                  Hand plaintext up
                           │
                           ▼
Application:      HTTP server (nginx, Apache, etc.) receives:
                  "GET / HTTP/1.1\r\nHost: example.com\r\n..."
                  Processes request, generates response
```

The web server processes the request — reads the index page from disk (or cache), wraps it in an HTTP response with headers like `Content-Type: text/html`, `Content-Encoding: br`, status `200 OK`, and hands it back down the stack.

## The Response Journey Back

The response undergoes the exact same encapsulation process, but in reverse direction:

```
Server (93.184.216.34)                    Client (203.0.113.55, NAT'd)
       |                                         |
       |  HTTP 200 OK + HTML body                 |
       |  → encrypted by TLS                      |
       |  → wrapped in TCP (src:443, dst:23847)   |
       |  → wrapped in IP (src:93.184.216.34,     |
       |                    dst:203.0.113.55)      |
       |  → wrapped in Ethernet                   |
       |                                         |
       |--- [packet traverses internet] -------->|
       |                                         |
       |  NAT translates dst to 192.168.1.42:49152|
       |  Decapsulated, decrypted, rendered       |
```

The packet traverses routers back (not necessarily the same path — internet routing is asymmetric), NAT reverses the address translation, and your machine's TCP stack reassembles the response, TLS decrypts it, and the browser begins parsing and rendering HTML.

For a typical web page, this process repeats **hundreds of times** — for CSS files, JavaScript bundles, images, fonts, API calls. HTTP/2 multiplexing and HTTP/3's QUIC protocol make this dramatically more efficient by allowing multiple request/response streams over a single connection.

## TCP Connection Teardown: The Four-Way FIN

When the browser is done (or after a keep-alive timeout), the TCP connection needs to be cleanly closed. TCP uses a **four-way handshake** for teardown:

```
Client                                    Server
  |                                          |
  |---------- FIN (seq=5000) -------------->|  "I'm done sending"
  |                                          |
  |<--------- ACK (ack=5001) ---------------|  "Got it"
  |                                          |
  |<--------- FIN (seq=8000) ---------------|  "I'm done too"
  |                                          |
  |---------- ACK (ack=8001) -------------->|  "Acknowledged"
  |                                          |
  |         CONNECTION CLOSED               |
```

Why four packets instead of three? Because TCP is **full-duplex** — each side can send independently. The client saying "I'm done sending" doesn't mean the server is done sending. So each side independently signals its FIN (finish), and each FIN must be acknowledged.

After sending the final ACK, the client enters a **TIME_WAIT** state (typically lasting 2× the Maximum Segment Lifetime, usually 60 seconds). During this period, the client keeps the connection state in memory to handle any delayed duplicate packets from the server. This prevents old packets from a closed connection from being misinterpreted as belonging to a new connection on the same port pair.

> [!WARNING]
> On busy servers, too many connections in TIME_WAIT can exhaust port space. This is why production servers tune `net.ipv4.tcp_tw_reuse` and use connection pooling.

## The Full Picture

Let's zoom all the way out. Here's the complete lifecycle, end to end:

```
 YOU TYPE "https://example.com" + ENTER
                 │
                 ▼
 ┌───────────────────────────────┐
 │  1. DNS Resolution            │  UDP query → recursive resolver → root
 │     example.com → 93.184.216.34│  → TLD → authoritative → cached answer
 └───────────────┬───────────────┘
                 ▼
 ┌───────────────────────────────┐
 │  2. TCP Handshake             │  SYN → SYN-ACK → ACK
 │     (1 RTT)                   │  Connection established
 └───────────────┬───────────────┘
                 ▼
 ┌───────────────────────────────┐
 │  3. TLS Handshake             │  ClientHello → ServerHello → keys
 │     (1 RTT for TLS 1.3)      │  Encrypted channel established
 └───────────────┬───────────────┘
                 ▼
 ┌───────────────────────────────┐
 │  4. HTTP GET Request          │  Encapsulated: HTTP → TLS → TCP → IP
 │     Encapsulation + transmit  │  → Ethernet → physical medium
 └───────────────┬───────────────┘
                 ▼
 ┌───────────────────────────────┐
 │  5. Network Transit           │  ARP → NAT → router hops (TTL--)
 │     Home → ISP → Internet     │  → BGP routing → destination network
 └───────────────┬───────────────┘
                 ▼
 ┌───────────────────────────────┐
 │  6. Server Processing         │  Decapsulation → decrypt → process
 │     Decapsulate + respond     │  → HTTP 200 + HTML body
 └───────────────┬───────────────┘
                 ▼
 ┌───────────────────────────────┐
 │  7. Response Transit          │  Re-encapsulate → route back
 │     Back through the internet │  → NAT reverse → deliver to client
 └───────────────┬───────────────┘
                 ▼
 ┌───────────────────────────────┐
 │  8. Render + Teardown         │  Browser parses HTML, fetches assets
 │     Display page, close conn  │  → FIN → ACK → FIN → ACK
 └───────────────────────────────┘
```

## Wrapping Up

What feels like an instant — typing a URL and seeing a page — involves:

- **~6 protocol handshakes** (DNS, TCP, TLS)
- **~10-20 router hops** each way
- **~5 layers of encapsulation and decapsulation** at each end
- **Address translations** at NAT boundaries
- **Cryptographic operations** for encryption and authentication
- **Global routing convergence** via BGP across autonomous systems

And this is the *simplified* version. We didn't cover TCP congestion control (slow start, cubic, BBR), PMTU discovery, ECMP load balancing, anycast routing, CDNs, HTTP/3's QUIC (which merges the TCP and TLS handshakes into a single RTT over UDP), or what happens when things go wrong (retransmissions, ICMP errors, connection resets).

Every one of these topics is its own rabbit hole. But understanding the basic lifecycle — DNS → TCP → TLS → HTTP → encapsulate → route → decapsulate → respond → route back → teardown — gives you the mental model to debug almost any network issue you'll ever encounter.

The next time `curl` hangs, you'll know *where* in this chain to look.
