---
tags:
  - linux-kernel
  - cpu
  - cache
  - memory
  - arm
  - x86
  - riscv
  - mmu
  - harvard-architecture
---
# CPU Cache Addressing: VIVT, PIPT, VIPT

CPU caches can be indexed and tagged using either **virtual** (VA) or **physical** (PA) addresses. The choice affects speed, correctness, and what the OS must do on context switch.

- **Index**: which cache set to look in
- **Tag**: stored alongside the line, used to verify a hit

---

## The Three Schemes

### VIVT — Virtually Indexed, Virtually Tagged

Both index and tag come from the virtual address.

```
   Virtual Address
        │
        ├─── index ──→ cache lookup
        │
        └─── tag   ──→ compare with stored tag
                       (no MMU needed)
```

**Pro**: Fastest — no address translation before lookup.

**Cons**:
- **Aliasing (synonyms)**: Two VAs in different processes mapping to the same PA → two separate cache lines for the same data → coherency problem.
- **Homonyms**: Same VA in two processes maps to different PAs → tag collision → must flush on context switch.

Almost no modern CPU uses pure VIVT for this reason.

---

### PIPT — Physically Indexed, Physically Tagged

Both index and tag come from the physical address (after MMU translation).

```
   Virtual Address
        │
        ▼
      [TLB] ──→ Physical Address
                     │
                     ├─── index ──→ cache lookup
                     └─── tag   ──→ compare
```

**Pro**: No aliasing, no homonyms. Correct by construction.

**Con**: TLB translation is on the critical path → adds latency.

Used for **L2/L3 caches**, where latency is less critical.

---

### VIPT — Virtually Indexed, Physically Tagged

Index from VA (fast), tag from PA (correct). Both lookups happen **in parallel**.

```
   Virtual Address
        │
        ├─── index ──→ cache lookup ──→ stored physical tag
        │                                       │
        │                                       ▼
        └─── [TLB] ──→ PFN ────────────→ compare → hit/miss
              (in parallel)
```

**Key insight**: If the index bits fall within the **page offset** (bits below the page boundary), they're identical in VA and PA — so VIPT is **alias-free** and behaves like PIPT.

```
Page size = 4KB → offset bits = [11:0]
If cache index bits ⊆ [11:0] → VIPT behaves like PIPT (no aliasing)
If index bits spill above bit 11 → aliasing is possible
```

**Pro**: L1 hit latency approaches VIVT speed with PIPT correctness (when alias-free).

**Con**: If the cache is large enough that index bits exceed the page offset, aliasing returns and the OS must handle it (e.g., page coloring or selective flushing).

Used for **ARM Cortex-A L1 caches** (including Raspberry Pi's Cortex-A72).

---

## Summary Table

| Scheme | Index | Tag | Speed | Aliasing Risk |
|--------|-------|-----|-------|---------------|
| VIVT   | VA    | VA  | Fast  | High          |
| PIPT   | PA    | PA  | Slow  | None          |
| VIPT   | VA    | PA  | Fast* | Conditional   |

\*Fast only when index bits ⊆ page offset bits.

---

## Worked Example: VIPT Lookup

**Configuration**: 2-way set-associative, 64-byte lines, 64 sets → total 8KB cache.

**Bit breakdown of a virtual address:**

```
 47                    12 11      6 5       0
┌──────────────────────┬─────────┬─────────┐
│   VPN (→ TLB)        │  index  │ offset  │
└──────────────────────┴─────────┴─────────┘
       36 bits             6 bits   6 bits
```

- `[5:0]` — **block offset** (64 bytes = 2⁶)
- `[11:6]` — **set index** (64 sets = 2⁶)
- `[47:12]` — **virtual page number**, sent to TLB

**Parallel lookup:**

```
       Virtual Address [47:0]
              │
       ┌──────┴───────────────┐
       │                      │
       ▼                      ▼
   [47:12] → TLB        [11:0] → cache
                │              │
                ▼              ▼
              PFN        physical tag (stored)
                \            /
                 \          /
                  ▼        ▼
                  compare equal?
                       │
                       ▼
                   hit / miss
```

Because the index bits `[11:6]` fall entirely within the 4KB page offset `[11:0]`, this configuration is **alias-free VIPT** — it behaves like PIPT with VIVT's speed.

**Counter-example**: If the cache had 512 sets (9 index bits), the index would reach `[14:6]`, spilling above bit 11, and aliasing becomes possible.

---

## Cache Hierarchy: Split L1, Unified L2/L3

At **L1**, virtually every modern CPU has **physically separate** I-cache and D-cache hardware. This is called a **modified Harvard architecture**.

```
                ┌──────────────┐
                │   CPU Core   │
                └──┬────────┬──┘
                   │        │
            instr fetch   load/store
                   │        │
              ┌────▼──┐  ┌──▼────┐
              │ L1-I  │  │ L1-D  │   ← separate SRAM arrays,
              │ cache │  │ cache │     separate ports, separate
              └───┬───┘  └───┬───┘     tag arrays, separate MSHRs
                  └────┬─────┘
                       ▼
                  ┌─────────┐
                  │   L2    │      ← unified
                  └─────────┘
```

**Why split at L1?**
- The CPU needs to **fetch instructions and data simultaneously** every cycle (pipelined execution). One unified cache would need dual-port SRAM, which is expensive and slow.
- I-cache is **read-mostly** — no store buffers, no write-back logic, simpler design, can be optimized for fetch bandwidth.
- D-cache needs **read + write paths**, store buffers, dirty bits, MESI coherence — different hardware concerns.
- Different access patterns: I-cache benefits from large lines (sequential prefetch); D-cache benefits from associativity (random access patterns).

**L2 / L3** are typically **unified** (instructions and data share storage). Capacity matters more than parallel bandwidth at that level.

### Real-world examples

| CPU | L1-I | L1-D | L2 | L3 |
|-----|------|------|-----|-----|
| **Intel Core i7-12700** (Golden Cove) | 32 KB / core | 48 KB / core | 1.25 MB / core (unified) | 25 MB shared |
| **AMD Ryzen 7 7700X** (Zen 4) | 32 KB / core | 32 KB / core | 1 MB / core (unified) | 32 MB shared |
| **Apple M1** (Firestorm) | 192 KB / core | 128 KB / core | 12 MB shared (unified) | — |
| **ARM Cortex-A72** (Pi 4) | 48 KB / core | 32 KB / core | 1 MB shared (unified) | — |
| **ARM Cortex-A76** | 64 KB / core | 64 KB / core | 256–512 KB / core | up to 4 MB shared |

### Inspecting your machine

```bash
lscpu --caches
# or
cat /sys/devices/system/cpu/cpu0/cache/index*/{level,type,size,shared_cpu_list}
```

Each `indexN` directory describes one cache. Key fields:
- `level` — 1, 2, 3
- `type` — `Instruction`, `Data`, or `Unified`
- `size` — capacity
- `shared_cpu_list` — which CPUs share this cache (used by the scheduler)

**Example output** (4-core machine):

```
Caches (sum of all):
  L1d: 128 KiB (4 instances)   ← 32 KiB per core, private
  L1i: 128 KiB (4 instances)   ← 32 KiB per core, private
  L2:  16 MiB (4 instances)    ← 4 MiB per core, private, unified
  L3:  16 MiB (1 instance)     ← shared across all 4 cores
```

The "(N instances)" tells you the sharing topology — 4 instances = one per core (private); 1 instance = shared by all cores.

```
   Core 0           Core 1         Core 2           Core 3
   ┌────┐          ┌────┐          ┌────┐           ┌────┐
   │CPU │          │CPU │          │CPU │           │CPU │
   └─┬──┘          └─┬──┘          └─┬──┘           └─┬──┘
  ┌──┴──┐         ┌──┴──┐         ┌──┴──┐          ┌──┴──┐
  │L1i+D│         │L1i+D│         │L1i+D│          │L1i+D│   private, split
  └──┬──┘         └──┬──┘         └──┬──┘          └──┬──┘
  ┌──┴──┐         ┌──┴──┐         ┌──┴──┐          ┌──┴──┐
  │ L2  │         │ L2  │         │ L2  │          │ L2  │   private, unified
  └──┬──┘         └──┬──┘         └──┬──┘          └──┬──┘
     └───────────────┴───────┬───────┴────────────────┘
                             ▼
                       ┌──────────┐
                       │    L3    │   shared, unified
                       └────┬─────┘
                            ▼
                          DRAM
```

### Coherency consequence

Because L1-I and L1-D are physically separate arrays, **writes to D-cache do not automatically appear in I-cache**. This is exactly why self-modifying code / JIT engines need `flush_icache_range()` — the kernel must:

1. **Clean** the D-cache lines to L2 (where I-cache can see them — the **Point of Unification**)
2. **Invalidate** the corresponding I-cache lines so the next fetch reloads from L2

x86 hides this in hardware (snoops between L1-I and L1-D), but ARM and RISC-V expose it to software — hence the architectural difference in kernel cache APIs (see below).

---

## Cache Maintenance Operations

ARM and Linux distinguish three operations:

| Term       | Meaning                                                          |
|------------|------------------------------------------------------------------|
| **Clean**      | Write dirty lines back to next level (RAM); lines remain valid |
| **Invalidate** | Mark lines invalid; discard contents (no write-back)            |
| **Flush**      | **Clean + Invalidate** — write back, then discard               |

---

## Context Switch Behavior

### VIVT — full flush required

Same VA in different processes maps to different PAs (homonyms). The cache must be cleaned of stale entries before the new process runs:

```
1. TLB    → invalidate    (no dirty bits to write back)
2. D-cache → flush         (clean + invalidate — dirty user data must hit RAM)
3. I-cache → invalidate    (read-only, nothing to write back)
```

This per-context-switch cost is why pure VIVT is largely abandoned.

### PIPT — no cache flush needed

Physical tags disambiguate ownership automatically. Only the TLB needs invalidation (or ASID switch).

### VIPT (alias-free) — no cache flush needed

Same as PIPT for correctness; only TLB management is required.

### TLB note

TLBs have no dirty bits — entries are valid or invalid. The correct verb is **invalidate**, not "clean" or "flush." On architectures with **ASID** (Address Space ID) support, TLB entries are tagged per-process, so a context switch only needs to swap the active ASID — no full TLB invalidation required.

---

## Linux Kernel Cache Maintenance APIs

Linux has a layered cache API: portable wrappers in `asm-generic/cacheflush.h` that each arch may override.

### Portable (cross-arch) APIs

Drivers should generally use these. Each arch overrides them — no-op on coherent caches, real work on others.

| API | Purpose |
|-----|---------|
| `flush_cache_all()` | Flush entire cache (rarely needed) |
| `flush_cache_mm(mm)` | Flush cache for an address space |
| `flush_cache_range(vma, start, end)` | Flush cache for a VMA range |
| `flush_cache_page(vma, addr, pfn)` | Flush a single page |
| `flush_dcache_page(page)` / `flush_dcache_folio(folio)` | Sync D-cache for a page (kernel wrote → user maps it) |
| `flush_icache_range(start, end)` | Sync I-cache after writing instructions to memory |
| `flush_icache_user_page(vma, page, addr, len)` | Same but for user-mapped pages |
| `flush_cache_vmap(start, end)` / `flush_cache_vunmap()` | Flush around `vmalloc` mappings |

Source: `include/asm-generic/cacheflush.h`. See also `Documentation/core-api/cachetlb.rst`.

---

### x86 / x86_64

x86 caches are **PIPT and hardware-coherent** (snooping protocol), so most kernel cache APIs are **no-ops**.

| API | Notes |
|-----|-------|
| `flush_cache_*` (all generic) | **No-op** — falls through to asm-generic |
| `flush_dcache_page()` | **No-op** |
| `flush_icache_range()` | **No-op** (I/D caches coherent on x86) |
| `clflush_cache_range(addr, size)` | Flush a range one line at a time |
| `clflushopt(addr)` | Single line — non-temporal, weakly ordered |
| `clwb(addr)` | Write-back without invalidate (PMEM/NVDIMM) |
| `wbinvd()` | Write-back + invalidate **entire** cache. Kernel-only, extremely expensive |

Files: `arch/x86/include/asm/cacheflush.h`, `arch/x86/include/asm/special_insns.h`.

---

### arm64

D-cache is non-aliasing VIPT/PIPT, I-cache is potentially aliasing VIPT — so most ops do real work.

ARM uses **point-of-coherency (PoC)** vs **point-of-unification (PoU)** distinctions:
- **PoU**: where I-cache and D-cache see the same data (typically L2)
- **PoC**: where DMA / other masters see memory (typically RAM)

#### D-cache

| API | Operation |
|-----|-----------|
| `dcache_clean_poc(start, end)` | Clean to PoC (DMA write to device) |
| `dcache_inval_poc(start, end)` | Invalidate to PoC (DMA read from device) |
| `dcache_clean_inval_poc(start, end)` | **Flush** (clean + invalidate) to PoC |
| `dcache_clean_pou(start, end)` | Clean to PoU (sync with I-cache) |
| `dcache_clean_pop(start, end)` | Clean to point-of-persistence (PMEM) |

#### I-cache

| API | Operation |
|-----|-----------|
| `icache_inval_pou(start, end)` | Invalidate range to PoU |
| `icache_inval_all_pou()` | Invalidate entire I-cache (`ic ialluis`) |

#### Combined / Generic

| API | Operation |
|-----|-----------|
| `caches_clean_inval_pou(start, end)` | Clean D + invalidate I to PoU (self-modifying code) |
| `flush_icache_range(start, end)` | Wraps `caches_clean_inval_pou` + IPI for SMP sync |
| `flush_dcache_page()` / `flush_dcache_folio()` | Defer-or-flush for page-cache pages |

Files: `arch/arm64/include/asm/cacheflush.h`, `arch/arm64/mm/cache.S`.

---

### RISC-V

Base ISA only provides `fence.i` (full I-cache flush). D-cache management requires the **Zicbom** extension (Cache Block Management Operations).

#### I-cache

| API | Operation |
|-----|-----------|
| `local_flush_icache_all()` | `fence.i` on current hart |
| `flush_icache_all()` | IPI all harts → `fence.i` |
| `flush_icache_mm(mm, local)` | Flush I-cache for an mm |
| `flush_icache_range(start, end)` | Falls back to `flush_icache_all()` — base ISA has no range op |

#### D-cache (requires Zicbom extension)

Used via the `ALT_CMO_OP` macro which emits `cbo.clean` / `cbo.flush` / `cbo.inval`:

```c
ALT_CMO_OP(CLEAN, addr, size, riscv_cbom_block_size);  // cbo.clean
ALT_CMO_OP(INVAL, addr, size, riscv_cbom_block_size);  // cbo.inval
ALT_CMO_OP(FLUSH, addr, size, riscv_cbom_block_size);  // cbo.flush (clean+inval)
```

Wrappers in `arch/riscv/mm/dma-noncoherent.c` and `arch/riscv/mm/pmem.c`:

| API | Operation |
|-----|-----------|
| `arch_dma_prep_coherent()` | Clean + invalidate before DMA |
| `arch_sync_dma_for_device()` | CLEAN or FLUSH depending on direction |
| `arch_sync_dma_for_cpu()` | INVAL or FLUSH depending on direction |
| `arch_wb_cache_pmem()` | `cbo.clean` for PMEM writeback |
| `arch_invalidate_pmem()` | `cbo.inval` for PMEM invalidate |

Files: `arch/riscv/include/asm/cacheflush.h`, `arch/riscv/mm/dma-noncoherent.c`.

---

### Quick Comparison Across Architectures

| Operation | x86 | arm64 | RISC-V |
|-----------|-----|-------|--------|
| **D-cache clean (range)** | `clwb()` per line | `dcache_clean_poc()` | `ALT_CMO_OP(CLEAN, ...)` |
| **D-cache invalidate (range)** | (no direct API) | `dcache_inval_poc()` | `ALT_CMO_OP(INVAL, ...)` |
| **D-cache flush (range)** | `clflush_cache_range()` | `dcache_clean_inval_poc()` | `ALT_CMO_OP(FLUSH, ...)` |
| **D-cache flush all** | `wbinvd()` | (not exposed) | (not exposed) |
| **I-cache invalidate (range)** | no-op | `icache_inval_pou()` | `flush_icache_all()` (no range) |
| **I-cache invalidate all** | no-op | `icache_inval_all_pou()` | `flush_icache_all()` |
| **Sync D→I (self-mod code)** | no-op (coherent) | `caches_clean_inval_pou()` / `flush_icache_range()` | `flush_icache_range()` |
| **Page-level dcache flush** | no-op | `flush_dcache_page()` | `flush_dcache_page()` |

**Rule of thumb**: when writing portable driver code, use the generic `flush_*` APIs. Reach for arch-specific names only inside arch code or when implementing `dma_sync_*`-style primitives.

---

## Why It Matters for Kernel Development

- **Context switch cost**: Determines whether the scheduler must flush caches when swapping processes.
- **DMA coherency**: Drivers doing DMA must ensure cache lines are flushed/invalidated against the physical address used by hardware. The kernel's `dma_map_*` API abstracts this.
- **Self-modifying code / JIT**: Writing instructions via the D-cache requires flushing the D-cache and invalidating the I-cache before execution, since I-cache and D-cache are not coherent on most ARM cores.
- **`flush_cache_*` kernel APIs**: Exist precisely to handle architecture-specific cache scheme requirements.
