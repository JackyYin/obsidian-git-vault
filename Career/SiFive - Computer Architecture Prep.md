---
tags:
  - career
  - interview
  - computer-architecture
  - RISC-V
---

# SiFive Interview — Computer Architecture Prep

---

## 1. RISC-V Architecture Fundamentals

### Privilege Levels
```
Machine Mode    (M-mode)   ← highest privilege
                           OpenSBI runs here
                           full hardware access
                           handles traps from lower modes

Supervisor Mode (S-mode)   ← OS level
                           Linux kernel runs here
                           uses SBI calls to reach M-mode
                           controls S-mode page tables

User Mode       (U-mode)   ← application level
                           userspace processes
                           no direct hardware access
```

### RISC-V vs ARM vs x86

```
                RISC-V          ARM             x86
                ──────────      ──────────      ──────────
ISA type        RISC, open      RISC, licensed  CISC, proprietary
Encoding        fixed 32-bit    fixed 32-bit    variable length
                (+ 16-bit C ext)(+ Thumb 16-bit)(1–15 bytes)
Registers       32 × 64-bit     31 × 64-bit     16 × 64-bit
Extensions      modular (IMAFDC)fixed profiles  fixed
Privilege spec  M/S/U           EL0–EL3         ring 0–3
Vector          RVV (flexible   SVE/NEON        AVX-512
                vlen)
Endianness      little (default)little          little
Memory model    RVWMO (relaxed) weakly ordered  TSO (strong)
```

---

## 2. CPU Pipeline

```
Classic 5-stage pipeline:

  IF           ID         EX        MEM       WB
  ──────       ──────     ──────    ──────    ──────
  Instruction  Decode     Execute   Memory    Write
  Fetch        registers  ALU/FPU   read/     back to
  from         read       branch    write     registers
  I-cache      operands   resolve


Hazards:
  ┌────────────────┬───────────────────────────────────────────────┐
  │ Data hazard    │ instruction needs result of previous one      │
  │                │ → solved by forwarding / stalls               │
  ├────────────────┼───────────────────────────────────────────────┤
  │ Control hazard │ branch outcome unknown at fetch time          │
  │                │ → solved by branch prediction                 │
  ├────────────────┼───────────────────────────────────────────────┤
  │ Structural     │ two instructions need same hardware unit      │
  │ hazard         │ → solved by stalls or duplication             │
  └────────────────┴───────────────────────────────────────────────┘
```

> Memory barriers (`smp_mb()`, `rmb()`, `wmb()`) exist because of pipeline reordering. Understanding pipelines explains *why* you need them in drivers.

---

## 3. Memory Hierarchy & Cache Architecture

```
CPU Registers    ~0 cycles        handful of bytes
      │
L1 I-cache       1–4 cycles       32–64KB   (per core)
L1 D-cache       1–4 cycles       32–64KB   (per core)
      │
L2 cache         10–20 cycles     256KB–4MB (per core or shared)
      │
L3 cache         30–50 cycles     8–64MB    (shared across cores)
      │
DRAM             100–300 cycles   GBs
      │
NVMe / Storage   100,000+ cycles  TBs


Cache line = 64 bytes (typical)

  ┌────────────────────────────────────────────────────────────┐
  │  Cache Line (64B)                                          │
  │                                                            │
  │  Tag  │ Valid │ Dirty │  Data (64 bytes of actual content) │
  │       │  bit  │  bit  │                                    │
  └────────────────────────────────────────────────────────────┘
```

**Key concepts:**
- **Cache associativity:** direct-mapped vs set-associative vs fully-associative
- **Write policies:** write-through vs write-back
- **Eviction policies:** LRU, PLRU, random
- **False sharing:** two cores writing different data on the same cache line → performance killer in drivers

---

## 4. Cache Coherency (Critical for Multi-core)

```
  Core 0                Core 1
  ──────                ──────
  L1 cache              L1 cache
  [x = 5]               [x = ?]   ← stale copy?
     │                     │
     └──────────┬───────────┘
                │
           Coherency Protocol
           (MESI / MOESI)
                │
          L2 / L3 cache
                │
              DRAM
              [x = 5]


MESI States per cache line:
  M  Modified   → only this cache has it, dirty (differs from RAM)
  E  Exclusive  → only this cache has it, clean
  S  Shared     → multiple caches have it, clean
  I  Invalid    → stale, must fetch from elsewhere


What happens when Core 1 writes to x while Core 0 has it Shared:
  Core 1 issues  RFO (Request For Ownership)
  Core 0's copy  transitions  S → I  (invalidated)
  Core 1's copy  transitions  S → M  (modified)
```

---

## 5. RISC-V Memory Model (RVWMO)

```
x86 (TSO — Total Store Order)     RISC-V (RVWMO — Relaxed)
──────────────────────────────    ──────────────────────────
Stores are seen in order          Stores can be reordered
Loads are seen in order           Loads can be reordered
Few explicit barriers needed      Must use FENCE instruction explicitly


RISC-V FENCE instruction:
  FENCE  predecessor, successor

  FENCE  r,  r   → load  → load  ordering
  FENCE  r,  w   → load  → store ordering
  FENCE  w,  w   → store → store ordering
  FENCE  rw, rw  → full barrier (equivalent to smp_mb())


Maps to Linux kernel barriers:
  smp_rmb()  →  FENCE r,r
  smp_wmb()  →  FENCE w,w
  smp_mb()   →  FENCE rw,rw
```

---

## 6. Virtual Memory & MMU

```
Virtual Address (39-bit Sv39, common in RISC-V Linux)

  ┌──────────────┬────────────┬───────────┬───────────┬───────────┐
  │   sign ext   │   VPN[2]   │   VPN[1]  │   VPN[0]  │   offset  │
  │   bits 63-39 │  bits 30-38│ bits 21-29│ bits 12-20│ bits 0-11 │
  │    (25 bits) │  (9 bits)  │  (9 bits) │  (9 bits) │ (12 bits) │
  └──────────────┴────────────┴───────────┴───────────┴───────────┘
         │              │           │           │
         │         Page Global   Page Middle  Page Table
         │         Directory     Directory    Entry
         │         (PGD)         (PMD)        (PTE)
         ▼
  3-level page table walk → physical address + 12-bit offset


TLB (Translation Lookaside Buffer):
  cache of recent virtual→physical translations
  RISC-V: software-managed TLB flush via SFENCE.VMA
  Linux:  flush_tlb_*() functions call SFENCE.VMA
```

---

## 7. Interrupts & Trap Handling

```
RISC-V Interrupt/Trap flow:

  Exception or interrupt occurs
        │
        ▼
  CPU saves:
    pc   → mepc / sepc    (return address)
    mode → mstatus/sstatus (previous privilege)
    cause→ mcause / scause (what happened)
    addr → mtval / stval   (faulting address)
        │
        ▼
  Jump to trap handler (mtvec / stvec)
        │
        ├── Synchronous exception  (e.g. page fault, illegal instruction)
        │     → handle, fix, mret/sret to resume
        │
        └── Asynchronous interrupt (e.g. timer, external IRQ via PLIC)
              → handle ISR, clear interrupt, mret/sret


RISC-V Interrupt Controllers:
  CLINT  (Core Local Interruptor)   ← timer + software interrupts
  PLIC   (Platform Level Interrupt Controller) ← external IRQs (GPIO, UART...)
```

---

## 8. Topics Mapped to Interview Likelihood

```
Topic                           Likelihood    Your Current Level
──────────────────────────────  ──────────    ──────────────────
RISC-V privilege levels         ⭐⭐⭐⭐⭐    ⚠️  prep needed
RISC-V vs ARM/x86 differences   ⭐⭐⭐⭐⭐    ⚠️  prep needed
Memory barriers / RVWMO         ⭐⭐⭐⭐⭐    ✅  strong (Retpoline/CET)
Cache coherency / MESI          ⭐⭐⭐⭐      ✅  strong
Virtual memory / page tables    ⭐⭐⭐⭐      ✅  strong (page table walking)
Pipeline hazards                ⭐⭐⭐        ⚠️  review
TLB management / SFENCE.VMA     ⭐⭐⭐⭐      ⚠️  RISC-V specific
CLINT / PLIC interrupt model    ⭐⭐⭐⭐      ⚠️  prep needed
Cache hierarchy / false sharing ⭐⭐⭐        ✅  likely familiar
RISC-V Vector (RVV)             ⭐⭐          ⚠️  conceptual only needed
```

---

## 9. Suggested Study Order

```
Week 1 — RISC-V specifics
  └── Privilege spec (M/S/U mode)
  └── SBI interface (what OpenSBI provides)
  └── CLINT + PLIC interrupt model
  └── SFENCE.VMA + TLB management
  └── Sv39 page table format

Week 2 — Architecture depth
  └── RVWMO memory model + FENCE instruction
  └── MESI coherency protocol
  └── Pipeline hazards + branch prediction basics
  └── Cache line false sharing in driver code
```

---

## 10. Key Resources

| Resource | Link |
|---|---|
| RISC-V Privileged Specification | https://github.com/riscv/riscv-isa-manual |
| OpenSBI source | https://github.com/riscv-software-src/opensbi |
| Linux RISC-V arch code | https://github.com/torvalds/linux/tree/master/arch/riscv |
| SiFive product portfolio | https://www.sifive.com/products |

> **Best single resource:** The official RISC-V Privileged Specification — SiFive engineers wrote parts of it. Reading chapters 1–4 shows you speak their language.
