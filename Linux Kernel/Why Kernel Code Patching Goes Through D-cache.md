---
tags:
  - linux-kernel
  - cpu
  - cache
  - harvard-architecture
  - self-modifying-code
  - code-patching
created: '2026-04-30'
---
# Why Kernel Code Patching Goes Through the D-cache

> "The kernel writes new instructions through the D-cache, but the CPU fetches them through the I-cache."
>
> A natural follow-up question: **why the D-cache?** If the CPU is going to *execute* those bytes, wouldn't it be more direct to write them straight into the I-cache?

The answer is simple but worth internalising:

> **The CPU has no other way to write to memory.**

Everything else flows from that fact.

---

## 1. The CPU Only Has One Write Path

A CPU doesn't have a "write to instruction stream" instruction. Every store — whether you're storing a number, a pointer, or a *machine-code byte* — goes through the same pipeline:

```
        Execution unit (store)
                 │
                 ▼
        Load / Store Unit (LSU)
                 │
                 ▼
          Store Buffer
                 │
                 ▼
            L1 D-cache
                 │
                 ▼
              L2 / L3
                 │
                 ▼
              DRAM
```

There is **no `MOV [I-cache], val`** opcode. From the CPU's perspective, instructions are just *bytes in memory*. When the kernel writes `0x90 0x90 0x90` (three NOPs) at some address, that's a normal store as far as the silicon is concerned — it has no idea those bytes will later be fetched and executed.

So the kernel has no choice. **All writes go through the D-cache, full stop.**

---

## 2. Asymmetric Ports: I-cache is Read-Only

Inside the L1, the two caches are wired very differently:

| Cache | Ports | Direction |
|-------|-------|-----------|
| **L1 I-cache** | 1 read port (fetch unit) | CPU **reads only** |
| **L1 D-cache** | Read + write ports (LSU) | CPU **reads and writes** |

The L1-I has **one port: a read port for the fetch unit**. There is no store port into it. The fetch unit pulls cache lines in from L2 when it misses, and that's the only way data ever enters the I-cache.

This isn't an oversight — it's a deliberate design choice. Adding a write port to the I-cache would:

- **Cost silicon area** (multi-port SRAM is expensive).
- **Add latency to fetch** (more port-arbitration logic on the critical path).
- **Defeat the purpose of splitting L1** — the whole reason for separate I and D caches is that fetch and load/store have very different access patterns and can run in parallel without contention. A writable I-cache reintroduces the contention they were trying to avoid.

So designers said: *the I-cache is a read-only mirror; if you want to change instructions, do it elsewhere and then refresh the mirror.*

---

## 3. The 4-Step Code-Patching Dance

Because of (1) and (2), self-modifying code (kernel module loading, `ftrace`, `jump_label`, `kprobes`, live-patching, JITs in user space) all follow the same choreography:

```
┌────────────────────────────────────────────────────────────┐
│ Step 1: STORE new instruction bytes                        │
│         CPU → LSU → store buffer → L1-D                    │
│         (the new bytes now live in the D-cache)            │
└────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌────────────────────────────────────────────────────────────┐
│ Step 2: CLEAN L1-D to PoU                                  │
│         Push the dirty line down to L2 (the meeting point  │
│         that both I-cache and D-cache share).              │
│         arm64: dcache_clean_pou(start, end)                │
└────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌────────────────────────────────────────────────────────────┐
│ Step 3: INVALIDATE L1-I                                    │
│         Throw away any stale instruction bytes for that    │
│         address range. Next fetch *must* miss in L1-I.     │
│         arm64: icache_inval_pou(start, end)                │
└────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌────────────────────────────────────────────────────────────┐
│ Step 4: FETCH from L2 reloads L1-I                         │
│         When the CPU later executes that address, the      │
│         fetch misses in L1-I, goes to L2, finds the *new*  │
│         bytes (because Step 2 put them there), and pulls   │
│         them into L1-I. Execution proceeds with the new    │
│         instructions.                                      │
└────────────────────────────────────────────────────────────┘
```

This is exactly what `flush_icache_range(start, end)` does on architectures with split, software-coherent I/D caches.

On **arm64** the implementation is literally a clean-to-PoU followed by an invalidate-to-PoU plus a context-synchronisation event:

> `arch/arm64/include/asm/cacheflush.h:82-104` — `flush_icache_range()` calls `caches_clean_inval_pou()` and then IPIs all CPUs so they execute an `ISB` and refetch.

On **RISC-V** the I-cache flush boils down to a `fence.i`, which orders previous stores against subsequent instruction fetches:

> `arch/riscv/include/asm/cacheflush.h:11-14` — `local_flush_icache_all()` is a single `fence.i`.

---

## 4. Why Didn't CPU Designers Just Add a Write Port to L1-I?

Three reasons, in rough order of importance:

1. **Frequency mismatch.** Code patching is *rare*. The fetch path runs *every cycle*. Optimising the fetch path for the common case (read-only, single-port, low-latency) and pushing the rare case (writes) out to a slower path through L2 is the correct engineering trade-off.

2. **L2 already is the meeting point.** Once the dirty line is in L2, both caches can see it. There's no architectural need for a direct I↔D back-channel — the unified lower level *is* the back-channel.

3. **Coherency complexity.** A writable I-cache would need its own coherence protocol with the D-cache (MESI-style snooping between the two L1s). That's a lot of transistors and verification effort to optimise something that happens, at most, a few thousand times per boot.

So the design says: *let the rare path be slow and explicit; keep the hot path narrow and fast.*

---

## 5. x86 Hides This; ARM and RISC-V Don't

x86 maintains **hardware coherency between I and D caches via snooping**. When a store hits a line that's also cached in the I-cache, the I-cache line is invalidated automatically by the hardware. From the kernel's point of view this means:

- `flush_icache_range()` on x86 is essentially a no-op (just a serialising instruction to drain the pipeline).
- Self-modifying code "just works" — write the bytes, execute a serialising instruction (`CPUID`, `IRET`, etc.) to flush the pipeline, and you're done.

ARM and RISC-V instead require **software-managed coherency** between I and D caches. The kernel must:

- Clean D-cache to PoU (so L2 has the new bytes).
- Invalidate I-cache to PoU (so stale bytes are evicted).
- Issue a context-synchronisation event (ARM: `ISB`; RISC-V: `fence.i`) so the fetcher pipeline is flushed.

This is why the same `flush_icache_range()` API exists on every architecture but the implementation differs wildly:

| Arch | `flush_icache_range()` does what? |
|------|-----------------------------------|
| **x86** | Almost nothing — hardware snoops handle it |
| **arm64** | Clean D to PoU + invalidate I to PoU + IPI all CPUs to ISB |
| **RISC-V** | `fence.i` (locally; SMP requires IPIs to broadcast) |

The kernel API hides the difference; the cost does not.

---

## 6. Bottom Line

> **The kernel writes instructions through the D-cache because that's the only write path the CPU has.**
>
> The I-cache is deliberately read-only — splitting L1 into a read-only fetch cache and a read/write data cache is *the* design that makes modern superscalar pipelines work. Any time we modify code, we're necessarily fighting against that asymmetry, and the price we pay is the explicit clean-then-invalidate dance through L2.

On x86, hardware pays that price for us silently. On ARM and RISC-V, the kernel pays it with `flush_icache_range()` — which, under the hood, is exactly the four-step choreography above.

---

## References

| Claim | Source |
|-------|--------|
| `flush_icache_range()` on arm64 = clean+invalidate to PoU + IPI for ISB | `arch/arm64/include/asm/cacheflush.h:82-104` |
| RISC-V uses `fence.i` for local I-cache flush | `arch/riscv/include/asm/cacheflush.h:11-14` |
| Cache maintenance API contract (PoU/PoC, what flush_icache_range guarantees) | `Documentation/core-api/cachetlb.rst` |
| Module loader path that triggers code-patching cache maintenance | `kernel/module/main.c` (`flush_icache_range` after relocation) |
| ARM PoU / PoC concepts | `arch/arm64/include/asm/cacheflush.h` (top-of-file comment block) |

---

## Related Notes

- [[CPU Cache Addressing]] — VIVT / PIPT / VIPT, cache hierarchy, maintenance APIs across x86/arm64/RISC-V.
- [[CPU Pipeline and Memory Barriers]] — `READ_ONCE`, `WRITE_ONCE`, `smp_rmb`, fence instructions.
