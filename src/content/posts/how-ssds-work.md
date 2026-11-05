---
author: Vaibhav Jha
pubDatetime: 2026-07-22T10:00:00Z
title: "How SSDs Work: No Moving Parts, Pure Speed"
slug: how-ssds-work
featured: false
draft: false
tags:
  - hardware
  - storage
  - explainer
description: "SSDs replaced spinning hard drives and changed computing forever. But how do they store data without any moving parts? Let's look inside the flash memory cells."
---

If you've ever swapped a hard drive for an SSD, you know the feeling. Your machine boots in seconds. Apps open instantly. The whole computer feels _new_. But how does a chip with zero moving parts outperform a precision mechanical device that spins a magnetic platter at 7200 RPM?

The answer lives in physics, clever engineering, and a whole lot of firmware tricks. Let's crack open an SSD and see what's really going on.

## Table of contents

## Why SSDs Exist: The HDD Problem

Hard disk drives store data on spinning magnetic platters. A read/write head floats nanometers above the surface, seeking to the right track and waiting for the right sector to rotate underneath it. This works — it's worked since the 1950s — but it has fundamental physics problems:

- **Seek time**: The head has to physically move to a different track. That takes 5–15 milliseconds on average. Sounds fast, but your CPU executes billions of instructions per second. To the processor, waiting for a disk seek feels like waiting in line at the DMV.
- **Rotational latency**: Even after the head arrives at the right track, it has to wait for the correct sector to spin around. At 7200 RPM, that's an average wait of ~4.2 ms.
- **Mechanical failure**: Moving parts wear out. Drop a running laptop and that read/write head can crash into the platter. Game over.

Random I/O is the killer. An HDD can do maybe 100–200 random IOPS (I/O operations per second). A modern NVMe SSD? Hundreds of thousands. That's not a percentage improvement — it's three orders of magnitude.

SSDs solve this by eliminating moving parts entirely. Data is stored as electrical charge trapped inside transistors. No seeking, no spinning, no waiting. Just electrons.

## NAND Flash Memory: Storing Bits as Charge

At the heart of every SSD is **NAND flash memory**. The fundamental storage element is a **floating gate transistor** (or in modern designs, a **charge trap** transistor). Here's how it works:

A normal MOSFET transistor has a gate that controls whether current flows between the source and drain. A floating gate transistor adds a second gate — the **floating gate** — sandwiched between the control gate and the channel, surrounded by insulating oxide layers.

```
        Control Gate
    ┌───────────────────┐
    │   Oxide Layer     │
    ├───────────────────┤
    │   Floating Gate   │  ← Charge trapped here = your data
    ├───────────────────┤
    │   Tunnel Oxide    │
    ├───────────────────┤
    │     Channel       │
    └───────────────────┘
   Source              Drain
```

**Writing** (programming) a cell means applying a high voltage to the control gate. This forces electrons through the tunnel oxide via quantum tunneling (Fowler-Nordheim tunneling) and traps them on the floating gate. Once the voltage is removed, the electrons are stuck — the insulating oxide keeps them there for years, even without power.

**Reading** the cell means applying a reference voltage to the control gate and checking if current flows through the channel. If the floating gate is charged (full of electrons), it partially shields the channel from the control gate's field, raising the transistor's threshold voltage. The cell won't turn on at the reference voltage — that's a `0`. If the floating gate is empty, the transistor turns on normally — that's a `1`.

**Erasing** means applying a high voltage in the opposite direction, pulling the electrons back out through the tunnel oxide. This is a destructive, block-level operation — more on that in a moment.

> [!NOTE]
> The term "NAND" comes from the circuit topology. Flash cells are wired in series like a NAND logic gate, which makes the array denser (fewer wires per cell) at the cost of slightly slower reads compared to NOR flash (where cells are in parallel). SSDs use NAND because density and cost-per-bit matter more than raw read latency for storage.

## SLC vs MLC vs TLC vs QLC: The Bits-Per-Cell Tradeoff

Here's where it gets interesting. That floating gate doesn't just store "charged" or "not charged." By carefully controlling _how much_ charge is on the gate, you can distinguish between multiple voltage levels — and thus store more than one bit per cell.

| Type | Bits/Cell | Voltage Levels | Endurance (P/E Cycles) | Relative Speed | Relative Cost |
|------|-----------|---------------|----------------------|---------------|--------------|
| **SLC** (Single-Level Cell) | 1 | 2 | ~100,000 | Fastest | Most Expensive |
| **MLC** (Multi-Level Cell) | 2 | 4 | ~10,000 | Fast | Moderate |
| **TLC** (Triple-Level Cell) | 3 | 8 | ~3,000 | Moderate | Cheap |
| **QLC** (Quad-Level Cell) | 4 | 16 | ~1,000 | Slowest | Cheapest |

The physics here is intuitive: the more voltage levels you cram into the same physical range, the narrower the margin between each level. Reading QLC requires distinguishing between 16 different charge states — the sensing circuitry needs to be incredibly precise, and the margins for error are razor-thin.

This has three consequences:

1. **Slower reads and writes**: More precision takes more time. TLC and QLC cells need multiple read passes at different reference voltages to decode all the bits.
2. **Lower endurance**: Every program/erase cycle slightly degrades the tunnel oxide. When your margins are already tight (QLC), fewer cycles of degradation push you past the point where levels become indistinguishable.
3. **Lower cost per GB**: You're storing 4x the data in the same silicon area compared to SLC. This is why consumer SSDs are affordable.

> [!TIP]
> Many modern consumer SSDs use TLC NAND but include a small **SLC cache** — a portion of the flash that's operated in SLC mode for burst writes. This gives you SLC-like speed for the first few GBs of writes. Once the cache fills, write speed drops to native TLC rates. This is why you might see write speed benchmarks that look great at first but tank during sustained writes.

## Pages, Blocks, and the Erase-Before-Write Problem

NAND flash has a hierarchical organization:

- **Cell**: Stores 1–4 bits (depending on SLC/MLC/TLC/QLC)
- **Page**: The smallest unit you can **read** or **write**. Typically 4–16 KB.
- **Block**: The smallest unit you can **erase**. Typically 128–512 pages, so 1–4 MB.

Here's the critical constraint that makes SSD firmware so complex:

**You can write to a clean (erased) page. But you cannot overwrite a page that already has data. You must erase the entire block first.**

Think of it like a whiteboard. You can write on a clean section, but you can't erase a single word — you have to wipe the entire board clean and rewrite everything you wanted to keep.

This creates a cascade of complexity:

1. If the OS wants to update a 4 KB file, the SSD can't just overwrite those 4 KB in place.
2. Instead, it writes the new data to a **different, clean page** and marks the old page as **invalid** (stale).
3. Eventually, blocks accumulate a mix of valid and invalid pages.
4. To reclaim space, the SSD must copy all valid pages from a block to a new location, then erase the old block.

This is the fundamental reason SSDs need sophisticated firmware. The simple act of "overwrite this file" turns into a complex dance of read-copy-write-erase.

## The Flash Translation Layer (FTL): The Grand Illusion

Your operating system thinks it's talking to a simple block device: "Write these bytes at logical block address 42." It has no idea about pages, blocks, erase cycles, or any of the NAND complexity. That illusion is maintained by the **Flash Translation Layer (FTL)** — firmware running on the SSD's controller.

The FTL maintains a **mapping table** from **Logical Block Addresses (LBAs)** (what the OS sees) to **Physical Page Addresses (PPAs)** (where data actually lives in NAND).

```
┌─────────────────┐        ┌─────────────────┐
│   OS / Host     │        │   NAND Flash     │
│                 │        │                  │
│  LBA 0 ────────┼──FTL──▶│  Block 5, Page 3 │
│  LBA 1 ────────┼──FTL──▶│  Block 2, Page 7 │
│  LBA 2 ────────┼──FTL──▶│  Block 8, Page 1 │
│  ...           │        │  ...             │
└─────────────────┘        └─────────────────┘
```

When the OS writes to LBA 42, the FTL:
1. Finds a clean physical page
2. Writes the data there
3. Updates the mapping: LBA 42 → new physical page
4. Marks the old physical page (if any) as invalid

This **log-structured write** approach means data for the same LBA bounces around the physical NAND over time. The mapping table is the source of truth — and losing it would be catastrophic (it's why SSDs have capacitors to flush the table to NAND during power loss).

> [!WARNING]
> The FTL mapping table for a 1 TB SSD can consume over 1 GB of DRAM. This is why high-performance SSDs have dedicated DRAM chips on the PCB. Budget "DRAM-less" SSDs use a smaller mapping cache in the controller's SRAM and page in portions of the table from NAND on demand — which works but adds latency for random reads to unmapped regions. Some use HMB (Host Memory Buffer), borrowing a small chunk of the system's RAM instead.

## Garbage Collection and Write Amplification

Over time, blocks become fragmented — a mix of valid and invalid pages scattered throughout. The SSD needs to reclaim those invalid pages to create clean blocks for future writes. This process is called **garbage collection (GC)**.

Here's how it works:

1. The GC algorithm selects a block with a high ratio of invalid pages (a "victim block").
2. It reads all **valid** pages from that block.
3. It writes those valid pages to a new, clean block.
4. It erases the victim block, making it available for new writes.

The problem? Those valid page copies are **extra writes that the host never asked for**. This is **write amplification (WA)**:

```
                   Physical Data Written to NAND
Write Amplification = ────────────────────────────────
                   Logical Data Written by Host
```

A WA of 1.0 is perfect — every byte the host writes corresponds to exactly one byte written to NAND. In practice, WA ranges from 1.1 (well-optimized, lightly used drive) to 3.0+ (worst case, nearly full drive with random writes).

Write amplification matters because:
- It **reduces write performance** (the controller is busy doing background housekeeping)
- It **accelerates NAND wear** (more physical writes = more P/E cycles consumed)

This is why you should **never fill an SSD to 100% capacity**. A full drive has no clean blocks for the FTL to use, forcing aggressive garbage collection that tanks performance and accelerates wear.

## Wear Leveling: Spreading the Pain

Some LBAs are "hot" — written to constantly (logs, temp files, swap space). Others are "cold" — written once and rarely touched (OS files, archived data). Without intervention, hot data would burn through the P/E cycles of a small subset of blocks while cold blocks sit nearly pristine.

**Wear leveling** is the FTL's strategy to distribute writes evenly across all blocks:

- **Dynamic wear leveling**: When writing new data, the FTL picks the block with the lowest erase count among the free blocks. This ensures new writes are spread around. Simple and effective, but doesn't help with cold data sitting on low-wear blocks.

- **Static wear leveling**: The FTL periodically moves cold data from low-wear blocks to high-wear blocks, freeing up the low-wear blocks for future hot writes. This is more aggressive but ensures truly even distribution.

The goal: every block in the SSD reaches its endurance limit at approximately the same time, maximizing total drive lifespan.

## The TRIM Command: Help From the OS

Here's a subtle problem: when you delete a file, the OS marks the space as free in its filesystem metadata. But the SSD doesn't know that. From the SSD's perspective, those pages still contain valid data — the FTL dutifully preserves them during garbage collection, wasting time copying data that nobody cares about.

**TRIM** (or `UNMAP` in SCSI terminology, `DEALLOCATE` in NVMe) is a command the OS sends to the SSD saying: "Hey, these LBAs are no longer in use. You don't need to preserve them."

When the SSD receives a TRIM command, it:
1. Marks those pages as invalid in its mapping table
2. Can skip them during garbage collection
3. Can erase blocks containing only trimmed pages without copying anything

The result: lower write amplification, better performance, and longer drive life.

```bash
# On Linux, you can manually TRIM with:
sudo fstrim -v /

# Or enable continuous TRIM in /etc/fstab:
# /dev/nvme0n1p1  /  ext4  discard,noatime  0  1

# Check if TRIM is supported:
lsblk --discard
```

> [!TIP]
> Most modern Linux distributions run `fstrim.timer` weekly via systemd, which is generally preferred over the `discard` mount option. The mount option TRIMs on every delete, which can add latency. Periodic batch TRIM is more efficient.

## NVMe vs SATA: The Interface Matters

An SSD is only as fast as the pipe connecting it to the CPU. Two interfaces dominate:

**SATA (Serial ATA)**
- Originally designed for hard drives
- Uses the **AHCI** (Advanced Host Controller Interface) protocol
- Maximum bandwidth: ~600 MB/s (SATA III)
- Command queue depth: 32 commands, 1 queue
- Connects via the SATA port on the motherboard

**NVMe (Non-Volatile Memory Express)**
- Designed from scratch for flash storage
- Runs over **PCIe** (Peripheral Component Interconnect Express)
- Maximum bandwidth: ~3,500 MB/s (PCIe 3.0 x4), ~7,000 MB/s (PCIe 4.0 x4), ~14,000 MB/s (PCIe 5.0 x4)
- Command queue depth: 65,535 commands per queue, up to 65,535 queues
- Connects via M.2 slot or U.2/U.3 connectors

The queue depth difference is staggering. AHCI was designed when a single HDD would never have more than a handful of outstanding requests. NVMe's multi-queue architecture maps directly to modern multi-core CPUs — each core can have its own submission and completion queue, eliminating lock contention.

```
SATA/AHCI:
  CPU Core 0 ─┐
  CPU Core 1 ─┤── Single Queue (32 deep) ──▶ SSD
  CPU Core 2 ─┤
  CPU Core 3 ─┘

NVMe:
  CPU Core 0 ── Queue 0 (64K deep) ──▶
  CPU Core 1 ── Queue 1 (64K deep) ──▶  SSD
  CPU Core 2 ── Queue 2 (64K deep) ──▶
  CPU Core 3 ── Queue 3 (64K deep) ──▶
```

NVMe also reduces latency by communicating directly over PCIe without the AHCI translation layer. The NVMe command set is leaner — it was designed knowing the device is fast, so every microsecond of protocol overhead matters.

> [!NOTE]
> For typical consumer workloads (booting, launching apps, gaming), a SATA SSD and an NVMe SSD feel nearly identical. The difference shows up in sustained sequential transfers (large file copies) and heavy random I/O workloads (databases, VMs, development builds). If you're on a budget, a SATA SSD is still a massive upgrade from an HDD.

## SSD Controller Architecture: The Brain

The SSD controller is a System-on-Chip (SoC) that orchestrates everything. It's arguably the most important component in the drive — good NAND with a bad controller makes a bad SSD. Here's what's inside:

### Host Interface
Handles communication with the host over SATA, PCIe/NVMe, or other protocols. Implements the command queue, processes reads and writes, handles TRIM commands.

### Flash Channel Controllers
The controller connects to NAND chips via multiple parallel **channels** (typically 4–8). Each channel can operate independently, so the controller can read from one chip while writing to another. This parallelism is what gives SSDs their speed — a single NAND chip might only deliver 50–100 MB/s, but 8 channels in parallel can deliver 800 MB/s+.

More channels and more NAND packages per channel (called **interleaving**) means more parallelism and higher throughput.

### Embedded Processor
A multi-core ARM or RISC-V CPU running the FTL firmware. This handles the mapping table, garbage collection scheduling, wear leveling, error handling, and power-loss protection logic.

### DRAM Controller
Manages the external DRAM chip(s) that hold the FTL mapping table and read/write caches. High-end enterprise SSDs might have several GB of DRAM.

### ECC Engine
NAND flash is inherently unreliable — bits can flip due to read disturb, program disturb, or oxide degradation. The controller applies **Error Correcting Codes** (ECC), typically LDPC (Low-Density Parity-Check) codes, capable of correcting dozens of bit errors per page. As the NAND ages and error rates climb, the ECC engine works harder, which is one reason very old SSDs slow down.

### Encryption Engine
Most modern SSDs include hardware AES-256 encryption. Data is encrypted before being written to NAND and decrypted on read. This enables features like instant secure erase — you just throw away the encryption key, and all data on the drive is instantly unrecoverable.

## Practical Implications

### Over-Provisioning
SSDs reserve a portion of their raw NAND capacity for internal use — typically 7–28% depending on the drive. This **over-provisioned** space provides:
- Clean blocks for the FTL to use during writes
- Spare blocks to replace failed ones
- Breathing room for garbage collection

A 1 TB SSD might have 1024 GiB of raw NAND but only expose 953 GB to the user. Enterprise SSDs often expose even less (e.g., 800 GB usable from 1 TB of NAND) for better sustained write performance and longer endurance.

### Endurance Ratings

SSD endurance is measured two ways:

- **TBW (Terabytes Written)**: Total data you can write over the drive's warranty period. A typical 1 TB consumer SSD might be rated at 600 TBW. For context, writing 50 GB/day would take you ~33 years to hit that limit.
- **DWPD (Drive Writes Per Day)**: How many times you can overwrite the entire drive capacity each day over the warranty period. An enterprise SSD rated at 3 DWPD over 5 years can handle 3× its capacity in writes every single day for five years.

### How SSDs Die

SSDs don't fail like HDDs (sudden mechanical catastrophe). They degrade gradually:

1. P/E cycles wear the tunnel oxide. Error rates increase.
2. The ECC engine corrects more and more errors. Read latency creeps up.
3. Eventually, blocks fail completely. The controller remaps them to spare blocks from the over-provisioned pool.
4. When spare blocks run out, the drive enters **read-only mode** — you can still retrieve your data, but no more writes.
5. If the controller firmware crashes or the mapping table is lost... that's the bad scenario. Data recovery from raw NAND is theoretically possible but extremely expensive.

> [!IMPORTANT]
> Most SSD deaths in practice are controller or firmware failures, not NAND wearout. Monitor your SSD's health using S.M.A.R.T. attributes — specifically **Media Wearout Indicator** (or **Percentage Used**) and **Available Spare**. On Linux: `sudo smartctl -a /dev/nvme0n1` (from the `smartmontools` package).

## The Future of Flash Storage

### PLC (Penta-Level Cell)
5 bits per cell, 32 voltage levels. Yes, really. Kioxia and others have demonstrated working PLC NAND. The endurance is abysmal (~100 P/E cycles), but for read-heavy cold storage (think: replacing tape archives), it could make flash cheaper than HDDs per GB. Expect PLC drives to rely heavily on SLC caching and to target archival workloads.

### 3D NAND: Going Vertical
The industry hit a wall scaling planar (2D) NAND below ~15nm. The solution? Stack layers vertically. Modern 3D NAND stacks 200+ layers, with some manufacturers pushing past 300. Each layer is a full plane of flash cells. This dramatically increases density without shrinking cell geometry, which actually _improves_ endurance compared to the tiniest 2D cells.

### CXL-Attached Flash
**Compute Express Link (CXL)** is a new interconnect built on PCIe that allows memory-semantic access to devices. CXL-attached flash would let the CPU access SSD storage as if it were (very slow) memory — no filesystem, no block layer, just load/store instructions. This blurs the line between storage and memory, enabling new architectures for databases and in-memory computing that need more capacity than DRAM can economically provide.

### Computational Storage
Why move data to the CPU when you can move computation to the data? **Computational storage** SSDs embed processing elements (FPGAs, custom ASICs, or even small CPUs) that can perform operations — compression, encryption, database scans, regex matching — directly inside the drive. For data-intensive workloads, this eliminates the PCIe bandwidth bottleneck entirely.

### ZNS (Zoned Namespaces)
**Zoned Namespace SSDs** expose the erase-block structure to the host instead of hiding it behind the FTL. The host (or application) is responsible for writing sequentially within zones and explicitly resetting (erasing) zones when done. This eliminates the need for garbage collection inside the SSD, dramatically reducing write amplification and over-provisioning. It's a harder programming model, but databases like RocksDB and filesystems like F2FS are adopting it.

## Wrapping Up

An SSD is not just "a hard drive but faster." It's a fundamentally different storage architecture with its own physics, constraints, and tradeoffs. That tiny M.2 stick on your motherboard contains:

- Billions of floating gate transistors trapping electrons as data
- A sophisticated multi-core controller running complex firmware
- Algorithms for garbage collection, wear leveling, and error correction
- A mapping table that makes the whole thing look like a simple block device to your OS

The genius of the SSD isn't just that it's fast — it's that it hides extraordinary complexity behind a simple interface. Your OS writes to LBA 42, and an entire symphony of firmware, ECC, and garbage collection plays out in microseconds to make that work reliably for years.

Next time your system boots in 5 seconds, you'll know exactly what's happening behind the scenes.
