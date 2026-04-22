---
tags:
  - career
  - jobs
  - interview
---

# SiFive — System Software Engineer (Linux) Interview Prep

**Job:** [System Software Engineer to Senior Staff Engineer (Linux)](https://www.104.com.tw/job/8c3v7)
**Company:** 美商賽發馥股份有限公司臺灣分公司
**Location:** 新竹縣竹北市復興三路二段168號10樓之3（部分遠端）
**Updated:** 2026-04-22

---

## SiFive Business Model

SiFive is a **fabless IP licensing company**, not a chip manufacturer. Think of them as the "ARM of RISC-V":

| | |
|---|---|
| **What they sell** | RISC-V processor IP (Core IP blocks) |
| **How they make money** | Upfront licensing fees + royalties per chip sold |
| **Who buys** | SoC companies, chip designers (automotive, data center, AI edge, IoT) |
| **What they don't do** | Fabricate chips — customers tape out using SiFive IP |

**Product lines:**
- **SiFive Essential** — embedded/IoT cores
- **SiFive Performance** — application-class Linux-capable cores
- **SiFive Intelligence** — AI/ML accelerator cores (RVV-based)
- **SiFive Automotive** — safety-certified (ISO 26262) cores

**Scale:** 400+ design wins, 2B+ devices shipped using SiFive IP, 85% engineers on staff.

**Why Taiwan?** Close to TSMC, MediaTek, Realtek, and the broader SoC ecosystem that licenses their IP.

---

## Role Breakdown

**Core work:**
- Linux kernel + device drivers for SiFive's RISC-V processors
- OpenSBI (RISC-V firmware), U-Boot, Yocto/OpenEmbedded
- Upstream contributions to mainline Linux
- Co-design with hardware/architecture teams

**Key signals in the JD:**
- They want **upstream Linux kernel** experience (not just internal forks)
- JTAG + GDB debugging of **multicore RISC-V** systems
- Experience with IOMMUs, power management, SoC platform security
- English fluency: **精通** — interviews will be in English

---

## Phone Interview Prep

### 1. Linux Kernel & Device Drivers ✅ (your strongest area)

- Explain a driver you've written: `probe()`, `remove()`, `file_operations`, `platform_driver`
- Difference between character, block, and platform devices
- Interrupt handling: `request_irq()`, threaded IRQs, softirqs
- Memory management: `kmalloc` vs `vmalloc`, DMA mapping, `ioremap`

### 2. RISC-V Architecture ⚠️ (prep needed)

- Privilege levels: **M-mode, S-mode, U-mode**
- **OpenSBI** = RISC-V equivalent of ARM TF-A — runs in M-mode, provides SBI calls to kernel in S-mode
- RISC-V Vector extension (RVV) — understand the concept
- How RISC-V differs from x86/ARM: modular, no mandatory ISA bloat

### 3. Boot Flow

```
ROM → U-Boot SPL → OpenSBI (M-mode) → U-Boot proper → Linux kernel (S-mode)
```

Know each stage's role and how they hand off control.

### 4. Upstream Kernel Workflow

- Patch submission: `git format-patch`, `git send-email`, LKML, maintainer trees
- Review tags: `Acked-by`, `Reviewed-by`, `Tested-by`
- Merge window cycle (rc1–rc8, stable releases)
- Be honest if no upstream experience — show you understand the process

### 5. Debugging ✅ (your strong suit)

- GDB + JTAG (`openocd` for RISC-V) — review workflow even if mainly used GDB + kdump
- Kernel panic without kdump: serial console, early printk, KASAN, lockdep

### 6. Frame Your ORCA Work

| Your Experience | How It Maps |
|---|---|
| PowerPC / s390x porting | Cross-arch thinking — exact skill for RISC-V bring-up |
| Retpoline / Intel CET | Security-aware kernel development |
| ELF / LLVM relocation | Toolchain + kernel interaction |
| Crash dump / kernel panic RCA | Multicore debugging |

---

## Behavioral Questions to Prepare

> SiFive values: *"open, honest, and direct communication"* and *"strong communicators and listeners."*

- "Tell me about a kernel patch you wrote and how you handled review feedback."
- "Describe a time you debugged a complex multicore issue."
- "How do you approach co-designing software with a hardware team?"

---

## Quick Study List

| Topic | Resource |
|---|---|
| RISC-V privilege spec (M/S/U mode, SBI) | https://riscv.org/specifications |
| OpenSBI internals | https://github.com/riscv-software-src/opensbi |
| Upstream kernel contribution flow | https://kernel.org/doc/html/latest/process |
| U-Boot + Yocto basics | Official docs |
| SiFive product portfolio | https://www.sifive.com/products |

---

## Assessment

Your kernel internals background is **genuinely strong** for this role. Main prep gap: get comfortable with **RISC-V-specific terminology** (SBI, privilege modes, RVV) so you can speak the language in the interview.
