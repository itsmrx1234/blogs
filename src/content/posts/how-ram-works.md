---
author: Vaibhav Jha
pubDatetime: 2026-09-10T10:00:00Z
title: "How RAM Works: From Capacitors to Cache Lines"
slug: how-ram-works
featured: true
draft: false
tags:
  - hardware
  - computer-architecture
  - explainer
description: "RAM is the fastest storage your CPU can talk to (besides cache). But how does a tiny capacitor store a bit, and how does the CPU read millions of them in nanoseconds?"
---

## Table of contents

## Why We Need RAM — The Memory Hierarchy

Your CPU is _fast_. A modern processor can execute billions of instructions per second. But there's a problem: it needs data to work on, and not all storage is created equal.

If the CPU had to fetch every piece of data from your SSD or hard drive, it would spend most of its time _waiting_. That's like being the fastest reader in the world but having to walk to a library across town every time you need the next page. You'd spend all your time walking, not reading.

This is why computers use a **memory hierarchy** — a layered system of storage that trades capacity for speed:

| Level | Storage | Typical Size | Typical Latency | Cost/GB |
|-------|---------|-------------|-----------------|---------|
| 0 | CPU Registers | ~1 KB | ~0.3 ns | — |
| 1 | L1 Cache | 32–128 KB | ~1 ns | $$$$$ |
| 2 | L2 Cache | 256 KB–1 MB | ~3–5 ns | $$$$ |
| 3 | L3 Cache | 4–64 MB | ~10–20 ns | $$$ |
| 4 | **RAM (DRAM)** | **8–128 GB** | **~50–100 ns** | $$ |
| 5 | SSD (NAND Flash) | 256 GB–4 TB | ~25–100 μs | $ |
| 6 | HDD | 1–20 TB | ~5–10 ms | ¢ |

The idea is simple: keep the most frequently accessed data in the fastest (and smallest) storage, and let less-used data spill into slower, cheaper, bigger storage.

RAM — specifically DRAM — sits right in the sweet spot. It's orders of magnitude faster than an SSD, and orders of magnitude cheaper and larger than cache. It's the **working memory** of your computer: the scratch pad where the OS loads programs, where your browser keeps its open tabs, and where your game stores the textures it might need in the next frame.

But how does it actually work? Let's go deep.

## SRAM vs DRAM — Two Flavors of RAM

"RAM" stands for Random Access Memory — meaning you can read any location in roughly the same amount of time (unlike a tape drive, where you have to physically wind to the right spot). But there are two fundamentally different ways to build RAM:

### SRAM — Static RAM

SRAM stores each bit using a **flip-flop** circuit — typically six transistors wired together in a feedback loop. As long as power is applied, the circuit holds its state. No refreshing needed. No moving parts (electrically speaking).

Think of SRAM like a light switch. Flip it up and it stays up. Flip it down and it stays down. It holds its state perfectly until you change it.

**Pros:** Extremely fast (~1 ns access time), doesn't need refreshing.
**Cons:** Expensive, physically large (6 transistors per bit), runs hot.

This is why SRAM is used for **CPU caches** (L1, L2, L3) — where speed is everything and you only need a few megabytes.

### DRAM — Dynamic RAM

DRAM stores each bit using just **one transistor and one capacitor**. The capacitor either holds a charge (1) or doesn't (0). This is dramatically simpler — and cheaper — than SRAM, so you can pack billions of cells onto a single chip.

But there's a catch: capacitors leak. The charge slowly drains away, and if you don't do anything about it, your data disappears. DRAM is called "dynamic" precisely because it requires constant **refreshing** to keep data alive.

Think of DRAM like a tiny bucket with a small hole in the bottom. You fill it with water to represent a `1`, or leave it empty for a `0`. But the water slowly leaks out, so every few milliseconds someone has to come by, check the bucket, and top it off if it's supposed to be full.

**Pros:** Cheap, dense (1 transistor + 1 capacitor per bit), high capacity.
**Cons:** Slower than SRAM, requires constant refreshing, destructive reads (more on this soon).

When people say "RAM" in everyday conversation — "I've got 16 gigs of RAM" — they mean DRAM. That's what your DIMMs are made of.

## How a DRAM Cell Works

Let's zoom in on a single DRAM cell — the smallest unit of storage. It's beautifully simple:

```
        Bit Line
           |
           |
    ───┤ T ├───┤├── GND
           |    C
       Word Line
```

- **C** is a tiny capacitor (we're talking femtofarads — 10⁻¹⁵ farads).
- **T** is an access transistor (acts as a switch/gate).
- The **Word Line** controls the gate of the transistor — it selects this cell.
- The **Bit Line** is the data highway — it carries data in or out.

### Writing a bit

To write a `1`, the memory controller:

1. Drives the **bit line** high (charges it up).
2. Activates the **word line**, which turns on transistor T.
3. Charge flows from the bit line into the capacitor C.
4. The word line is deactivated, trapping the charge in C.

To write a `0`, the bit line is driven low, and the capacitor is drained.

### Reading a bit (the destructive part)

Reading is more interesting — and sneakier than you might expect.

1. The bit line is **precharged** to exactly half the supply voltage (Vdd/2). This is the reference point.
2. The word line is activated, connecting the capacitor to the bit line.
3. If the capacitor was charged (storing a `1`), the tiny amount of charge flows _out_ of the capacitor and nudges the bit line voltage _slightly_ above Vdd/2.
4. If the capacitor was empty (storing a `0`), charge flows _into_ the capacitor from the bit line, pulling it _slightly_ below Vdd/2.
5. A **sense amplifier** detects this incredibly small voltage difference and amplifies it to a full `1` or `0`.

> [!WARNING]
> Reading a DRAM cell is **destructive**. The act of reading drains the capacitor. After every read, the data must be **written back** to the cell. This is one reason DRAM is slower than SRAM — every read is secretly a read-then-write.

The voltage difference the sense amplifier detects is on the order of millivolts — often less than 100 mV. This is one of the most delicate operations in all of digital electronics, happening billions of times per second across your RAM modules.

## The Refresh Problem

Remember the leaky bucket analogy? A DRAM capacitor can hold its charge for only a few milliseconds before it decays below the threshold where the sense amplifier can reliably distinguish a `1` from a `0`.

To prevent data loss, the memory controller periodically performs a **refresh cycle**:

1. It reads each row of cells (which triggers the sense amplifiers).
2. The sense amplifiers restore the charges to full strength.
3. The data is written back.

The JEDEC standard requires that every cell in a DRAM chip be refreshed at least once every **64 milliseconds** (32 ms in some high-temp environments). With millions of rows, this means the controller is constantly performing refresh operations in the background.

> [!NOTE]
> Refresh operations temporarily block normal read/write access to the rows being refreshed. This is one of the hidden sources of latency in DRAM. Modern controllers try to be clever about scheduling refreshes during idle periods, but on a busy system, refresh overhead can reduce effective bandwidth by 5–10%.

This is also why DRAM loses all its data when you power off your computer — no power means no refresh, and the capacitors drain completely within seconds.

## Memory Addressing — Rows, Columns, Banks, and Ranks

A DRAM chip isn't just a flat array of cells. It's organized in a hierarchy designed for efficient access:

### The 2D Array: Rows and Columns

At the lowest level, cells are arranged in a 2D grid. A typical DRAM array might have **65,536 rows** and **8,192 columns**. Each intersection is one cell — one bit.

When you access memory, you first select a **row** (using the Row Address Strobe, or RAS), and then a **column** within that row (using the Column Address Strobe, or CAS). This two-step process exists because sending the full address at once would require too many pins — multiplexing the address across two cycles cuts the pin count in half.

### Banks

A single DRAM chip is divided into multiple **banks** — typically 8 or 16 in modern DDR4/DDR5. Each bank has its own row buffer and sense amplifiers, and can operate somewhat independently.

This is critical for performance: while one bank is busy with a precharge or refresh, another bank can be serving a read or write. The memory controller interleaves operations across banks to keep the data flowing.

### Ranks

A **rank** is a group of DRAM chips that work together to fill the full width of the memory bus. For example, if your memory bus is 64 bits wide and each DRAM chip has an 8-bit interface, you need 8 chips working in parallel to form one rank.

A typical DIMM (the stick of RAM you plug into your motherboard) has one or two ranks. Dual-rank DIMMs give the controller more banks to interleave across, which can improve bandwidth — but they're also slightly harder for the controller to manage.

### The Full Address

So a complete DRAM address looks something like:

```
[Channel] → [DIMM] → [Rank] → [Bank] → [Row] → [Column]
```

Each level of this hierarchy narrows down to a specific cell. The memory controller translates the CPU's flat physical address into this hierarchical address.

## How a Memory Read Actually Works

Let's walk through what happens when your CPU needs to read a byte from DRAM. This is where the timing parameters you see on RAM spec sheets — CAS latency, tRCD, tRP, tRAS — come from.

### Step 1: Row Activation (tRCD)

The controller sends the **row address** to the appropriate bank. The word line for that row is activated, connecting all capacitors in the row to their bit lines. The sense amplifiers fire, reading and amplifying every bit in the row simultaneously.

The entire row — typically 8 KB — is now loaded into the **row buffer** (also called the sense amplifier array).

The time this takes is called **tRCD** (RAS-to-CAS Delay), typically **13–22 clock cycles** in DDR4/DDR5.

### Step 2: Column Read (CAS Latency / CL)

Now the controller sends the **column address**. The bank selects the appropriate bits from the row buffer and sends them out.

This delay is the famous **CAS Latency (CL)** — the number everyone argues about on forums. In DDR4, CL ranges from about 14 to 22. In DDR5, it's 28 to 40 (but at a higher clock speed, so absolute latency is similar).

### Step 3: Data Transfer

The data travels from the DRAM chip, through the memory bus, to the memory controller on the CPU. In DDR memory, data is transferred on **both the rising and falling edges** of the clock signal (hence "Double Data Rate"), so a 1600 MHz DDR bus actually transfers data at 3200 MT/s (megatransfers per second).

### Step 4: Precharge (tRP)

If the next access targets a _different_ row in the same bank, the current row must be closed first. This is the **precharge** operation — the bit lines are reset to Vdd/2 in preparation for the next row activation.

This takes **tRP** (Row Precharge time) — another 13–22 clock cycles.

### Row Hit vs Row Miss

This is why **access patterns matter enormously**:

- **Row hit**: If the next read targets the same row that's already in the row buffer, you skip both the activation and precharge steps. You just send the new column address. This is fast — just CL.
- **Row miss**: If you need a different row in the same bank, you have to precharge + activate + read. That's tRP + tRCD + CL. Much slower.
- **Row conflict**: The worst case — a different row is active, so you must precharge, wait, activate the new row, wait, then read.

This is one reason sequential memory access is so much faster than random access in DRAM. Sequential access tends to hit the same row repeatedly.

> [!TIP]
> The **tRAS** (Row Active Time) parameter specifies the minimum time a row must stay active before it can be precharged. It's essentially a lower bound: tRAS ≥ tRCD + CL + some margin. You can think of it as a safety timer ensuring the row has been fully read before you slam it shut.

### Putting the Timing Together

The full latency for a row miss read is:

```
Total = tRP + tRCD + CL
```

For a DDR4-3200 CL16 module:

- Clock period = 1 / 1600 MHz = 0.625 ns
- tRP = 16 cycles × 0.625 ns = 10 ns
- tRCD = 16 cycles × 0.625 ns = 10 ns
- CL = 16 cycles × 0.625 ns = 10 ns
- **Total ≈ 30 ns** (best case row miss, not counting bus transfer and controller overhead)
- **Real-world total: ~50–70 ns** from CPU request to data arrival

## DDR Explained — Double Data Rate

The "DDR" in DDR4 or DDR5 stands for **Double Data Rate**. In the original SDR (Single Data Rate) SDRAM, data was transferred once per clock cycle — on the rising edge. DDR doubles this by transferring data on _both_ the rising and falling edges.

Here's how the generations stack up:

| Generation | Year | Voltage | Max Speed | Prefetch | Key Innovation |
|-----------|------|---------|-----------|----------|----------------|
| DDR1 | 2000 | 2.5V | 400 MT/s | 2n | Double data rate |
| DDR2 | 2003 | 1.8V | 1066 MT/s | 4n | Higher clock, lower voltage |
| DDR3 | 2007 | 1.5V | 2133 MT/s | 8n | Even lower voltage, XMP profiles |
| DDR4 | 2014 | 1.2V | 3200 MT/s | 8n | Bank groups, higher density |
| DDR5 | 2020 | 1.1V | 6400+ MT/s | 16n | On-die ECC, two 32-bit channels per DIMM |

### What's "Prefetch"?

Each DDR generation increases the **prefetch buffer** — the number of bits fetched from the internal memory array in a single access. DDR1 fetches 2 bits per cycle (2n prefetch), DDR4 fetches 8 (8n), and DDR5 fetches 16 (16n).

This is how DDR speeds keep increasing even though the internal DRAM core hasn't gotten _that_ much faster. The trick is to fetch more data per internal cycle and then burst it out at the higher I/O rate.

Think of it like a warehouse worker. The warehouse shelves are accessed at the same speed, but each generation gives the worker a bigger cart. DDR1's worker grabs 2 items per trip to the shelf. DDR5's worker grabs 16. The shelf hasn't gotten faster — but the loading dock is shipping items out faster because there's more ready to go.

### DDR5's Big Changes

DDR5 introduced two major architectural changes:

1. **Two independent 32-bit channels per DIMM** (instead of one 64-bit channel). This improves concurrency — the two sub-channels can serve different requests simultaneously.
2. **On-die ECC**: Each DRAM chip performs internal error correction. This isn't the same as the "ECC RAM" you buy for servers (which adds a whole extra chip for full-rank ECC), but it helps compensate for the smaller, leakier capacitors that come with higher density.

## Memory Channels — Bandwidth Through Parallelism

A single memory channel is one 64-bit-wide data path between the CPU's memory controller and a DIMM slot. But most modern systems support multiple channels:

- **Single channel**: One 64-bit path. Bandwidth = transfer rate × 8 bytes.
- **Dual channel**: Two independent 64-bit paths running in parallel. Double the bandwidth.
- **Quad channel**: Four paths (common on HEDT and server platforms).
- **Six/eight channel**: Server and HPC territory (Intel Xeon, AMD EPYC).

For DDR4-3200 in dual channel:

```
Bandwidth = 3200 MT/s × 8 bytes × 2 channels = 51.2 GB/s
```

> [!IMPORTANT]
> To enable dual channel, you typically need to install DIMMs in matching pairs in the correct slots (usually alternating slots, labeled A1/B1 or A2/B2 — check your motherboard manual). If you install a single stick, you're leaving half your memory bandwidth on the table. This is one of the most common, easily avoidable performance mistakes in PC building.

Channel configuration affects **bandwidth**, not latency. Whether you have one channel or four, the time to fetch a single cache line is roughly the same. But when multiple cores are hammering memory simultaneously, more channels prevent bottlenecks.

## How the CPU Talks to RAM

In the old days (pre-2008 for Intel, pre-2003 for AMD), the CPU talked to RAM through a separate chip on the motherboard called the **Northbridge**. The CPU sent memory requests over the Front Side Bus (FSB) to the Northbridge, which then communicated with the DRAM.

Modern processors have an **integrated memory controller (IMC)** — the memory controller lives right on the CPU die. This eliminates the Northbridge bottleneck and significantly reduces latency.

### The Memory Bus

The physical connection between the IMC and the DRAM modules consists of:

- **Data lines**: 64 bits wide per channel (plus 8 ECC bits if applicable).
- **Address lines**: Multiplexed row/column address lines.
- **Command lines**: RAS#, CAS#, WE# (write enable), and others.
- **Clock lines**: Differential clock pairs for timing.

All signals are transmitted in parallel on the PCB traces that run from the CPU socket to the DIMM slots. Signal integrity is critical — this is why memory overclocking is sensitive to trace length, and why DIMM slots closer to the CPU can sometimes run at higher speeds.

### Latency Breakdown

When the CPU issues a memory request, here's roughly where the time goes:

```
CPU core → L1 miss → L2 miss → L3 miss (10-20 ns so far)
→ Memory controller queuing and scheduling (~5 ns)
→ Command transmission to DRAM (~2-5 ns)
→ tRCD + CL in the DRAM chip (~20-30 ns)
→ Data transmission back (~2-5 ns)
→ Memory controller to CPU core (~5 ns)
─────────────────────────────────
Total: ~50-100 ns
```

This is why you'll see benchmarks report RAM latency in the 60–90 ns range for most desktop systems. It's not _just_ the CAS latency — it's the entire round trip.

### The CAS Latency Paradox

Here's something that confuses a lot of people: DDR5 has _higher_ CAS latency numbers than DDR4 (CL36–40 vs CL14–18). Does that mean DDR5 is slower?

No. CAS latency is measured in **clock cycles**, and DDR5 runs at a much higher clock speed. What matters is the **absolute latency in nanoseconds**:

```
DDR4-3200 CL16:  16 / 1600 MHz = 10.0 ns (CAS only)
DDR5-6400 CL36:  36 / 3200 MHz = 11.25 ns (CAS only)
```

The CAS latency in real time is nearly identical. But DDR5's higher bandwidth means it can move _much_ more data per second once the access begins. For bandwidth-heavy workloads, DDR5 wins easily. For latency-sensitive workloads, it's closer to a wash — at least until DDR5 matures with tighter timings.

## Virtual Memory and Page Tables

So far, we've talked about DRAM as a physical device. But programs don't interact with physical RAM addresses directly. They use **virtual memory** — an abstraction layer managed by the OS and CPU hardware.

### Why Virtual Memory?

Three problems virtual memory solves:

1. **Isolation**: Each process gets its own private address space. Process A can't accidentally (or maliciously) read Process B's memory.
2. **Abstraction**: Programs don't need to know or care where in physical RAM their data lands. They always see a clean, contiguous address space starting from 0.
3. **Overcommit**: The OS can promise more memory than physically exists, using disk (swap/page file) as overflow.

### How It Works: Page Tables

The virtual address space is divided into fixed-size **pages** (typically 4 KB on x86, though 2 MB and 1 GB "huge pages" are also supported). Physical RAM is divided into same-sized **frames**.

The OS maintains a **page table** for each process — a mapping from virtual page numbers to physical frame numbers. When the CPU accesses a virtual address:

```
Virtual Address:  [Virtual Page Number | Offset within page]
                         ↓ (page table lookup)
Physical Address: [Physical Frame Number | Offset within page]
```

### The TLB — Caching the Translation

Looking up the page table for every single memory access would be catastrophically slow (the page table itself is stored in RAM!). To avoid this, the CPU has a **Translation Lookaside Buffer (TLB)** — a small, fast cache that stores recent virtual-to-physical translations.

A TLB hit resolves the translation in ~1 cycle. A TLB miss requires a **page table walk** — traversing the multi-level page table structure in memory — which can cost 10–100+ cycles depending on how many levels are cached.

On x86-64, page tables are **4 levels deep** (PML4 → PDPT → PD → PT), and with 5-level paging (used for >256 TB address spaces), it's 5 levels. Each level is a memory access, so a full TLB miss is _expensive_.

> [!TIP]
> This is one reason **huge pages** (2 MB or 1 GB) can significantly improve performance for large-memory workloads like databases and VMs. Fewer pages means fewer TLB entries needed to cover the same memory range, which means fewer TLB misses. If you're running a Redis instance with 64 GB of data, switching from 4 KB pages to 2 MB huge pages can reduce TLB misses by orders of magnitude.

### Page Faults

When a process accesses a page that isn't currently in physical RAM (maybe it was swapped to disk, or was never allocated), the CPU generates a **page fault**. The OS handles this by:

1. Finding a free physical frame (or evicting an existing page to swap).
2. Loading the requested data from disk into the frame.
3. Updating the page table.
4. Restarting the faulting instruction.

A page fault that requires disk I/O is called a **major page fault** and takes milliseconds — an eternity in CPU time. This is why running out of RAM and hitting swap is so devastating to performance.

## Real-World Impact — Why RAM Speed Matters

### Gaming

Modern games load enormous amounts of data — textures, meshes, game state, AI data — into RAM. Memory **bandwidth** affects how quickly the GPU can be fed with texture data, and how quickly the CPU can process game logic that spans large data structures.

In practice, the biggest gaming impact comes from:

- **Dual channel vs single channel**: Running in single channel can cost 10–30% in CPU-bound scenarios. This is the most impactful and cheapest upgrade.
- **Memory speed**: Going from DDR4-2400 to DDR4-3600 might gain 5–15% in CPU-limited games (especially on AMD Ryzen, which ties its Infinity Fabric clock to memory speed).
- **Capacity**: 16 GB is the practical minimum in 2026. Some titles with heavy modding or at ultra settings can push past this.

### Compiling

Compilers are surprisingly memory-hungry. Large C++ projects (think Chromium, LLVM, Unreal Engine) create millions of intermediate objects, resolve millions of symbols, and perform link-time optimization across gigabytes of object files.

More RAM means the OS can keep more of the project's source files, object files, and compiler data structures in memory instead of paging to disk. RAM speed matters less here than RAM _capacity_ — but on massive builds, faster memory can shave minutes off compile times.

### Servers and Databases

This is where RAM matters most. Databases like PostgreSQL and MySQL keep their hot data sets in memory (the buffer pool). A database that fits entirely in RAM serves queries in microseconds. Once it spills to disk, query latency jumps to milliseconds.

For servers:

- **ECC RAM** is essential — a single bit flip in a database can corrupt data silently. ECC (Error-Correcting Code) memory adds an extra chip per rank that can detect and correct single-bit errors, and detect double-bit errors.
- **Capacity** is king — more RAM means more of your dataset stays in memory.
- **Channel count** matters at scale — a server with 8 memory channels can sustain far more concurrent memory-intensive queries than one with 2 channels.
- **Registered (RDIMM) and Load-Reduced (LRDIMM)** modules allow servers to support higher capacities per channel by adding a buffer chip to the DIMM.

### The Bottom Line

RAM isn't glamorous. It doesn't have the visible performance impact of a GPU upgrade or the satisfying snappiness of an SSD. But it's the silent backbone of your system. Too little of it, and everything grinds to a halt. Misconfigured (single channel, slow timings), and you're leaving real performance on the table.

The next time you look at a stick of RAM, remember: inside that green PCB are billions of microscopic capacitors, each one holding a tiny puddle of electrons, each one slowly leaking, each one being frantically topped off sixty-four times a second — all so your CPU can pretend it has fast, infinite memory. It's one of the most elegant hacks in all of computing.
