---
author: Vaibhav Jha
pubDatetime: 2026-08-05T10:00:00Z
title: "How TCP/IP Actually Works: Reliable Delivery on an Unreliable Network"
slug: how-tcp-ip-works
featured: false
draft: false
tags:
  - networking
  - protocols
  - deep-dive
description: "TCP makes the chaotic internet feel reliable. But how does it guarantee delivery, handle congestion, and keep packets in order? Let's trace through the protocol."
---

Every time you load a webpage, send an email, or SSH into a server, TCP is doing the heavy lifting underneath. It takes a network that drops packets, reorders them, duplicates them, and corrupts them — and somehow delivers your data perfectly, in order, every time.

That's a remarkable trick. Let's figure out how it works.

## Table of contents

## The Problem: Networks Are Unreliable

The internet is a packet-switched network. Your data gets chopped into small chunks (packets), and each one is thrown into the network independently. Each packet might take a different route. Some might arrive out of order. Some might arrive twice. Some might not arrive at all.

IP — the Internet Protocol — gives us addressing and routing. It gets a packet from point A to point B. But IP makes absolutely **no guarantees**:

- **No delivery guarantee** — packets can be dropped by congested routers
- **No ordering guarantee** — packet 5 might arrive before packet 3
- **No duplicate protection** — a packet might be delivered twice
- **No integrity guarantee** — bits can flip in transit (though IP has a basic header checksum)

IP is a "best-effort" protocol. It _tries_ to deliver your packet, but if it fails, it shrugs and moves on. No notification, no retry.

This is actually a brilliant design decision — it keeps the network layer simple and fast. But somebody has to build reliability on top. That's TCP's job.

## TCP vs UDP: Choosing Your Trade-offs

Before we dive deep into TCP, it's worth understanding why UDP exists at all. Both are transport-layer protocols that sit on top of IP, but they make fundamentally different trade-offs.

| Feature | TCP | UDP |
|---|---|---|
| Delivery guarantee | Yes (retransmissions) | No |
| Ordering guarantee | Yes (sequence numbers) | No |
| Connection state | Connection-oriented | Connectionless |
| Flow control | Yes (receiver window) | No |
| Congestion control | Yes | No |
| Header overhead | 20-60 bytes | 8 bytes |
| Latency | Higher (handshake + ACKs) | Lower |

**Use TCP when** you need every byte to arrive correctly and in order: HTTP, SSH, SMTP, database connections, file transfers.

**Use UDP when** speed matters more than perfect delivery: DNS lookups, video streaming, online gaming, VoIP. In these cases, a dropped packet is better than a delayed one — you'd rather skip a video frame than freeze the stream waiting for a retransmission.

> [!NOTE]
> UDP isn't "unreliable TCP." It's a deliberate choice. Many protocols that use UDP implement their own reliability mechanisms on top, tuned for their specific needs. QUIC, for example, builds reliable transport over UDP.

## The TCP Segment Structure

Every TCP segment has a header that carries all the metadata the protocol needs. Let's walk through it:

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|          Source Port          |       Destination Port        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                        Sequence Number                        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Acknowledgment Number                      |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  Data |       |C|E|U|A|P|R|S|F|                               |
| Offset| Rsrvd |W|C|R|C|S|S|Y|I|            Window             |
|       |       |R|E|G|K|H|T|N|N|                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|           Checksum            |         Urgent Pointer        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Options (variable)                   |  Pad |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                             Data                              |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

The key fields:

- **Source/Destination Port (16 bits each)**: Identifies the sending and receiving application. This is how your OS knows that incoming data on port 443 goes to your web server, not your SSH daemon.

- **Sequence Number (32 bits)**: The byte offset of the first byte of data in this segment. If you've sent 1000 bytes already and are now sending bytes 1000–1499, the sequence number is 1000. This is how TCP keeps data in order.

- **Acknowledgment Number (32 bits)**: The next byte the sender _expects_ to receive. "I've received everything up to byte 5000, so send me byte 5000 next." This is cumulative — it acknowledges everything before it.

- **Flags (9 bits)**: Control bits that signal what kind of segment this is:
  - **SYN** — Synchronize sequence numbers (connection setup)
  - **ACK** — Acknowledgment number is valid
  - **FIN** — Sender is finished sending data (connection teardown)
  - **RST** — Reset the connection (something went wrong)
  - **PSH** — Push data to the application immediately
  - **URG** — Urgent pointer is valid
  - **CWR/ECE** — Congestion notification (ECN)

- **Window Size (16 bits)**: How many bytes the receiver is willing to accept right now. This is the core of flow control.

- **Checksum (16 bits)**: Covers the header, data, and a pseudo-header from the IP layer. Catches corrupted segments.

> [!TIP]
> The minimum TCP header is 20 bytes (no options). With common options like timestamps and window scaling, it's usually 32 bytes. That's significant overhead for small payloads — one reason why UDP is preferred for tiny, frequent messages.

## The Three-Way Handshake

Before any data flows, TCP establishes a connection. Both sides need to agree on initial sequence numbers and exchange capabilities. This happens in three steps:

```
    Client                                  Server
      |                                       |
      |  1. SYN (seq=100)                     |
      |-------------------------------------->|
      |                                       |
      |  2. SYN-ACK (seq=300, ack=101)        |
      |<--------------------------------------|
      |                                       |
      |  3. ACK (seq=101, ack=301)            |
      |-------------------------------------->|
      |                                       |
      |         Connection Established         |
```

Let's trace the sequence number math:

**Step 1 — SYN**: The client picks a random initial sequence number (ISN), say 100. It sends a SYN segment with `seq=100`. This says "I'd like to start a connection, and my byte numbering starts at 100." The SYN flag itself consumes one sequence number.

**Step 2 — SYN-ACK**: The server picks its own ISN, say 300. It responds with both SYN and ACK flags set: `seq=300, ack=101`. The `ack=101` means "I received your SYN (which consumed sequence number 100), and I'm now expecting byte 101 next." The server's SYN also consumes one sequence number.

**Step 3 — ACK**: The client acknowledges the server's SYN: `seq=101, ack=301`. "I got your SYN (sequence 300), expecting byte 301 next." At this point, both sides are synchronized.

### Why random initial sequence numbers?

If sequence numbers always started at 0, an attacker could predict them and inject forged packets into a connection. Random ISNs make this significantly harder. They also prevent confusion from old segments belonging to a previous connection between the same host:port pairs (a "reincarnated connection").

### Why three steps, not two?

Two steps wouldn't work. The server needs confirmation that the client received the server's ISN. Without step 3, the server would start sending data with sequence numbers the client hasn't acknowledged understanding. The third step proves the round-trip works in both directions.

> [!WARNING]
> The three-way handshake adds one full round-trip time (RTT) of latency before any data can be sent. On a 200ms transatlantic link, that's 200ms of waiting. This is one reason HTTP/1.1 introduced keep-alive connections, and it's the motivation behind TCP Fast Open (covered later).

## Data Transfer: Sequence Numbers and Acknowledgments

Once the connection is established, data flows. Let's watch a simple exchange:

```
    Client                                  Server
      |                                       |
      |  seq=101, data="Hello" (5 bytes)      |
      |-------------------------------------->|
      |                                       |
      |  seq=106, data=" World" (6 bytes)     |
      |-------------------------------------->|
      |                                       |
      |            ack=112                     |
      |<--------------------------------------|
      |                                       |
```

The client sends "Hello" (5 bytes) starting at sequence 101. The next segment starts at 106 (101 + 5). The server can acknowledge both at once with `ack=112` (106 + 6), meaning "I've received everything up to byte 112."

This is **cumulative acknowledgment** — a single ACK covers all bytes up to the acknowledged number. It's efficient, but it has a downside: if packet 2 is lost but packet 3 arrives, the receiver can only acknowledge up to the last contiguously received byte. The sender doesn't know which specific packet was lost.

### Selective Acknowledgment (SACK)

**SACK** (RFC 2018) fixes this. It's a TCP option that lets the receiver say: "I've received bytes 101–105 and 112–120, but I'm missing 106–111." The sender can then retransmit only the missing segment, instead of everything from 106 onward. SACK is almost universally enabled today.

### The Sliding Window

TCP doesn't send one segment and wait for an ACK. That would waste bandwidth — you'd spend most of your time waiting for acknowledgments to cross the network.

Instead, TCP uses a **sliding window**. The sender is allowed to have multiple unacknowledged segments "in flight" at once. The window size determines how many:

```
    Sent & ACKed    |  Sent, not ACKed  |  Can send  |  Cannot send yet
                    |   (in flight)     |            |
    [==============]|[=====][=====][===]|[    ][    ]|[              ]
                    ^                    ^            ^
                SendBase            NextSeqNum    Window Edge
```

As ACKs come back, the left edge of the window slides forward, allowing new data to be sent. This is where TCP gets its throughput — it keeps the pipe full.

The actual window size is the **minimum** of two values:
- The **receiver window** (rwnd) — how much buffer space the receiver has
- The **congestion window** (cwnd) — how much the sender thinks the network can handle

## Flow Control: Don't Overwhelm the Receiver

Flow control prevents a fast sender from flooding a slow receiver. The mechanism is simple: the receiver advertises its available buffer space in the **Window** field of every ACK.

```
    Client                                    Server
      |                                         |
      |  data (1000 bytes)                      |
      |---------------------------------------->|
      |                                         |  buffer: 3000/4096
      |  ack, window=3096                       |
      |<----------------------------------------|
      |                                         |
      |  data (2000 bytes)                      |
      |---------------------------------------->|
      |                                         |  buffer: 5000/4096
      |  ack, window=1096                       |  (app consumed some)
      |<----------------------------------------|
      |                                         |
      |  data (1096 bytes)                      |  window full!
      |---------------------------------------->|
      |                                         |  buffer full
      |  ack, window=0                          |  "STOP SENDING"
      |<----------------------------------------|
      |                                         |
      |  ... sender waits ...                   |
      |                                         |  app reads data
      |  ack, window=2048                       |  "OK, send more"
      |<----------------------------------------|
      |                                         |
```

When the window hits zero, the sender stops. It periodically sends tiny **window probe** segments to check if the receiver has freed up space (in case the window update ACK was lost).

### Window Scaling

The Window field is 16 bits, giving a maximum of 65,535 bytes. That's far too small for high-bandwidth, high-latency links (a 1 Gbps link with 100ms RTT needs a window of ~12.5 MB to stay full). **Window scaling** (RFC 7323) is a TCP option negotiated during the handshake that multiplies the window value by a power of 2, supporting windows up to ~1 GB.

## Congestion Control: Don't Overwhelm the Network

Flow control protects the receiver. **Congestion control** protects the network. A sender that blasts data faster than intermediate routers can handle will cause queue buildup, increased latency, and eventually packet drops.

TCP's congestion control is one of the most important algorithms on the internet. Without it, the network would collapse under load — a phenomenon called **congestion collapse**, which actually happened in the early internet (October 1986).

The core idea: TCP maintains a **congestion window** (cwnd) that limits how much data is in flight. The sender adjusts cwnd based on feedback from the network (packet loss, delay).

### Slow Start

When a connection begins, TCP doesn't know the network's capacity. It starts conservatively:

1. Set `cwnd = 1 MSS` (Maximum Segment Size, usually ~1460 bytes)
2. For every ACK received, increase cwnd by 1 MSS
3. This doubles cwnd every RTT — exponential growth

```
RTT 1:  cwnd = 1 MSS   → send 1 segment
RTT 2:  cwnd = 2 MSS   → send 2 segments
RTT 3:  cwnd = 4 MSS   → send 4 segments
RTT 4:  cwnd = 8 MSS   → send 8 segments
RTT 5:  cwnd = 16 MSS  → send 16 segments
```

"Slow start" is a misnomer — it's actually exponential growth, which is quite aggressive. It's only "slow" compared to the alternative of starting at full blast.

Slow start continues until cwnd reaches the **slow start threshold** (ssthresh), at which point TCP switches to congestion avoidance.

### Congestion Avoidance

Once cwnd ≥ ssthresh, TCP enters congestion avoidance mode. Instead of doubling every RTT, cwnd grows by roughly 1 MSS per RTT — linear growth. This probes for available bandwidth more carefully.

The formula: for each ACK, increase cwnd by `MSS * MSS / cwnd`. This achieves roughly 1 MSS increase per full window of data acknowledged.

### What Happens When a Packet Is Lost?

Packet loss is TCP's primary signal that the network is congested. (This assumption doesn't hold on wireless networks, but that's a different story.) How TCP reacts depends on the variant:

#### TCP Tahoe (original)

On **any** packet loss (detected by timeout or duplicate ACKs):
1. Set `ssthresh = cwnd / 2`
2. Set `cwnd = 1 MSS`
3. Re-enter slow start

This is extremely conservative. You go all the way back to square one.

#### TCP Reno

Reno distinguishes between **timeout** (severe congestion) and **triple duplicate ACKs** (mild congestion, a single lost packet):

**On timeout:**
- Same as Tahoe: `ssthresh = cwnd / 2`, `cwnd = 1`, slow start

**On triple duplicate ACK (Fast Retransmit + Fast Recovery):**
1. Retransmit the lost segment immediately (don't wait for timeout)
2. Set `ssthresh = cwnd / 2`
3. Set `cwnd = ssthresh + 3 MSS` (the 3 dup-ACKs mean 3 segments got through)
4. Enter fast recovery: for each additional duplicate ACK, inflate cwnd by 1 MSS
5. When a new (non-duplicate) ACK arrives, set `cwnd = ssthresh` and enter congestion avoidance

This is much better — the connection doesn't collapse to a crawl after a single lost packet.

```
cwnd
  |        /\
  |       /  \                     /
  |      /    \__________/\       /
  |     /     Congestion   \     /
  |    /      Avoidance     \   /
  |   /                     Fast
  |  / Slow                 Recovery
  | /  Start
  |/________________________\_________> time
  |                         ^
  |                    Triple dup ACK
```

#### TCP CUBIC

**CUBIC** (the default on Linux since 2.6.19) replaces Reno's linear congestion avoidance with a cubic function. After a loss event, cwnd recovery follows a cubic curve centered on the pre-loss window size. This makes CUBIC:

- **More aggressive on high-bandwidth links** — it probes for bandwidth faster
- **More conservative near the loss point** — it approaches the previous maximum slowly to avoid repeated losses
- **RTT-fair** — unlike Reno, CUBIC's window growth doesn't depend on RTT, so connections with different latencies get fairer bandwidth allocation

The cubic function:

```
W(t) = C × (t - K)³ + W_max
```

Where `W_max` is the window size at the last loss event, `K` is the time it takes to reach `W_max` again, and `C` is a scaling constant.

## Connection Teardown

TCP connections are full-duplex — data flows in both directions independently. So closing a connection requires each side to finish independently. This is the **four-way termination**:

```
    Client                                  Server
      |                                       |
      |  FIN (seq=500)                        |
      |-------------------------------------->|  "I'm done sending"
      |                                       |
      |            ACK (ack=501)              |
      |<--------------------------------------|  "Got it"
      |                                       |
      |   (server may keep sending data...)   |
      |                                       |
      |            FIN (seq=800)              |
      |<--------------------------------------|  "I'm done too"
      |                                       |
      |  ACK (ack=801)                        |
      |-------------------------------------->|  "Got it"
      |                                       |
      |  [TIME_WAIT - 2×MSL]                  |
      |                                       |
```

The FIN flag, like SYN, consumes one sequence number.

### The TIME_WAIT State

After sending the final ACK, the client enters **TIME_WAIT** for 2×MSL (Maximum Segment Lifetime, typically 60 seconds on Linux, making TIME_WAIT last 60s total). This exists for two reasons:

1. **Reliability**: If the final ACK is lost, the server will retransmit its FIN. The client needs to be alive to re-ACK it.

2. **Old segment protection**: If a new connection is immediately established on the same port pair, old segments from the previous connection might be misinterpreted as new data. TIME_WAIT ensures they've expired.

> [!WARNING]
> On busy servers, thousands of sockets in TIME_WAIT can exhaust ephemeral ports. Common mitigations include `SO_REUSEADDR`, `SO_REUSEPORT`, `tcp_tw_reuse`, and reducing the MSL value. Be careful — blindly lowering TIME_WAIT can cause subtle data corruption in edge cases.

## TCP in the Real World

The textbook version of TCP is clean. Real-world TCP has decades of accumulated optimizations and quirks.

### Nagle's Algorithm

When an application writes data one byte at a time (think: typing in a telnet session), each byte gets its own TCP segment — 20+ bytes of header for 1 byte of data. That's catastrophically inefficient.

**Nagle's algorithm** (RFC 896) buffers small writes. It only sends a sub-MSS segment if there are no unacknowledged segments in flight. Otherwise, it waits until either:
- Enough data accumulates to fill an MSS, or
- The outstanding ACK comes back

This dramatically reduces the number of tiny packets.

### TCP_NODELAY

But Nagle's algorithm introduces latency. If you're building a real-time protocol (game server, trading system, interactive RPC), that extra buffering delay is unacceptable. Setting the `TCP_NODELAY` socket option disables Nagle's algorithm — every write becomes a segment immediately.

```c
int flag = 1;
setsockopt(fd, IPPROTO_TCP, TCP_NODELAY, &flag, sizeof(flag));
```

### Delayed ACKs

TCP doesn't ACK every segment immediately. It waits up to ~40ms hoping to piggyback the ACK on outgoing data (saving a packet). This interacts badly with Nagle's algorithm — Nagle waits for an ACK, delayed ACKs wait for data, and nothing moves for 40ms. This is the infamous **Nagle-delayed ACK interaction**. It's why many servers set `TCP_NODELAY`.

### TCP Keep-Alive

An idle TCP connection has zero network traffic. Neither side sends anything. The connection could be dead (the other end crashed, a NAT table expired, a firewall timed out) and you wouldn't know.

**TCP keep-alive** sends periodic probe segments on idle connections. On Linux, the defaults are:

```
tcp_keepalive_time  = 7200  (seconds before first probe - 2 hours!)
tcp_keepalive_intvl = 75    (seconds between probes)
tcp_keepalive_probes = 9    (number of failed probes before dropping)
```

Two hours is a long time. Most applications that care about connection liveness implement their own heartbeat at the application layer.

## Modern Improvements

TCP is nearly 50 years old (RFC 793 was published in 1981), but it's still evolving.

### TCP Fast Open (TFO)

Standard TCP requires a full round-trip handshake before data can be sent. **TCP Fast Open** (RFC 7413) allows data in the SYN packet on subsequent connections.

How it works:
1. **First connection**: Client requests a TFO cookie during the handshake. Server generates one from the client's IP.
2. **Subsequent connections**: Client includes the cookie and data in the SYN. Server validates the cookie and processes the data immediately — saving one RTT.

```
    Standard TCP              TCP Fast Open (repeat)
    Client    Server          Client    Server
      |  SYN     |              |  SYN+Data+Cookie  |
      |--------->|              |------------------>|
      |  SYN-ACK |              |  SYN-ACK+Data     |
      |<---------|              |<------------------|
      |  ACK+Data|              |  ACK              |
      |--------->|              |------------------>|
      |          |              |                    |
    (1 RTT before data)       (0 RTT before data)
```

### Multipath TCP (MPTCP)

Standard TCP binds a connection to a single pair of IP addresses. **MPTCP** (RFC 8684) allows a single TCP connection to use multiple paths simultaneously — for example, Wi-Fi and cellular on a phone.

Benefits:
- **Redundancy**: If one path fails, traffic shifts to the other
- **Bandwidth aggregation**: Use both paths at once for higher throughput
- **Seamless mobility**: Move between networks without dropping the connection

Apple uses MPTCP extensively in iOS for Siri and other services.

### BBR Congestion Control

Traditional congestion control (Reno, CUBIC) uses **packet loss** as the congestion signal. But modern networks often have deep buffers — by the time packets are dropped, the queues are already full and latency has spiked. This is **bufferbloat**.

**BBR** (Bottleneck Bandwidth and Round-trip propagation time), developed by Google, takes a different approach. Instead of waiting for loss, it builds a model of the network path by measuring:
- **Bottleneck bandwidth (BtlBw)**: The maximum delivery rate observed
- **Round-trip propagation time (RTprop)**: The minimum RTT observed

BBR then paces its sending rate to `BtlBw × RTprop`, keeping the pipe full without filling buffers. This achieves:
- Higher throughput on bufferbloated networks
- Lower latency (it doesn't fill queues)
- Better performance on lossy links (random loss doesn't trigger cwnd collapse)

BBR is used by Google's servers and is available as a congestion control module in the Linux kernel:

```bash
# Check current congestion control algorithm
sysctl net.ipv4.tcp_congestion_control

# Switch to BBR
sudo sysctl -w net.ipv4.tcp_congestion_control=bbr
```

## Where TCP Falls Short

For all its sophistication, TCP has fundamental limitations that are hard to fix without starting over.

### Head-of-Line Blocking

TCP guarantees **in-order delivery**. If you're multiplexing multiple independent streams over a single TCP connection (as HTTP/2 does), a single lost packet blocks _all_ streams until it's retransmitted. Stream B's data is sitting in the receive buffer, complete and ready, but the kernel won't deliver it because stream A's lost packet hasn't arrived yet.

This is **head-of-line blocking**, and it's TCP's Achilles' heel for multiplexed protocols.

### Ossification

TCP is implemented in OS kernels and understood by middleboxes (firewalls, NATs, load balancers). Changing TCP's wire format is nearly impossible — middleboxes drop or mangle packets with unfamiliar TCP options. This makes deploying new TCP features incredibly slow.

### The QUIC Solution

**QUIC** (RFC 9000) addresses both problems by rebuilding transport on top of UDP:

- **No head-of-line blocking**: QUIC is natively multi-stream. Loss on one stream doesn't affect others.
- **No ossification**: QUIC encrypts almost the entire transport header. Middleboxes can't inspect or modify what they can't see.
- **0-RTT connection establishment**: QUIC combines the transport and TLS handshakes, establishing an encrypted connection in a single round-trip (or zero for repeat connections).
- **Connection migration**: Connections are identified by a connection ID, not the IP/port tuple, so they survive network changes.

HTTP/3 is HTTP over QUIC. It's the direction the internet is heading. But QUIC itself is deeply influenced by TCP — it uses many of the same congestion control algorithms, the same concept of ACKs and sequence numbers, and the same reliability guarantees. Understanding TCP is understanding the foundation that QUIC builds on.

## Wrapping Up

TCP is a masterclass in protocol design under constraints. It takes an unreliable network and builds reliable, ordered, flow-controlled, congestion-aware byte streams on top. The key mechanisms:

1. **Sequence numbers** keep data in order
2. **Acknowledgments** confirm delivery
3. **Retransmissions** recover from loss
4. **The sliding window** keeps the pipe full
5. **Flow control** protects the receiver
6. **Congestion control** protects the network
7. **The three-way handshake** establishes shared state
8. **The four-way teardown** cleanly releases it

Every one of these mechanisms involves trade-offs. TCP chose reliability over speed, correctness over simplicity, and the internet runs on those choices. Even as QUIC takes over, the ideas in TCP — the sequence number arithmetic, the congestion control algorithms, the fundamental tension between throughput and fairness — remain the bedrock of networked computing.

Next time you `curl` a URL and get a response back, remember: an army of sequence numbers, acknowledgments, and congestion windows just fought through the chaos of the internet to deliver those bytes to you, perfectly intact and in order. That's TCP.
