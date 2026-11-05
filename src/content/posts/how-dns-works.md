---
author: Vaibhav Jha
pubDatetime: 2026-08-20T10:00:00Z
title: "How DNS Works: The Internet's Phone Book, Explained"
slug: how-dns-works
featured: false
draft: false
tags:
  - networking
  - dns
  - explainer
description: "Every time you visit a website, DNS silently translates the domain name into an IP address. Here's the full story of how that lookup happens in milliseconds."
---

Every time you type `google.com` into your browser, something remarkable happens in the background — a distributed, hierarchical lookup system resolves that human-friendly name into an IP address like `142.250.80.46`, all in a matter of milliseconds. That system is the **Domain Name System (DNS)**, and it's one of the most critical pieces of infrastructure on the internet.

Let's pull apart how it actually works.

## Table of contents

## What is DNS and why it exists

Computers on the internet identify each other using IP addresses — numerical labels like `93.184.216.34` (IPv4) or `2606:2800:220:1:248:1893:25c8:1946` (IPv6). These are great for machines, but terrible for humans. Nobody wants to memorize `142.250.80.46` to check their email.

DNS solves this by acting as a translation layer — a distributed phone book that maps human-readable domain names (like `example.com`) to their corresponding IP addresses.

But it's more than just a lookup table. DNS is:

- **Distributed** — no single server holds all the records. The data is spread across millions of servers worldwide.
- **Hierarchical** — it's structured like an inverted tree, with authority delegated at each level.
- **Resilient** — it's designed to survive individual server failures without taking down the internet.

DNS was formalized in 1983 by Paul Mockapetris in [RFC 1034](https://datatracker.ietf.org/doc/html/rfc1034) and [RFC 1035](https://datatracker.ietf.org/doc/html/rfc1035). Before DNS, the entire internet relied on a single `HOSTS.TXT` file maintained by the Stanford Research Institute. Every machine on the ARPANET downloaded a copy. As you can imagine, that didn't scale.

## The DNS hierarchy

DNS is organized as a tree, with delegation flowing from the top down. There are four major levels:

### The root zone

At the very top of the DNS tree sits the **root zone**, represented by a dot (`.`). When you write `example.com`, the fully qualified domain name (FQDN) is actually `example.com.` — that trailing dot refers to the root.

The root zone is served by **13 root server clusters**, labeled `a.root-servers.net` through `m.root-servers.net`. But "13 servers" is misleading — thanks to **anycast routing**, there are actually over 1,700 physical instances spread across the globe. Any packet addressed to a root server IP gets routed to the nearest physical instance.

The root servers don't know the IP of `example.com`. What they _do_ know is which servers are authoritative for `.com`, `.org`, `.net`, and every other top-level domain.

### TLD (Top-Level Domain) servers

The next level down manages top-level domains. There are several flavors:

- **Generic TLDs (gTLDs)**: `.com`, `.org`, `.net`, `.info`, `.dev`
- **Country-code TLDs (ccTLDs)**: `.uk`, `.de`, `.jp`, `.in`
- **Infrastructure TLD**: `.arpa` (used for reverse DNS and other infrastructure purposes)
- **Sponsored TLDs**: `.edu`, `.gov`, `.mil`

For `.com`, the TLD servers are managed by Verisign. They don't know the IP of `example.com` either — but they know which **authoritative name servers** are responsible for `example.com`.

### Authoritative name servers

These are the servers that actually hold the DNS records for a domain. When you register `example.com` and configure its DNS, you're populating records on its authoritative name servers. These are often provided by your domain registrar, hosting provider, or a dedicated DNS service like Cloudflare or AWS Route 53.

### Recursive resolvers

Technically not part of the hierarchy, but essential to how DNS works in practice. A **recursive resolver** (also called a recursive DNS server) is the server your computer talks to when it needs to resolve a domain name. It does the hard work of walking the hierarchy on your behalf.

Your ISP provides one by default, but you can configure alternatives like Cloudflare's `1.1.1.1` or Google's `8.8.8.8`.

## DNS record types

DNS doesn't just map names to IP addresses. It supports a variety of record types, each serving a different purpose:

| Record | Purpose | Example |
|--------|---------|---------|
| **A** | Maps a domain to an IPv4 address | `example.com → 93.184.216.34` |
| **AAAA** | Maps a domain to an IPv6 address | `example.com → 2606:2800:220:1:...` |
| **CNAME** | Canonical name — an alias pointing to another domain | `www.example.com → example.com` |
| **MX** | Mail exchange — specifies the mail server for a domain | `example.com → mail.example.com (priority 10)` |
| **NS** | Name server — delegates a zone to authoritative servers | `example.com → ns1.example.com` |
| **TXT** | Arbitrary text data — used for SPF, DKIM, domain verification | `"v=spf1 include:_spf.google.com ~all"` |
| **SOA** | Start of Authority — metadata about the zone (serial number, refresh intervals, admin email) | One per zone |
| **PTR** | Pointer — reverse DNS, maps an IP back to a domain name | `34.216.184.93.in-addr.arpa → example.com` |
| **SRV** | Service locator — specifies host and port for specific services | `_sip._tcp.example.com → sipserver.example.com:5060` |

> [!TIP]
> The `CNAME` record cannot coexist with other record types at the same name. This is why you can't set a `CNAME` on a zone apex (like `example.com` itself) in standard DNS — though many providers offer workarounds like `ALIAS` or `ANAME` records.

## The resolution process, step by step

Let's walk through what happens when you type `blog.example.com` into your browser.

### Step 1: Browser cache

Your browser first checks its own internal DNS cache. If you visited `blog.example.com` recently, the answer might already be stored locally. Chrome, for example, maintains its own DNS cache that you can inspect at `chrome://net-internals/#dns`.

### Step 2: OS stub resolver

If the browser cache misses, the request goes to the operating system's **stub resolver**. This is a lightweight DNS client built into the OS. It checks:

1. The local `/etc/hosts` file (or `C:\Windows\System32\drivers\etc\hosts` on Windows)
2. The OS-level DNS cache (managed by `systemd-resolved` on modern Linux, or the DNS Client service on Windows)

If neither has the answer, the stub resolver forwards the query to the configured **recursive resolver**.

### Step 3: Recursive resolver

The recursive resolver (say, `1.1.1.1`) first checks its own cache. If it has a cached answer that hasn't expired (based on TTL), it returns it immediately.

If not, it begins the **iterative resolution process**:

### Step 4: Query the root servers

The resolver sends a query to one of the 13 root server clusters:

```
"Where can I find blog.example.com?"
```

The root server doesn't know, but it responds with a **referral**:

```
"I don't know, but here are the name servers for .com"
→ a.gtld-servers.net (192.5.6.30)
→ b.gtld-servers.net (192.33.14.30)
...
```

### Step 5: Query the TLD servers

The resolver takes that referral and queries a `.com` TLD server:

```
"Where can I find blog.example.com?"
```

The TLD server responds with another referral:

```
"I don't know, but the authoritative name servers for example.com are:"
→ ns1.example.com (198.51.100.1)
→ ns2.example.com (198.51.100.2)
```

### Step 6: Query the authoritative server

Finally, the resolver queries `ns1.example.com`:

```
"What is the A record for blog.example.com?"
```

The authoritative server has the actual answer:

```
blog.example.com.  300  IN  A  203.0.113.50
```

### Step 7: Response flows back

The recursive resolver caches this answer (honoring the TTL of 300 seconds) and returns it to the stub resolver, which returns it to the browser, which connects to `203.0.113.50`.

The whole process typically takes **20–120 milliseconds** on a cold cache. With caching, subsequent lookups resolve in under a millisecond.

Here's what this looks like visually:

```
Browser → Stub Resolver → Recursive Resolver
                                ↓
                          Root Server (.): "Ask .com servers"
                                ↓
                          TLD Server (.com): "Ask ns1.example.com"
                                ↓
                          Authoritative Server: "Here's the IP: 203.0.113.50"
                                ↓
                          Response flows back up the chain
```

## DNS caching and TTL

Caching is what makes DNS fast. Without it, every single page load would trigger a full walk of the hierarchy.

### TTL (Time to Live)

Every DNS record comes with a **TTL** value in seconds. This tells resolvers how long they can cache the record before they need to re-query the authoritative server.

```
example.com.  3600  IN  A  93.184.216.34
```

That `3600` means resolvers can cache this record for one hour.

### The caching layers

DNS responses are cached at multiple levels:

1. **Browser cache** — Chrome, Firefox, and others maintain their own short-lived DNS caches (usually 60 seconds).
2. **OS cache** — The operating system's resolver daemon caches responses. On Linux with `systemd-resolved`, you can inspect it with `resolvectl statistics`.
3. **Recursive resolver cache** — Your configured DNS server (e.g., `1.1.1.1`) caches responses for up to the full TTL duration.
4. **Upstream caches** — In corporate or ISP networks, there may be additional layers of caching proxies.

> [!NOTE]
> Low TTL values (like 60 seconds) give you more agility — you can change an IP and have it propagate quickly. But they also increase the load on your authoritative servers and can make resolution slightly slower for users. A common default is 300 seconds (5 minutes) for records that change occasionally, and 86400 seconds (24 hours) for stable records.

### Negative caching

DNS also caches _negative_ results. If a domain doesn't exist (NXDOMAIN), that answer is cached too, governed by the `minimum` field in the SOA record. This prevents repeated queries for typos and non-existent domains.

## Using `dig` and `nslookup`

These are the essential tools for debugging DNS. `dig` is the more powerful of the two and is the standard on Linux/macOS.

### Basic `dig` query

```bash
$ dig example.com

; <<>> DiG 9.18.18 <<>> example.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 54321
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; QUESTION SECTION:
;example.com.                   IN      A

;; ANSWER SECTION:
example.com.            3600    IN      A       93.184.216.34

;; Query time: 23 msec
;; SERVER: 1.1.1.1#53(1.1.1.1) (UDP)
;; WHEN: Wed Aug 20 10:00:00 UTC 2026
;; MSG SIZE  rcvd: 56
```

### Query specific record types

```bash
# MX records (mail servers)
$ dig example.com MX

# AAAA records (IPv6)
$ dig example.com AAAA

# NS records (name servers)
$ dig example.com NS

# TXT records (SPF, DKIM, etc.)
$ dig example.com TXT
```

### Trace the full resolution path

This is the killer feature of `dig` — you can watch the entire iterative resolution process:

```bash
$ dig +trace example.com

; <<>> DiG 9.18.18 <<>> +trace example.com
;; global options: +cmd
.                       518400  IN      NS      a.root-servers.net.
.                       518400  IN      NS      b.root-servers.net.
...
;; Received 239 bytes from 1.1.1.1#53(1.1.1.1) in 1 ms

com.                    172800  IN      NS      a.gtld-servers.net.
com.                    172800  IN      NS      b.gtld-servers.net.
...
;; Received 1170 bytes from 198.41.0.4#53(a.root-servers.net) in 24 ms

example.com.            172800  IN      NS      a.iana-servers.net.
example.com.            172800  IN      NS      b.iana-servers.net.
;; Received 639 bytes from 192.5.6.30#53(a.gtld-servers.net) in 15 ms

example.com.            3600    IN      A       93.184.216.34
;; Received 56 bytes from 199.43.135.53#53(a.iana-servers.net) in 11 ms
```

You can see exactly what I described earlier — root → TLD → authoritative, with the response coming back from `a.iana-servers.net`.

### Query a specific DNS server

```bash
# Ask Cloudflare's resolver
$ dig @1.1.1.1 example.com

# Ask Google's resolver
$ dig @8.8.8.8 example.com

# Ask an authoritative server directly
$ dig @a.iana-servers.net example.com
```

### Short output

```bash
$ dig +short example.com
93.184.216.34
```

### Using `nslookup`

`nslookup` is simpler and available on Windows by default:

```bash
$ nslookup example.com
Server:         1.1.1.1
Address:        1.1.1.1#53

Non-authoritative answer:
Name:   example.com
Address: 93.184.216.34
```

> [!TIP]
> The "Non-authoritative answer" label means the response came from a cache rather than directly from the authoritative server. This is normal and expected for most queries.

## DNS over HTTPS (DoH) and DNS over TLS (DoT)

Traditional DNS queries are sent in **plaintext over UDP port 53**. This means anyone on the network path — your ISP, a coffee shop WiFi operator, a government — can see every domain you're looking up. That's a significant privacy concern.

Two protocols aim to fix this:

### DNS over TLS (DoT)

Specified in [RFC 7858](https://datatracker.ietf.org/doc/html/rfc7858), DoT wraps DNS queries in a TLS connection on **port 853**. It's straightforward — same DNS wire format, just encrypted.

**Pros:** Simple, purpose-built, easy for network admins to identify and manage.
**Cons:** Uses a dedicated port (853), so it's easy to block.

### DNS over HTTPS (DoH)

Specified in [RFC 8484](https://datatracker.ietf.org/doc/html/rfc8484), DoH sends DNS queries as HTTPS requests on **port 443** — the same port used for regular web traffic. The DNS query is encoded in the HTTP request body or URL.

**Pros:** Indistinguishable from normal HTTPS traffic, making it very hard to block or inspect.
**Cons:** Some network administrators consider this a downside — it makes it harder to enforce DNS-based content filtering.

### Trying DoH with `curl`

You can actually make a DoH query manually:

```bash
$ curl -s -H 'accept: application/dns-json' \
    'https://cloudflare-dns.com/dns-query?name=example.com&type=A' | jq .

{
  "Status": 0,
  "TC": false,
  "RD": true,
  "RA": true,
  "AD": true,
  "CD": false,
  "Question": [
    {
      "name": "example.com",
      "type": 1
    }
  ],
  "Answer": [
    {
      "name": "example.com",
      "type": 1,
      "TTL": 3600,
      "data": "93.184.216.34"
    }
  ]
}
```

Most modern browsers now support DoH natively. Firefox, Chrome, and Edge all offer it as a setting, and some enable it by default.

## DNS security

DNS was designed in the 1980s without security in mind. Several attack vectors exist, and several defenses have been developed.

### DNS spoofing / cache poisoning

An attacker sends forged DNS responses to a recursive resolver, tricking it into caching incorrect records. If successful, every user of that resolver could be redirected to a malicious server.

The classic attack was demonstrated by Dan Kaminsky in 2008, who showed that the birthday paradox made cache poisoning much easier than previously thought. This led to the rapid deployment of **source port randomization** — making it harder for attackers to guess the transaction ID and port of outgoing queries.

### DNSSEC (Domain Name System Security Extensions)

DNSSEC adds **cryptographic signatures** to DNS records. It doesn't encrypt DNS traffic — it ensures **authenticity and integrity**. When a resolver receives a signed response, it can verify that:

1. The response actually came from the authoritative server (not an attacker).
2. The response hasn't been tampered with in transit.

DNSSEC uses a chain of trust that mirrors the DNS hierarchy:

```
Root zone (signed with root KSK)
  └── .com (signed, DS record in root)
       └── example.com (signed, DS record in .com)
```

You can check DNSSEC validation with `dig`:

```bash
$ dig +dnssec example.com

;; flags: qr rd ra ad;  # The 'ad' flag means Authenticated Data
```

> [!WARNING]
> DNSSEC adoption is still incomplete. While the root zone and most TLDs are signed, many individual domains are not. And misconfigured DNSSEC can cause domains to become completely unreachable — it's one of those things that's worse half-deployed than not deployed at all.

### DNS amplification attacks

DNS can be abused for DDoS attacks. The attacker sends a small query with a **spoofed source IP** (the victim's IP) to an open resolver. The resolver sends a much larger response to the victim. Since DNS responses can be 50–100x larger than queries (especially with DNSSEC or `ANY` queries), this amplifies the attacker's bandwidth significantly.

Mitigation strategies include:
- **Response Rate Limiting (RRL)** on authoritative servers
- **BCP38 / source address validation** at ISPs to prevent IP spoofing
- Disabling open recursive resolvers
- Refusing `ANY` queries (which Cloudflare and others now do)

### DNS rebinding

A more subtle attack where an attacker's DNS server returns a short-TTL record pointing to a public IP, and then switches to a private/internal IP (like `192.168.1.1`). This can trick a browser into making requests to internal network resources, bypassing same-origin policies.

## Popular DNS providers

Your choice of recursive resolver affects privacy, speed, and features.

### Cloudflare — `1.1.1.1` / `1.0.0.1`

- **Focus:** Privacy and speed
- Consistently one of the fastest public resolvers globally
- Supports DoH and DoT
- Committed to purging query logs within 24 hours (audited by KPMG)
- Also offers `1.1.1.2` (malware blocking) and `1.1.1.3` (malware + adult content blocking)

### Google — `8.8.8.8` / `8.8.4.4`

- **Focus:** Reliability and global reach
- One of the oldest public DNS services (launched 2009)
- Supports DoH and DoT
- Extensive anycast network
- Google does log queries, though they anonymize the data

### Quad9 — `9.9.9.9`

- **Focus:** Security
- Automatically blocks known malicious domains using threat intelligence feeds
- Non-profit, based in Switzerland (strong privacy laws)
- Supports DNSSEC validation by default

### Your ISP's DNS

Your ISP provides DNS resolvers by default. They work, but they're often:
- **Slower** — less investment in infrastructure and anycast
- **Less private** — ISPs may log and sell your DNS query history
- **More prone to interference** — some ISPs hijack NXDOMAIN responses to show ad-filled search pages
- **Less secure** — often lacking DoH/DoT support

You can test resolver performance yourself:

```bash
# Time a query to Cloudflare
$ dig @1.1.1.1 example.com | grep "Query time"
;; Query time: 4 msec

# Time a query to your ISP's resolver
$ dig @$(cat /etc/resolv.conf | grep nameserver | head -1 | awk '{print $2}') example.com | grep "Query time"
;; Query time: 28 msec
```

> [!TIP]
> The tool [DNS Benchmark](https://www.grc.com/dns/benchmark.htm) (Windows) or `namebench` (cross-platform) can test multiple resolvers against your actual location and usage patterns.

## Fun facts about DNS

Let me close with some things about DNS that I find genuinely delightful:

1. **The entire root zone file is only ~2 MB.** The root zone contains roughly 1,500 TLD delegations. You can download it from [IANA](https://www.iana.org/domains/root/files). It's surprisingly small for something that underpins the entire internet.

2. **The 13 root server limit is a UDP constraint.** DNS traditionally uses UDP, and a single UDP packet can hold 512 bytes. Fitting 13 root server names and their IPv4 addresses into that space was the practical maximum. With EDNS0 (extended DNS) allowing larger packets, we could theoretically add more, but there's been no need.

3. **There's a DNS record type for geographic location.** The `LOC` record ([RFC 1876](https://datatracker.ietf.org/doc/html/rfc1876)) lets you specify the latitude, longitude, and altitude of a server. Almost nobody uses it.

4. **DNS was used to distribute SSH fingerprints.** The `SSHFP` record stores SSH public key fingerprints, allowing SSH clients to verify a host key via DNS (with DNSSEC) instead of asking "Are you sure you want to continue connecting?"

5. **The `.` TLD technically exists.** The root zone itself is a valid (if empty) zone. Some resolvers will actually respond to queries for just `.`.

6. **DNS over QUIC (DoQ) is a thing now.** [RFC 9250](https://datatracker.ietf.org/doc/html/rfc9250) defines DNS over QUIC, combining the privacy benefits of encryption with the performance advantages of QUIC's 0-RTT connection establishment.

7. **The longest valid domain name is 253 characters.** Each label (section between dots) can be up to 63 characters, and the total FQDN can be up to 253 characters. People have definitely tested this limit.

8. **DNS was once used to measure the size of the internet.** Researchers queried reverse DNS for every possible IPv4 address to estimate how many hosts were online. This is called an "Internet Census" and the ethics of it are... debated.

---

DNS is one of those systems that's easy to take for granted precisely because it works so well. It was designed in 1983, scales to billions of queries per day, and still runs on fundamentally the same architecture. Understanding how it works — really works, at every layer — makes you a better engineer, a better debugger, and gives you a deeper appreciation for the infrastructure we all depend on.

Next time a website doesn't load, before you blame the server, try `dig`. You might be surprised where the problem actually lies.
