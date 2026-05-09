---
tags:
  - linux-kernel
  - cache
  - RISC-V
  - interview
  - embedded-C
related:
  - DMA and Cache Coherency
  - SiFive - Coding Interview Problems
---

# Manual Cache Control — Beyond DMA and JIT

> When else does the kernel need to issue **clean / invalidate / flush** explicitly? Useful framing for SiFive interviews — see [[SiFive - Coding Interview Problems]] Tier 6.

The pattern is always the same: **two agents see the same physical address through different paths, and at least one path bypasses or pre-dates the cache.**

---

## The three diagnostic questions

If you answer **yes** to any of these, someone has to issue cache ops + a barrier:

1. **Producer-consumer pair where one path bypasses the cache?** → DMA, JIT, persistence, accelerator.
2. **Either side changes attributes / permissions / encryption on the same physical address?** → `mprotect`, page poisoning, CoVE/SEV/TDX.
3. **One side about to go away or be lost?** → `cpu_die`, kexec, suspend, kdump.

On RISC-V the primitives are `cbo.clean` / `cbo.inval` / `cbo.flush` + `fence` (and `fence.i` for code). On SMP they're hart-local → IPI everyone else.

---

## 1. Self-modifying code (kernel flavor) — beyond JIT

JIT is the userspace flavor. The kernel does the same thing constantly:

| Mechanism                     | What it writes                  | Where in tree                       |
| ----------------------------- | ------------------------------- | ----------------------------------- |
| **ftrace**                    | nop ↔ call to tracer            | `kernel/trace/ftrace.c` + arch glue |
| **kprobes**                   | first byte of fn → breakpoint   | `arch/riscv/kernel/probes/`         |
| **jump_label / static_key**   | branch ↔ nop based on toggle    | `kernel/jump_label.c`               |
| **alternative instructions**  | hot-patch CPU-feature variants  | `arch/riscv/kernel/alternative.c`   |
| **module loader**             | runs relocations after load     | `kernel/module/main.c`              |
| **kgdb / hw breakpoints**     | sw breakpoint inserts           | `kernel/debug/`                     |
| **livepatch (kpatch)**        | redirect old fn to patched one  | `kernel/livepatch/`                 |

All of them write through the **D-cache** (the kernel writes via normal stores) and need the **I-cache** to drop its stale copy before that hart re-fetches. The arch primitive is `flush_icache_range(start, end)` (RISC-V: `arch/riscv/mm/cacheflush.c`).

### The four-step sync sequence

Every self-modifying-code path executes the same four logical steps in order:

```
1. Write new instruction          → lands in D-cache (writeback mode)
2. Clean / writeback D-cache      → push to a level the I-cache can see
3. Invalidate I-cache             → drop any stale fetched copy
4. Flush instruction pipeline     → discard already-decoded stale ops
```

Step 4 is easy to forget. Even after the I-cache is clean, the **fetch / decode / branch-predictor pipeline can still hold the stale instruction**. So an explicit pipeline-flushing instruction is required after I-cache invalidate (`isb` on ARM, `fence.i` on RISC-V — `fence.i` does both invalidate + pipeline flush, see below).

### Three subtleties

**(a) The pipeline matters.** I-cache invalidate alone isn't sufficient — the prefetch queue, decode buffers, and branch predictor may already hold the old bytes. Always pair with a pipeline-sync instruction.

**(b) Multi-hart invalidation is a separate problem.** The store from CPU0 reaches DRAM via cache coherency just fine. But CPU1's *I-cache* doesn't snoop CPU0's stores on most weakly-coherent ISAs. Every hart that might fetch from this address needs its **own** I-cache invalidate, delivered via IPI.

**(c) Architectural cost varies enormously.**

| Arch | Single-core SMC cost | Cross-core SMC cost |
|------|----------------------|----------------------|
| **x86** | ~free — I-cache snoops D-cache; just need a serializing instruction | Mostly free — coherency handles cross-core |
| **ARM/AArch64** | Cheap — `dc cvau` + `ic ivau` + `dsb ish` + `isb`, broadcast in inner-shareable domain | Cheap — `ic ivau` is broadcast, no IPI needed |
| **RISC-V** | Cheap — `fence.i` (hart-local) | **Expensive** — SBI `remote_fence_i` ecall + IPI to every other hart |

This is why RISC-V is the architecture you have to think hardest about for SMC.

### RISC-V — concrete single-hart sequence

On a SiFive core where I-cache and D-cache are coherent through L2 (most modern parts), `fence.i` alone is enough — the D-cache writeback to L2 is exactly what `fence.i` waits for, and the I-cache fetches from L2:

```asm
    sw      a1, 0(a0)        # store new instruction at *a0
    fence.i                  # invalidate local I-cache + flush pipeline
                             # subsequent fetches from this hart see new bytes
```

On a weakly-coherent embedded RISC-V where the I-cache only sees DRAM (not L2), you need explicit Zicbom ops to push the line all the way out:

```asm
    sw      a1, 0(a0)        # store new instruction
    cbo.clean (a0)           # writeback D-cache line to a level I-cache sees
    fence rw, rw             # ensure store globally visible
    fence.i                  # invalidate local I-cache + flush pipeline
```

The kernel detects which path to take from the device tree (`riscv_cbom_block_size`) and the ISA-extension flags (`Zicbom`, `Zicboz`).

What each primitive actually does:

- **`cbo.clean`** (Zicbom) — writes back the D-cache line if dirty, leaves it in cache. (`cbo.flush` = writeback + invalidate; rarely needed for SMC.)
- **`fence.i`** (base ISA) — guarantees subsequent instruction fetches from *this hart* observe all prior stores from *this hart*. Implementation typically invalidates the local I-cache and flushes the fetch pipeline. **Hart-local** — does not propagate.
- **`fence rw, rw`** — global memory ordering; ensures the store is visible to other observers before `fence.i`.

### RISC-V — multi-hart variant

```asm
    sw      a1, 0(a0)        # store new instruction
    fence   rw, rw           # ensure store globally visible
    fence.i                  # this hart's I-cache + pipeline
    # Other harts still have stale I-caches. Ask SBI to broadcast:
    li      a7, SBI_EXT_RFENCE
    li      a6, SBI_EXT_RFENCE_REMOTE_FENCE_I
    li      a0, -1           # hart_mask = all harts
    ecall                    # OpenSBI sends IPIs; each remote hart
                             # executes fence.i locally
```

That `sbi_remote_fence_i` ecall is what makes multi-hart SMC expensive on RISC-V — every hart that might run this code takes an IPI and executes its own `fence.i`. A `kprobe` insertion or live patch on a 16-core SiFive part means **15 IPIs and 15 pipeline flushes** on top of the local writeback + invalidate.

### What Linux abstracts away

Single API, arch-specific implementation:

```c
void flush_icache_range(unsigned long start, unsigned long end);
```

Used by every self-modifying mechanism in the kernel: `ftrace`, `kprobes`, `jump_label`, `alternatives`, the module loader, `bpf_jit_binary_lock_ro`, `kgdb`, `livepatch`.

On RISC-V, `arch/riscv/mm/cacheflush.c` implements it roughly as:

```c
void flush_icache_range(unsigned long start, unsigned long end)
{
    asm volatile ("fence.i" ::: "memory");

    if (num_online_cpus() > 1) {
        if (riscv_use_ipi_for_rfence())
            on_each_cpu(ipi_remote_fence_i, NULL, 1);
        else
            sbi_remote_fence_i(NULL);
    }
}
```

On ARM64 it expands to `dc cvau` + `dsb ish` + `ic ivau` + `dsb ish` + `isb`. On x86 it's essentially a no-op (the architecture handles SMC implicitly; only a serializing instruction is needed if the modified code might run *immediately* on the same CPU).

### So the corrected mental model

Yes, the canonical sequence is **D-cache writeback → I-cache invalidate → pipeline flush**. On RISC-V this collapses into `fence.i` if I/D caches share a coherent point (most SiFive cores), or `cbo.clean` + `fence.i` if they don't. **Plus** on multi-hart systems, every other hart that might fetch from this address needs its own `fence.i` — delivered via SBI `remote_fence_i` (an IPI under the hood), because `fence.i` is hart-local.

---

## 2. CPU power-state transitions — hotplug, suspend, deep idle

Before a hart can be powered off, **dirty L1 lines must reach a domain that survives**. Otherwise the data is gone with the cache.

```
cpu_die() / suspend / WFI-with-power-collapse
       │
       ▼
flush_cache_all() / clean+invalidate L1
       │
       ▼
inform L2 (or the platform — SBI HSM call on RISC-V)
       │
       ▼
power off
```

- RISC-V: `arch/riscv/kernel/cpu-hotplug.c` calls into **SBI HSM** (`sbi_hart_stop`).
- ARM: `flush_cache_louis()` ("level of unification, inner shareable").
- Skipping it silently corrupts whatever variable lived in that line — extremely fun to debug.

---

## 3. Firmware / boot-stage handoffs

```
ROM → U-Boot SPL → OpenSBI → U-Boot proper → Linux
                  (M-mode)                  (S-mode)
```

Each transition can change:
- whether caches are enabled at all,
- the memory-attribute mapping (**PMA / PBMT** on RISC-V),
- whether the MMU is on.

Every stage that touches its successor's image must **clean** caches so the next stage reads from DRAM, and may need to **invalidate** to drop pre-MMU stale lines. See OpenSBI `lib/sbi/sbi_hart.c` and Linux `arch/riscv/kernel/head.S`. This is exactly the seam SiFive's BSP team owns.

---

## 4. `kexec` / `kdump`

Same idea as boot handoff, but at runtime. Before `kexec_arch_jump()` enters the new kernel, the old kernel must clean caches so the new one (which often boots with caches off briefly) sees the loaded image:

```
arch/riscv/kernel/machine_kexec.c
   → riscv_kexec_norelocate / riscv_kexec_relocate
   → flushes caches before sret-style jump
```

**kdump** twist: the *crash kernel* needs the *crashed* memory captured faithfully. If dirty lines from the crashed kernel are stranded in L1 of an offline hart, the dump misses them. That's why `crash_save_cpu()` includes cache flushes.

---

## 5. Persistent memory — NVDIMM, CXL.mem, pmem/DAX

Writes hitting the cache aren't durable. To survive power loss they must reach the **persistence domain** (battery-backed buffers / NVDIMM / flash):

```c
memcpy(pmem_addr, src, len);
arch_wb_cache_pmem(pmem_addr, len);   // CLWB on x86, cbo.clean on RISC-V
pmem_drain();                          // sfence / fence — ordering wrt persistence
```

This is what `libpmem` / `libnvdimm` / DAX filesystems wrap. Wrong ordering or a missed flush leaves the filesystem "atomically" inconsistent after a crash. RISC-V uses **Zicbom + Zicboz** (and planned **Zicbop**).

---

## 6. Memory-encryption / confidential-compute transitions

When a page transitions between trust domains — **AMD SEV-SNP**, **Intel TDX**, **ARM CCA**, **RISC-V CoVE** — cache tags include an encryption/world identifier. Going **guest-private → host-shared**:

1. Guest cleans+invalidates the page.
2. Hypervisor flips the page attribute.
3. Host invalidates the page (previous tag's lines are now unreachable but might still be present).

Miss either step → cross-world data leakage or a crash on next access. RISC-V CoVE is still landing upstream; the cache-management contract is already in the spec.

---

## 7. Non-coherent accelerators on shared memory

Strict extension of the DMA story but worth pulling out: many SoCs put an **FPGA, vision DSP, or NPU** on the same DRAM but outside the cache-coherence fabric. From the CPU's view it's "DMA"; from the device's view "the CPU keeps a stale copy of my output." Same `dma_sync_*` primitives, but custom ioctls are common because the device lifecycle is more complex than a network packet.

---

## 8. Niche but real

- **Cache locking / preloading** for hard-real-time. Pull lines into L1/L2 and lock them so worst-case latency is bounded. SiFive 7-series and many automotive cores expose this.
- **Page poisoning + secret scrubbing.** After zeroing a page that held key material, a flush guarantees the zeros (not the secret) are what survives in DRAM. Used by `crypto/`, `keys/`, KASAN debug modes.
- **Live page-attribute changes** — `set_memory_ro`, `set_memory_x`, `change_page_attr`. RW → RX requires flush + I-cache invalidate the range, else old D-cache writes might still be in flight when I-cache starts fetching.
- **SMP IPI-driven cache ops.** RISC-V `fence.i` and `cbo.*` are hart-local. Any of the above on SMP becomes "do it locally + IPI everyone else" — `flush_icache_all()` and `smp_call_function` patterns. Easy to get wrong.

---

## RISC-V cheat sheet

| Need                              | Instruction                          | Spec                |
| --------------------------------- | ------------------------------------ | ------------------- |
| Clean (writeback)                 | `cbo.clean`                          | Zicbom              |
| Invalidate                        | `cbo.inval`                          | Zicbom              |
| Flush (clean + invalidate)        | `cbo.flush`                          | Zicbom              |
| Zero a cache block                | `cbo.zero`                           | Zicboz              |
| Prefetch hints                    | `prefetch.{i,r,w}`                   | Zicbop              |
| I-cache invalidate (this hart)    | `fence.i`                            | base ISA            |
| Memory ordering                   | `fence rw,rw` / typed predecessor    | base ISA + RVWMO    |
| Cross-hart code sync              | `fence.i` + IPI all harts to do same | (Linux: `flush_icache_all()`) |
| SBI cross-hart code sync          | `sbi_remote_fence_i(hart_mask)`      | SBI RFENCE ext      |
| TLB / translation cache           | `sfence.vma`                         | privileged spec     |

Linux entry points: `arch/riscv/mm/cacheflush.c`, `arch/riscv/mm/dma-noncoherent.c`, `arch/riscv/mm/tlbflush.c`.

---

## Cross-references

- [[DMA and Cache Coherency]]
- [[SiFive - Computer Architecture Prep]] — RVWMO, MESI, fence semantics
- [[SiFive - Coding Interview Problems]] — Tier 6 driver scenarios
- [[CPU Control Hazard]]
