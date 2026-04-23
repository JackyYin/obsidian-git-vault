---
tags:
  - career
  - interview
---

# SiFive — System Software Engineer (Linux) Interview Prep

**Job:** [System Software Engineer to Senior Staff Engineer (Linux)](https://www.104.com.tw/job/8c3v7)
**Company:** 美商賽發馥股份有限公司臺灣分公司
**Location:** 新竹縣竹北市復興三路二段168號10樓之3（部分遠端）
**Updated:** 2026-04-22

---

# SiFive Business Model

SiFive is a **fabless IP licensing company**, not a chip manufacturer. Think of them as the "ARM of RISC-V":

|                         |                                                                       |
| ----------------------- | --------------------------------------------------------------------- |
| **What they sell**      | RISC-V processor IP (Core IP blocks)                                  |
| **How they make money** | Upfront licensing fees + royalties per chip sold                      |
| **Who buys**            | SoC companies, chip designers (automotive, data center, AI edge, IoT) |
| **What they don't do**  | Fabricate chips — customers tape out using SiFive IP                  |

**Product lines:**
- **SiFive Essential** — embedded/IoT cores
- **SiFive Performance** — application-class Linux-capable cores
- **SiFive Intelligence** — AI/ML accelerator cores (RVV-based)
- **SiFive Automotive** — safety-certified (ISO 26262) cores

**Scale:** 400+ design wins, 2B+ devices shipped using SiFive IP, 85% engineers on staff.

**Why Taiwan?** Close to TSMC, MediaTek, Realtek, and the broader SoC ecosystem that licenses their IP.


---

# Flow for chip production

```
  ┌─────────────────────────────────────────────────────────────────────┐
  │                         IP VENDORS                                  │
  │                                                                     │
  │   ┌─────────────┐   ┌─────────────┐   ┌──────────────────────┐      │
  │   │   SiFive    │   │     ARM     │   │  Synopsys / Cadence  │      │
  │   │  (CPU core) │   │  (CPU core) │   │  (USB / PCIe / DDR)  │      │
  │   └──────┬──────┘   └──────┬──────┘   └────────────┬─────────┘      │
  └──────────┼─────────────────┼───────────────────────┼────────────────┘
             │   license RTL + software stack          │
             └─────────────────┬───────────────────────┘
                               ▼
  ┌─────────────────────────────────────────────────────────────────────┐
  │                    CHIP DESIGNERS (Fabless)                         │
  │                                                                     │
  │   ┌──────────────────────────────────────────────────────────┐      │
  │   │              SoC (e.g. MediaTek / Apple / Qualcomm)      │      │
  │   │                                                          │      │
  │   │  ┌────────────┐  ┌─────────┐  ┌───────┐  ┌──────────┐    │      │
  │   │  │ RISC-V CPU │  │   USB   │  │  DDR  │  │   PCIe   │    │      │
  │   │  │ (SiFive IP)│  │  (IP)   │  │  (IP) │  │   (IP)   │    │      │
  │   │  └────────────┘  └─────────┘  └───────┘  └──────────┘    │      │
  │   │                    custom glue logic                     │      │
  │   └──────────────────────────────────────────────────────────┘      │
  └─────────────────────────────┬───────────────────────────────────────┘
                                │  tape-out (GDS-II layout files)
                                ▼
  ┌─────────────────────────────────────────────────────────────────────┐
  │                     MANUFACTURERS (Foundries)                       │
  │                                                                     │
  │         ┌──────────┐       ┌──────────┐       ┌──────────┐          │
  │         │   TSMC   │       │ Samsung  │       │  Intel   │          │
  │         │ (台積電)  │       │ Foundry  │       │ Foundry  │          │
  │         └──────────┘       └──────────┘       └──────────┘          │
  └─────────────────────────────┬───────────────────────────────────────┘
                                │  physical chips
                                ▼
                    ┌───────────────────────┐
                    │      End Products     │
                    │  Phones / Servers /   │
                    │     Cars / IoT        │
                    └───────────────────────┘
```


```
  ─────────────────────────────────────────────────────────────────────
    MONEY FLOW (reversed)
  ─────────────────────────────────────────────────────────────────────

    End customer ──▶ Chip designer ──▶ Foundry       (fab fee / wafer)
                                 └───▶ IP vendor     (license fee + royalty/chip)
```


---

# What is License RTL?

  **RTL** stands for **Register Transfer Level** — it's the source code of a chip.

  ---

## What RTL Is

  Just like software has source code (C, Python...), hardware has its own "source code"
  written in hardware description languages:

  ```
  Software world          Hardware world
  ──────────────          ──────────────
  C / C++ code      ≈     RTL (Verilog / VHDL / Chisel)
  GCC compiler      ≈     Synthesis tool (Synopsys Design Compiler)
  Binary (.exe)     ≈     GDS-II (layout file sent to TSMC)
  ```

  RTL describes **how data moves between registers every clock cycle** — hence
  "Register Transfer Level."


  ## A Simple Example

  A hardware adder in RTL (Verilog):

  ```verilog
  module adder (
      input  clk,
      input  [31:0] a, b,
      output reg [31:0] result
  );
      always @(posedge clk) begin
          result <= a + b;   // transfer: compute a+b, store in result register
      end
  endmodule
  ```

  This describes **logic behaviour**, not physical transistors. The synthesis tool then
  converts this into actual gates and wires on silicon.
  
  ---

## What "License RTL" Means

  When SiFive licenses their CPU to MediaTek, they hand over:

  ```
  SiFive delivers:
  ├── core.v          ← RTL source (Verilog) of the CPU
  ├── testbench/      ← verification suite
  ├── docs/           ← architecture spec
  └── software/       ← Linux BSP, OpenSBI, drivers  ← your job
  ```

  MediaTek then:
  1. Integrates the CPU RTL into their SoC design
  2. Runs synthesis → generates GDS-II
  3. Sends GDS-II to TSMC to fabricate

  ---

## Why It's Valuable (and Licensed, Not Sold)

  SiFive spent years and millions of dollars designing, verifying, and optimizing their
  CPU cores. The RTL **is** their product — equivalent to source code.

  They don't sell it outright. They **license** it, meaning:
  - You can use it under agreed terms
  - You can't resell it or sublicense it
  - Every chip you ship pays a royalty back to SiFive

  > Same reason you pay Microsoft per Windows license, not a one-time purchase
  > of the source code.

---

# Role Breakdown

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
