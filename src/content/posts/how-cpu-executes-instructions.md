---
author: Vaibhav Jha
pubDatetime: 2026-07-08T10:00:00Z
title: "How a CPU Executes Instructions: Fetch, Decode, Execute, Repeat"
slug: how-cpu-executes-instructions
featured: false
draft: false
tags:
  - hardware
  - computer-architecture
  - deep-dive
description: "Billions of operations per second. But what does a CPU actually do in each clock cycle? Let's walk through the instruction pipeline and see how silicon thinks."
---

Your CPU is executing billions of operations every second. Not millions — *billions*. An Apple M4 core running at 4.4 GHz completes roughly 4.4 billion clock cycles per second, and modern superscalar designs retire multiple instructions *per cycle*. But what actually happens inside that chip? What does "executing an instruction" even mean at the hardware level?

Let's tear apart the abstraction and walk through the entire journey — from a line of C code to electrons racing through silicon.

## Table of contents

## What Is an Instruction?

At the lowest level, a CPU doesn't understand Python, C, or even assembly language. It understands **machine code** — fixed-width (or variable-width) binary patterns that encode exactly one operation.

Every instruction has two conceptual parts:

- **Opcode** — *what* to do (add, subtract, load from memory, jump)
- **Operands** — *what to do it on* (registers, memory addresses, immediate values)

For example, on ARM64 (AArch64), adding registers `x1` and `x2` and storing the result in `x0` looks like:

```asm
ADD x0, x1, x2
```

Which encodes to the 32-bit machine word:

```
0x8b020020
```

Broken down:

| Bits    | Field      | Value       |
|---------|------------|-------------|
| 31      | sf         | 1 (64-bit)  |
| 30-24   | opcode     | 0001011     |
| 23-22   | shift      | 00 (LSL)    |
| 21      | 0          | 0           |
| 20-16   | Rm (src2)  | x2          |
| 15-10   | imm6       | 000000      |
| 9-5     | Rn (src1)  | x1          |
| 4-0     | Rd (dest)  | x0          |

On x86-64, the equivalent `add rax, rcx` is a variable-length encoding — it could be 3 bytes, or with REX prefixes and addressing modes, much longer. This variable-length encoding is one of x86's defining characteristics (and one of its biggest headaches for hardware designers, as we'll see).

### Instruction Set Architectures

An **ISA** (Instruction Set Architecture) is the contract between software and hardware. It defines:

- The available instructions (arithmetic, logic, memory access, control flow)
- The register file (how many registers, their width)
- The memory model (byte ordering, alignment requirements)
- The encoding format

The two dominant ISAs today are **x86-64** (Intel/AMD, desktops and servers) and **ARM64** (Apple Silicon, Qualcomm, most phones and increasingly laptops/servers). We'll contrast them throughout this post.

## The Fetch-Decode-Execute Cycle

At its heart, every CPU — from the 8-bit 6502 in the original Apple II to a modern Zen 5 core — runs the same fundamental loop:

1. **Fetch** — Read the next instruction from memory (pointed to by the **Program Counter** / **Instruction Pointer**)
2. **Decode** — Figure out what the instruction means (which operation, which operands)
3. **Execute** — Perform the operation (arithmetic, memory access, branch)
4. **Write Back** — Store the result (to a register or memory)
5. **Update PC** — Advance the program counter to the next instruction (or to a branch target)

In the simplest possible CPU, these happen sequentially. One instruction finishes entirely before the next one begins. If each stage takes one clock cycle, a single instruction takes 4-5 cycles.

This is how the earliest processors worked. It's also incredibly wasteful — while the execute stage is doing arithmetic, the fetch and decode hardware sits idle. A modern CPU would never tolerate this.

## Pipelining: The Assembly Line

The insight that transformed CPU design in the 1980s was borrowed from manufacturing: **pipelining**.

Just like a car factory doesn't wait for one car to be fully assembled before starting the next, a pipelined CPU overlaps instruction stages. While instruction N is executing, instruction N+1 is being decoded, and instruction N+2 is being fetched.

### The Classic 5-Stage RISC Pipeline

The textbook RISC pipeline (pioneered by the MIPS R2000 and similar designs) has five stages:

```
Time →    Cycle 1  Cycle 2  Cycle 3  Cycle 4  Cycle 5  Cycle 6  Cycle 7
Instr 1:  [IF]     [ID]     [EX]     [MEM]    [WB]
Instr 2:           [IF]     [ID]     [EX]     [MEM]    [WB]
Instr 3:                    [IF]     [ID]     [EX]     [MEM]    [WB]
```

- **IF** — Instruction Fetch
- **ID** — Instruction Decode (and register read)
- **EX** — Execute (ALU operation or address calculation)
- **MEM** — Memory access (load/store)
- **WB** — Write Back (result to register file)

The throughput improvement is dramatic. An unpipelined CPU takes 5 cycles per instruction (CPI = 5). A fully pipelined CPU, ideally, completes one instruction *every cycle* (CPI = 1), even though each individual instruction still takes 5 cycles of **latency**.

> [!NOTE]
> Throughput vs. latency is a distinction that shows up everywhere in computing — networks, disks, memory. Pipelining improves throughput (instructions per second) without improving latency (time for one instruction to complete).

Modern CPUs have much deeper pipelines. Intel's Skylake architecture has roughly 14-19 pipeline stages. Apple's performance cores are estimated at 12-16 stages. Deeper pipelines allow higher clock speeds (each stage does less work, so it completes faster), but introduce new problems.

## Pipeline Hazards: When the Assembly Line Stalls

Pipelining works beautifully when instructions are independent. But real programs are full of dependencies. When the pipeline can't smoothly advance, it **stalls** — inserting "bubbles" (wasted cycles) into the pipeline.

There are three categories of hazards:

### Data Hazards

A data hazard occurs when an instruction depends on the result of a previous instruction that hasn't finished yet.

```asm
ADD x1, x2, x3    ; x1 = x2 + x3
SUB x4, x1, x5    ; x4 = x1 - x5  ← needs x1, but ADD hasn't written it back yet!
```

The SUB instruction needs the value of `x1`, but in a simple pipeline, `ADD` doesn't write back until its WB stage — two cycles after SUB needs to read it in its ID stage.

**Solution: Forwarding (bypassing).** The hardware adds shortcut paths that route the ALU result directly from the EX stage output back to the EX stage input, without waiting for the write-back. This eliminates many data hazard stalls completely.

Some hazards can't be forwarded away. A **load-use hazard** happens when a load instruction is immediately followed by an instruction using the loaded value:

```asm
LDR x1, [x2]      ; load x1 from memory
ADD x3, x1, x4    ; needs x1 — but the data isn't available until MEM stage
```

Even with forwarding, there's a one-cycle bubble here because the data isn't available until after the MEM stage. Compilers actively reorder instructions to fill these "load delay slots" with useful work.

### Control Hazards

A control hazard occurs at branches. When the CPU encounters a conditional branch (`BEQ`, `JNZ`, etc.), it doesn't know which instruction to fetch next until the branch condition is evaluated — which might not happen for several pipeline stages.

```asm
CMP x1, x2
BEQ target        ; branch if equal — but which instruction should we fetch next?
ADD x3, x4, x5    ; this instruction? ...
...
target:
SUB x6, x7, x8    ; ...or this one?
```

In a 5-stage pipeline, this is a 1-2 cycle penalty. In a 19-stage pipeline, it could mean flushing *many* stages of work. This is why branch prediction (covered next) is so critical.

### Structural Hazards

A structural hazard occurs when two instructions need the same hardware resource simultaneously — for example, if there's only one memory port and both a fetch and a memory access want to use it in the same cycle. Modern CPUs largely eliminate structural hazards through resource duplication (separate instruction and data caches, multiple ALUs, etc.).

## Branch Prediction: The CPU's Crystal Ball

Modern CPUs don't wait to resolve branches. They **predict** the outcome and speculatively fetch/decode/execute down the predicted path. If the prediction is right (and it usually is), there's no penalty at all. If it's wrong, the pipeline is flushed and the CPU recovers — wasting all the speculative work.

### Why It Works

Branch prediction accuracy in modern CPUs is staggeringly good — typically **95-99%** for well-structured code. This isn't magic; it's pattern recognition in hardware.

The simplest predictor is a **2-bit saturating counter** per branch: it tracks whether the branch was recently taken or not taken, requiring two consecutive mispredictions to flip its prediction. This handles loops well — a loop branch is "taken" every iteration except the last, so only the final iteration mispredicts.

Modern predictors are far more sophisticated. AMD's Zen 4 and Intel's Golden Cove use **TAGE (TAgged GEometric history length) predictors** — multi-table predictors indexed by different lengths of branch history, allowing them to detect complex patterns like "this branch is taken every 3rd time" or "this branch depends on the outcome of a branch 7 instructions ago."

Apple's M-series cores use a massive branch predictor that correlates across a very large history window, contributing to their high IPC.

### When Prediction Fails

A misprediction is expensive. The CPU must:

1. Detect that the prediction was wrong (when the branch resolves in the execute stage)
2. Flush all speculatively fetched/decoded/executed instructions from the pipeline
3. Redirect the fetch unit to the correct path
4. Restart execution

On a deep pipeline, this can waste 15-20 cycles of work. This is why "branch-heavy" code with unpredictable patterns (random `if/else` choices, virtual dispatch in OOP) can be surprisingly slow.

> [!WARNING]
> The infamous **Spectre** vulnerability (2018) exploits speculative execution. When the CPU speculatively executes down a mispredicted path, it accesses memory that it shouldn't. Even though the architectural state is rolled back on misprediction, the *cache* state is not — the speculatively accessed data leaves traces in the cache that an attacker can measure via timing side-channels. This is a fundamental tension between security and performance that the industry is still grappling with.

You can see the impact of branch prediction in practice. Consider sorting an array before processing it with a branch:

```c
// Unpredictable branches — ~50% taken
for (int i = 0; i < N; i++) {
    if (data[i] >= 128)  // random data: unpredictable!
        sum += data[i];
}

// Sort first, then process — branches become predictable
sort(data, N);
for (int i = 0; i < N; i++) {
    if (data[i] >= 128)  // sorted: first all < 128, then all >= 128
        sum += data[i];
}
```

The sorted version can be **5-6x faster** on the same data — not because of any algorithmic improvement, but because the branch predictor can now see an obvious pattern (all not-taken, then all taken).

## Out-of-Order Execution: Breaking the Sequence

Here's a mind-bending fact: modern CPUs don't execute instructions in program order. They execute them in **data-dependency order** — whatever instruction has its operands ready first gets to execute first, regardless of where it appears in the program.

Consider:

```asm
LDR  x1, [x0]       ; (1) load from memory — might take 100+ cycles on cache miss
ADD  x2, x1, x3     ; (2) depends on x1 — must wait for (1)
MUL  x4, x5, x6     ; (3) independent — why wait?
ADD  x7, x4, x8     ; (4) depends on x4
```

An in-order CPU would stall at instruction (2), waiting for the load to complete, and instructions (3) and (4) would sit idle even though they have nothing to do with `x1`. An out-of-order (OoO) CPU recognizes that (3) is independent and executes it immediately.

### The Out-of-Order Machinery

The hardware that enables this is remarkably complex:

1. **Frontend (in-order):** Instructions are fetched and decoded in program order. x86 CPUs decode complex CISC instructions into simpler **micro-ops (µops)** at this stage.

2. **Register Renaming:** The CPU maintains far more physical registers than the ISA exposes. When an instruction writes to `x1`, it's actually assigned a new physical register (say, `p47`). This eliminates false dependencies — two instructions writing to the same architectural register don't actually conflict because they write to different physical registers.

3. **Reservation Stations / Scheduler:** Decoded µops are placed in a queue. Each entry tracks which operands are ready. When all operands are available, the µop is **issued** to an execution unit. This is the key data-flow engine — instructions execute as soon as they can, not when it's "their turn."

4. **Execution Units:** Multiple functional units (ALUs, FPUs, load/store units, branch units) execute µops in parallel.

5. **Reorder Buffer (ROB):** Results are written into the ROB in program order. Even though execution is out-of-order, **retirement** (committing results to architectural state) is strictly in-order. This ensures that if an exception or misprediction occurs, the CPU can cleanly roll back to a precise state.

```
                    ┌──────────────┐
   Fetch → Decode → │   Rename /   │ → Scheduler → Execute → ROB → Retire
                    │  Allocate    │                (OoO)      (in-order)
                    └──────────────┘
```

Modern ROBs are large. AMD Zen 4 has a 320-entry ROB. Apple M2's performance core reportedly has over 600 entries. The bigger the ROB, the further ahead the CPU can look for independent work to do while waiting on long-latency operations like cache misses.

## Superscalar Execution: Doing Multiple Things At Once

Pipelining gets us to one instruction per cycle. **Superscalar** execution goes beyond that — issuing and retiring *multiple* instructions per clock cycle.

A modern CPU core is typically **6-wide** or wider, meaning it can decode, issue, and retire up to 6 (or more) µops per cycle. Apple's M-series performance cores are reportedly 8-wide decode, making them among the widest cores in production.

To sustain this, the core needs:

- **Multiple decoders** (6-8 parallel decode slots)
- **Multiple execution ports** (perhaps 4 integer ALUs, 2 load units, 2 store units, 2 FP/SIMD units)
- **A wide retirement path** (retiring 6-8 µops per cycle)
- **A massive scheduler** to keep all those execution units fed

The practical IPC (instructions per cycle) achieved depends on the workload. Typical desktop workloads achieve around 2-4 IPC. Highly optimized, compute-dense code can push closer to the theoretical maximum.

> [!TIP]
> You can measure your actual IPC on Linux using `perf stat`:
> ```bash
> perf stat -e instructions,cycles ./your_program
> ```
> The `instructions / cycles` ratio is your IPC. Typical values range from 1.5 to 4.0 for well-optimized code.

## The Role of Cache: Hiding Memory Latency

The CPU-memory speed gap is perhaps the single biggest bottleneck in modern computing. A modern CPU core can execute an instruction in ~0.3 nanoseconds. A main memory (DRAM) access takes ~60-100 nanoseconds — a **200-300x gap**.

Without caches, out-of-order execution and superscalar width would be meaningless — the CPU would spend almost all its time waiting for memory.

### The Cache Hierarchy

Modern CPUs use a multi-level cache hierarchy:

| Level | Typical Size | Latency (cycles) | Shared? |
|-------|-------------|-------------------|---------|
| L1 I-cache | 32-64 KB | 3-5 | Per core |
| L1 D-cache | 32-48 KB | 4-5 | Per core |
| L2 cache | 256 KB - 1 MB | 12-14 | Per core |
| L3 cache | 16-96 MB | 30-50 | Shared across cores |
| Main memory (DRAM) | 8-128+ GB | 150-300+ | Shared |

The L1 cache is split into separate **instruction cache** (I-cache) and **data cache** (D-cache) to avoid structural hazards between fetch and load/store.

### Cache Lines

Memory is loaded into cache in fixed-size chunks called **cache lines**, typically 64 bytes. When you access a single byte, the entire 64-byte line containing it is loaded. This is why **spatial locality** matters — accessing memory sequentially is fast because subsequent accesses hit the already-loaded cache line.

This has real consequences for how you structure data:

```c
// Cache-friendly: sequential access
struct Point { float x, y, z; };
struct Point points[1000];
for (int i = 0; i < 1000; i++)
    sum += points[i].x;  // Stride = sizeof(Point) = 12 bytes, fits in cache lines

// Cache-hostile: linked list traversal
struct Node { int data; struct Node* next; };
// Each node could be anywhere in memory — almost every access is a cache miss
```

### Cache Misses

When the CPU needs data that's not in the cache, it's a **cache miss**. There are three types:

- **Compulsory (cold) miss** — First access to a cache line; unavoidable
- **Capacity miss** — The working set is larger than the cache
- **Conflict miss** — Two addresses map to the same cache set and evict each other

An L1 miss costs ~10 cycles (satisfied by L2). An L2 miss costs ~30-50 cycles (L3). An L3 miss costs 150-300+ cycles (DRAM). An L3 miss is a catastrophic stall — the out-of-order engine tries desperately to find other work to do while waiting, but if there's nothing independent, the core stalls.

Hardware **prefetchers** try to detect access patterns (sequential, strided) and fetch cache lines *before* the CPU asks for them. This is why sequential array access is often even faster than the L1 hit rate would suggest.

## Clock Speed vs. IPC: The Performance Equation

There's a simple equation that governs CPU performance:

```
Performance = Instructions × CPI × Clock Period

or equivalently:

Throughput = IPC × Clock Frequency
```

- **Clock frequency** (GHz) — how many cycles per second
- **IPC** (Instructions Per Cycle) — how many instructions complete per cycle

A CPU at 3 GHz with IPC of 4 completes 12 billion instructions per second. A CPU at 5 GHz with IPC of 2 completes only 10 billion — the slower-clocked CPU is actually faster.

This is exactly why Apple's M-series chips (which run at ~3.2-4.4 GHz) can outperform Intel chips running at 5+ GHz on many workloads. Apple achieves higher IPC through wider pipelines, larger out-of-order windows, and more aggressive branch prediction.

**Clock speed** is limited by:
- **Power consumption** — scales roughly with V²·f (voltage squared times frequency). Cranking up frequency requires higher voltage, which means quadratically more power.
- **Pipeline depth** — deeper pipelines enable higher clocks but increase branch misprediction penalties.
- **Thermal limits** — more power means more heat, and silicon has maximum operating temperatures.

**IPC** is improved by:
- Wider superscalar execution
- Better branch prediction
- Larger out-of-order windows (bigger ROB, more physical registers)
- Better cache hierarchy
- Smarter prefetching

The industry trend since ~2005 (when the "GHz race" hit a wall due to power/thermal limits) has been to focus on IPC improvements and core count rather than raw clock speed.

## CISC vs. RISC: Two Philosophies

### CISC: x86's Approach

**Complex Instruction Set Computing** gives the programmer (and compiler) rich, powerful instructions. A single x86 instruction can load from memory, perform arithmetic, and store the result:

```asm
ADD [rax+rbx*4+8], rcx   ; memory[rax + rbx*4 + 8] += rcx
```

This is one instruction that does a load, an add, and a store. Convenient for compilers, but a nightmare for hardware. Internally, modern x86 CPUs **crack** (AMD's term) or **decode** these into multiple simpler **micro-ops (µops)**:

```
µop 1: LOAD  temp, [rax+rbx*4+8]    ; load from memory
µop 2: ADD   temp, temp, rcx         ; perform addition
µop 3: STORE [rax+rbx*4+8], temp     ; store result
```

The decoder is one of the most power-hungry and complex parts of an x86 core. Intel and AMD both use a **µop cache** (Intel calls it the DSB — Decoded Stream Buffer) that stores already-decoded µops, allowing the core to skip the decode step for hot code.

### RISC: ARM's Philosophy

**Reduced Instruction Set Computing** keeps instructions simple and uniform. Every ARM64 instruction is exactly 32 bits. Load/store architecture means only `LDR`/`STR` instructions access memory — arithmetic operates only on registers:

```asm
LDR  x1, [x0, x2, LSL #2]   ; load from memory
ADD  x1, x1, x3              ; add
STR  x1, [x0, x2, LSL #2]   ; store back
```

Three instructions instead of one, but each is simple to decode. The decoder is smaller, uses less power, and is easier to make wide. This is a major reason ARM dominates mobile (power efficiency) and why Apple Silicon achieves such high IPC — the saved power and die area from simpler decoders is reinvested into larger caches, wider execution, and bigger out-of-order buffers.

> [!NOTE]
> The CISC vs. RISC distinction has blurred significantly. Modern x86 CPUs are essentially RISC cores with a CISC-to-RISC translation frontend. The µop cache means hot loops run at RISC-like efficiency. Meanwhile, ARM has added increasingly complex instructions (SVE2, crypto extensions) that push toward CISC territory. The real distinction today is more about the **encoding format** (fixed vs. variable width) and the **decoder complexity** it implies.

## A Modern Instruction's Journey

Let's trace a single instruction through a modern high-performance core (think Apple M3 performance core or AMD Zen 4):

### 1. Branch Prediction & Fetch

Before anything else, the **branch predictor** examines the current PC and predicts the next several PCs — identifying branches and predicting their targets. The fetch unit reads a block of instructions (32-64 bytes) from the **L1 instruction cache**.

On x86, the fetched bytes may also hit the **µop cache**, skipping decode entirely.

### 2. Decode

On ARM64: Instructions are fixed-width (32-bit), so the decoder can easily pick out 8 instructions in parallel from the fetch block. Decoding is largely just wire routing — reading fixed bit fields.

On x86: The decoder must handle variable-length instructions (1-15 bytes). It first determines instruction boundaries (a hard problem!), then decodes up to 4-6 instructions per cycle. Complex instructions are decoded into multiple µops using a microcode ROM.

### 3. Rename & Dispatch

Each instruction's architectural registers are mapped to physical registers via the **Register Alias Table (RAT)**. False dependencies (WAR, WAW hazards) are eliminated. Instructions are placed into the **Reorder Buffer** and dispatched to **reservation stations** grouped by execution port.

### 4. Schedule & Execute

The scheduler monitors operand availability. When all source operands are ready (either from the register file or forwarded from a completing execution unit), the µop is issued to an appropriate execution unit:

- **Integer ALU** — add, subtract, logic, shift (~1 cycle latency)
- **Integer MUL** — multiply (~3 cycles)
- **FP/SIMD unit** — floating-point and vector operations (~3-5 cycles)
- **Load unit** — sends address to L1 data cache (~4-5 cycles)
- **Store unit** — writes data through the store buffer
- **Branch unit** — resolves branch conditions, checks predictions

### 5. Memory Access

If our instruction is a load, the load unit computes the effective address, checks the **TLB** (Translation Lookaside Buffer, which caches virtual-to-physical address translations), then accesses the L1 data cache. On a hit, data returns in ~4-5 cycles. On a miss, the request propagates through L2 → L3 → DRAM.

Meanwhile, the **store buffer** holds pending stores and allows loads to snoop it for recent writes (store-to-load forwarding), avoiding unnecessary memory accesses.

### 6. Write Back & Retire

The result is written to the physical register file and the ROB entry is marked as completed. When this instruction reaches the head of the ROB and all older instructions have also completed, it **retires** — its results become part of the committed architectural state.

If a misprediction or exception is detected, all instructions younger than the faulting instruction are squashed, and the rename map is restored from a checkpoint.

### The Whole Picture

Here's the journey from a 10,000-foot view:

```
                         ┌─────────────────────────────┐
                         │      Branch Predictor        │
                         └────────────┬────────────────┘
                                      ▼
┌─────────┐   ┌─────────┐   ┌────────────────┐   ┌──────────────┐
│ L1 I-$  │──▶│  Fetch   │──▶│     Decode     │──▶│   Rename /   │
│         │   │  (6-8w)  │   │  (µop crack)   │   │   Dispatch   │
└─────────┘   └─────────┘   └────────────────┘   └──────┬───────┘
                                                         │
                              ┌───────────────────────────┘
                              ▼
                    ┌───────────────────┐
                    │    Scheduler /    │
                    │ Reservation Stns  │
                    └───┬───┬───┬───┬───┘
                        │   │   │   │
                        ▼   ▼   ▼   ▼
                    ┌─────────────────────────────────┐
                    │  ALU  ALU  MUL  FP  LD  ST  BR  │
                    │     Execution Units              │
                    └───────────────┬──────────────────┘
                                    │
                              ┌─────▼─────┐
                              │    ROB     │──▶ Retire (in-order)
                              │ (320-600+) │
                              └────────────┘
```

Every cycle, this entire machine advances — fetching new instructions at the top, retiring completed ones at the bottom, with potentially hundreds of instructions in various stages of execution simultaneously.

## Putting It All Together

Understanding how a CPU executes instructions reveals why certain performance intuitions are wrong:

1. **"More GHz = faster"** — Not necessarily. IPC matters just as much, and often more.
2. **"Branches are free"** — Unpredictable branches are expensive. Sorting data, using branchless code, or restructuring conditionals can yield dramatic speedups.
3. **"Memory access is just a load"** — A cache miss to DRAM is 200+ cycles. Data layout, access patterns, and cache-friendliness are often the dominant performance factors.
4. **"Instructions execute one at a time"** — A modern CPU might have 300+ instructions in flight simultaneously, executing out of order across 8+ execution units.
5. **"x86 is slow because CISC"** — The µop cache means hot code runs at essentially RISC speeds. The decode penalty is real but amortized.

The modern CPU is, in many ways, a speculative, out-of-order, superscalar **dataflow engine** wearing the mask of a simple sequential processor. It maintains the *illusion* of sequential execution (because that's what the ISA promises) while internally running as a massively parallel machine. Every clock cycle, dozens of decisions are made by hardware — which branches to predict, which instructions to issue, which cache lines to prefetch — all to keep the pipeline full and the execution units busy.

Billions of times per second. Every second you use your computer.

And that's just *one* core.
