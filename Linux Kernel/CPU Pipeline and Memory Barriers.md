---
tags:
  - linux-kernel
  - cpu
  - memory-barrier
  - pipeline
  - RISC-V
  - computer-architecture
---

# CPU Pipeline and Memory Barriers

---

## Why Memory Barriers Exist — It's All Because of the Pipeline

The core insight: **memory barriers are software's way of talking to the CPU pipeline.**

---

## What the Pipeline Does to Memory Operations

You write this in C:
```c
a = 1;
b = 2;
```

You expect this order in hardware. But the pipeline doesn't guarantee it.

```
What YOU wrote:          What the CPU MAY do:
──────────────           ───────────────────
STORE a = 1              STORE b = 2   ← reordered!
STORE b = 2              STORE a = 1   ← reordered!
```

This is called **memory reordering** — and it happens for real, legitimate performance reasons inside the pipeline.

---

## Why the Pipeline Reorders Memory Operations

### Reason 1 — Store Buffer

The CPU doesn't write to cache immediately. It uses a **store buffer** to decouple the pipeline from slow cache/memory:

```
  CPU Core
  ┌──────────────────────────────────────────────────────┐
  │                                                      │
  │   Pipeline                                           │
  │   ────────                                           │
  │   STORE a=1 ──▶  Store Buffer  ──▶ (drains later)    │
  │   STORE b=2 ──▶  Store Buffer  ──▶ (drains later)    │
  │                  ┌──────────┐                        │
  │                  │ a=1      │                        │
  │                  │ b=2      │                        │
  │                  └────┬─────┘                        │
  │                       │  drains to cache when free   │
  └───────────────────────┼──────────────────────────────┘
                          ▼
                    L1 D-cache
                    (eventual destination)
```

The store buffer drains **out of order** based on cache availability. So even if you wrote `a=1` before `b=2`, `b` might reach cache first if `b`'s cache line happens to be hot.

### Reason 2 — Load Speculation

The CPU speculatively executes loads **before** knowing if an earlier store will affect them:

```
  STORE a = 1       ← store goes to store buffer (slow)
  LOAD  r1 = b      ← CPU doesn't wait — speculatively loads b NOW
                       even though the store hasn't committed yet

  From another core's perspective:
    it sees the LOAD result before the STORE is visible
    → load appears to happen BEFORE the store
```

### Reason 3 — Out-of-Order Execution

The pipeline executes instructions when their **operands are ready**, not in program order:

```
  LOAD  r1 = [addr_a]   ← cache miss, will take 200 cycles
  ADD   r2, r3, r4      ← r3, r4 already in registers → execute NOW
  STORE [addr_b] = r2   ← r2 now ready → execute NOW

  Result: STORE to b appears to complete before LOAD from a
          even though the LOAD came first in your code
```

---

## The 4 Types of Reordering

```
  Type          What gets reordered       Example
  ──────────    ───────────────────       ───────────────────────────
  StoreStore    store A before store B    a=1; b=2  →  b=2; a=1
  LoadLoad      load A before load B      r=a; s=b  →  s=b; r=a
  LoadStore     load before store         r=a; b=1  →  b=1; r=a
  StoreLoad     store before load         a=1; r=b  →  r=b; a=1
                ← MOST dangerous —
                  even x86 TSO allows this one
```

---

## Memory Barriers — Telling the Pipeline to Stop Reordering

A memory barrier is a CPU instruction that **forces the store buffer to drain** and **prevents the pipeline from reordering across it**:

```
  WITHOUT barrier:                WITH barrier:

  STORE a = 1                     STORE a = 1
  STORE b = 2                     ───── smp_wmb() ─────  ← FENCE w,w
                                  STORE b = 2

  Pipeline free to reorder        Pipeline MUST complete
  b=2 might be visible            a=1 before making b=2
  before a=1                      visible to other cores
```

---

## How Each Barrier Maps to Pipeline Behavior

```
Barrier         Prevents              Pipeline effect
──────────────  ────────────────────  ──────────────────────────────────────
smp_wmb()       StoreStore reorder    drain store buffer before next store
FENCE w,w       (RISC-V)              all pending stores commit to cache
                                      before any subsequent store

smp_rmb()       LoadLoad reorder      invalidate load buffer before next load
FENCE r,r       (RISC-V)              prevent speculative loads from crossing

smp_mb()        all reordering        full pipeline drain
FENCE rw,rw     (RISC-V)              most expensive — stops everything

READ_ONCE(x)    compiler reorder      prevents compiler from caching x in reg
                + LoadLoad            also implies a hardware load barrier

WRITE_ONCE(x,v) compiler reorder      prevents compiler from eliminating store
                + StoreStore          also implies a hardware store barrier
```

---

## Real Example — Linux Kernel Producer/Consumer

```c
/* Producer (CPU 0) */
data = 42;          // (1) write data
smp_wmb();          // (2) barrier — ensure (1) visible before (3)
flag = 1;           // (3) signal ready

/* Consumer (CPU 1) */
while (!flag);      // (4) wait for signal
smp_rmb();          // (5) barrier — ensure (4) visible before (6)
use(data);          // (6) read data — guaranteed to see 42
```

Without `smp_wmb()`:
```
CPU 0 pipeline might reorder:
  flag = 1      ← visible first!
  data = 42     ← visible second

CPU 1 sees flag=1, reads data → gets 0 (stale!)  ← data race bug
```

---

## x86 vs RISC-V — Why RISC-V Needs More Barriers

```
  x86 (TSO — Total Store Order)
  ─────────────────────────────
  Hardware prevents StoreStore reordering automatically
  Hardware prevents LoadLoad reordering automatically
  Hardware prevents LoadStore reordering automatically
  Only StoreLoad can reorder (mfence needed)

  → Linux kernel needs fewer explicit barriers on x86
  → smp_rmb() / smp_wmb() are NOPs on x86!
     only smp_mb() compiles to a real instruction (mfence / lock prefix)


  RISC-V (RVWMO — Relaxed Memory Order)
  ──────────────────────────────────────
  ALL four reordering types are allowed by hardware
  → every barrier compiles to a real FENCE instruction
  → kernel driver code on RISC-V needs MORE explicit barriers
  → missing a barrier on RISC-V = silent data corruption

  smp_rmb()  →  FENCE r,r    (real instruction)
  smp_wmb()  →  FENCE w,w    (real instruction)
  smp_mb()   →  FENCE rw,rw  (real instruction)
```

---

## Summary — The Chain of Causation

```
Out-of-Order Pipeline
  │
  ├── Store Buffer          → causes StoreStore, StoreLoad reordering
  ├── Load Speculation      → causes LoadLoad, LoadStore reordering
  └── Out-of-Order Exec     → causes all types of reordering
          │
          │  multi-core programs observe these reorderings
          │  as data races and memory consistency bugs
          ▼
Memory Barriers
  │
  ├── smp_wmb() / FENCE w,w   → fixes StoreStore
  ├── smp_rmb() / FENCE r,r   → fixes LoadLoad
  ├── smp_mb()  / FENCE rw,rw → fixes everything
  └── READ/WRITE_ONCE()       → fixes compiler reordering too
          │
          ▼
  Correct multi-core behavior
  (driver code, lock-free data structures, ring buffers...)
```

> **Key insight for RISC-V:** RISC-V's relaxed memory model (RVWMO) means every barrier in a Linux driver compiles to a real `FENCE` instruction — unlike x86 where many barriers are NOPs. When porting drivers to RISC-V, missing barriers that were harmless on x86 become real bugs.
