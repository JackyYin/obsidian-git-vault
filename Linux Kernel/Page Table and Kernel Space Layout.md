---
tags:
  - linux-kernel
  - memory-management
  - page-table
  - virtual-memory
  - MMU
---

# Page Table and Kernel Space Layout

---

## Bit Math — What One PGD Entry Covers

```
3-level page table (e.g. RISC-V Sv39):

  Virtual Address (39 bits):
  ┌───────────┬───────────┬───────────┬────────────┐
  │  VPN[2]   │  VPN[1]   │  VPN[0]   │   offset   │
  │  9 bits   │  9 bits   │  9 bits   │  12 bits   │
  │   (PGD)   │   (PMD)   │   (PTE)   │            │
  └───────────┴───────────┴───────────┴────────────┘

One PGD entry covers:
  VPN[1] + VPN[0] + offset = 9 + 9 + 12 = 30 bits
  2^30 = 1GB per PGD entry
```

### x86_64 (4-level / Sv48)

```
4-level page table (x86_64, 48-bit canonical):

  Virtual Address (48 bits, sign-extended to 64):
  ┌──────────┬──────────┬──────────┬──────────┬────────────┐
  │ PGD idx  │ PUD idx  │ PMD idx  │ PTE idx  │   offset   │
  │ 9 bits   │ 9 bits   │ 9 bits   │ 9 bits   │  12 bits   │
  │ (PGD)    │ (PUD)    │ (PMD)    │ (PTE)    │            │
  └──────────┴──────────┴──────────┴──────────┴────────────┘

Per-level coverage:
  One PTE entry → 4 KB              (standard page)
  One PMD entry → 2 MB              (large page "2M")
  One PUD entry → 1 GB              (huge page "1G")
  One PGD entry → 512 GB
  Whole PGD     → 256 TB            (512 entries × 512 GB)

The PGD itself is a single 4 KB page holding 512 × 8-byte entries.
```


---

## Does All Kernel Space Share One PGD Entry?

**It depends on the architecture.**

### 32-bit Linux (3:1 split) — YES

```
32-bit address space = 4GB total
  User space:   0x00000000 – 0xBFFFFFFF  = 3GB  (3 PGD entries)
  Kernel space: 0xC0000000 – 0xFFFFFFFF  = 1GB  (1 PGD entry)

  PGD table (4 entries total on 32-bit PAE):
  ┌─────────────────────────────────┐
  │ PGD[0]  covers 0GB–1GB  (user)  │
  │ PGD[1]  covers 1GB–2GB  (user)  │
  │ PGD[2]  covers 2GB–3GB  (user)  │
  │ PGD[3]  covers 3GB–4GB  (kernel)│ ◀── all kernel in ONE entry
  └─────────────────────────────────┘

  → YES, all kernel space fits in PGD[3] on 32-bit ✅
```

### 64-bit Linux — NO, kernel spans many PGD entries

```
RISC-V Sv39 (3-level) address space:
  Total:        512GB
  User space:   lower 256GB
  Kernel space: upper 256GB

  Each PGD entry covers 1GB
  Kernel space = 256GB / 1GB = 256 PGD entries

  PGD table (512 entries):
  ┌──────────────────────────────────────────┐
  │ PGD[0]   covers 0GB   – 1GB   (user)     │
  │ PGD[1]   covers 1GB   – 2GB   (user)     │
  │  ...                                     │
  │ PGD[255] covers 255GB – 256GB (user)     │
  │ PGD[256] covers 256GB – 257GB (kernel)   │◀──┐
  │ PGD[257] covers 257GB – 258GB (kernel)   │   │
  │  ...                          (kernel)   │   │ 256 kernel
  │ PGD[511] covers 511GB – 512GB (kernel)   │◀──┘ PGD entries
  └──────────────────────────────────────────┘
```


```
64-bit x86 (Sv48 / 4-level) address space:
  User space:   0x0000000000000000 – 0x00007FFFFFFFFFFF  = 128TB
  Kernel space: 0xFFFF800000000000 – 0xFFFFFFFFFFFFFFFF  = 128TB

  Each PGD entry covers 512GB (on 4-level)
  Kernel space = 128TB / 512GB = 256 PGD entries

  PGD table (512 entries total, one 4 KB page):
  ┌──────────────────────────────────────────────────────────┐
  │ PGD[0]    covers 0GB     – 512  GB                (user) │
  │ PGD[1]    covers 512GB   - 1024 GB                (user) │
  │  ...                                                     │
  │ PGD[255]  covers 127.5TB - 128  TB                (user) │
  │ PGD[256]  covers 128  TB - 128.5TB              (kernel) │
  │ PGD[257]                                        (kernel) │
  │  ...                                                     │
  │ PGD[511]  covers 255.5TB – top                  (kernel) │
  └──────────────────────────────────────────────────────────┘
```

### x86_64 kernel-half layout (4-level)

```
  Address (start)            Region
  ─────────────────────────  ───────────────────────────────────
  0xFFFF_8880_0000_0000      direct map of all physical memory
  0xFFFF_C900_0000_0000      vmalloc / ioremap area
  0xFFFF_EA00_0000_0000      virtual memory map (vmemmap)
  0xFFFF_EC00_0000_0000      KASAN shadow memory
  0xFFFF_FE00_0000_0000      cpu_entry_area mapping
  0xFFFF_FFFF_8000_0000      kernel text / data image (.text, .data, .bss)
  0xFFFF_FFFF_A000_0000      modules area
  0xFFFF_FFFF_FF00_0000      fixmap / percpu / vsyscall
```

Authoritative table: `Documentation/x86/x86_64/mm.rst`.

---

## The More Important Insight — Shared Kernel PGDs

Even though kernel space spans many PGD entries, all processes share the **same** kernel PGD entries:

```
Process A page table          Process B page table
┌──────────────────┐          ┌──────────────────┐
│ PGD[0]  (user)   │          │ PGD[0]  (user)   │
│ PGD[1]  (user)   │          │ PGD[1]  (user)   │
│  ...    (user)   │          │  ...    (user)   │
│ PGD[256](kernel) │──┐    ┌──│ PGD[256](kernel) │
│ PGD[257](kernel) │──┼────┼──│ PGD[257](kernel) │
│  ...             │  │    │  │  ...             │
│ PGD[511](kernel) │──┘    └──│ PGD[511](kernel) │
└──────────────────┘    │     └──────────────────┘
                        │
                        ▼
              SAME physical PMD pages
              (kernel mappings shared
               across ALL processes)

User PGD entries     → unique per process (each process's own memory)
Kernel PGD entries   → point to SAME physical pages for all processes
```

---

## Why Share Kernel PGDs?

```
If kernel had its own separate page table:

  syscall → switch CR3/satp to kernel page table  ← expensive TLB flush
  return  → switch CR3/satp back to user table    ← expensive TLB flush

With kernel mapped in every process:

  syscall → no page table switch needed ✅
  return  → no page table switch needed ✅
  only privilege level changes (U-mode → S-mode)
```

This is why:
- A **context switch** only swaps the user portion of the page table
- Kernel code and data are **always mapped** in every process's address space
- A **syscall** doesn't need to switch page tables — kernel is already mapped

---

## Meltdown & KPTI

```
Meltdown exploited this design:
  kernel memory was mapped (but protected) in every process
  speculative execution could read kernel memory before
  permission check completed → kernel data leaked to userspace

Fix — KPTI (Kernel Page Table Isolation):
  user-mode page table  → kernel PGD entries REMOVED
  kernel-mode page table → full mapping restored

  syscall → switch to kernel page table  (TLB flush)
  return  → switch to user page table   (TLB flush)

  Cost: ~5–30% performance regression on syscall-heavy workloads
  Benefit: kernel memory no longer speculatively accessible
```

### x86_64 KPTI dual-PGD layout

```
Each mm allocates TWO 4 KB PGD pages back-to-back (order-1 allocation):

  mm->pgd ─→ ┌──────────────────────────────┐  ← page 0  (kernel PGD)
             │  entries 0..255   (user)     │  ← process's user mappings
             │  entries 256..511 (kernel)   │  ← FULL kernel half
             └──────────────────────────────┘
             ┌──────────────────────────────┐  ← page 1  (user PGD)
             │  entries 0..255   (user)     │  ← same user mappings, mirrored
             │  entries 256..511 (kernel)   │  ← ONLY entry trampoline,
             │                              │    IDT, SYSCALL stub,cpu_entry_area
             └──────────────────────────────┘

The two PGDs sit at addresses differing by exactly one bit: bit 12 (= PAGE_SIZE).
Helper: kernel_to_user_pgdp(p) = (pgd_t *)((unsigned long)p | (1 << 12)).
```

```
CR3 layout under KPTI (+ PCID):

  ┌────────────────────────────────────────────┐
  │ bits 63–12 : physical address of PGD page  │
  │ bit  12    : 0 = kernel PGD, 1 = user PGD  │ ← KPTI switch bit
  │ bits 11– 0 : PCID (Process Context ID)     │
  └────────────────────────────────────────────┘

Entry/exit trampoline (arch/x86/entry/entry_64.S):
  syscall entry  : AND CR3, ~(1 << 12)  → kernel PGD
  return to user : OR  CR3,  (1 << 12)  → user PGD

One bit flip = page-table swap. No table lookup needed.
PCID lets the swap skip the TLB flush.
```

---

## Summary

```
                    32-bit (1GB kernel)    64-bit (128TB+ kernel)
                    ───────────────────    ──────────────────────
PGD entries used    1 entry for kernel     256+ entries for kernel
by kernel space

All kernel in       ✅ YES                 ❌ NO — spans many PGD
one PGD entry?                             entries

Kernel PGDs         ✅ shared across       ✅ shared across
shared across       all processes          all processes
processes?
```
