---
tags:
  - hardware
  - architecture
  - gameboy
  - emulation
created: '2026-04-27'
---
## Memory-Mapped I/O (MMIO)

### The core idea

A CPU only knows how to do one thing: read and write to **addresses**. It has no special "talk to hardware" instruction. Memory-mapped I/O is the design decision that says:

> Just put the hardware registers into the same address space as RAM. The CPU writes to an address, and instead of a memory cell changing, a hardware peripheral reacts.

The address bus doesn't care what's physically behind an address — RAM chip, ROM chip, or a sound controller. The CPU can't tell the difference. The hardware designer decides what each address connects to.

```
CPU writes to 0xFF11
       │
       ▼
  Address decoder
       │
       ├── 0x0000–0x7FFF  →  ROM chip
       ├── 0x8000–0x9FFF  →  VRAM chip
       ├── 0xC000–0xDFFF  →  WRAM chip
       └── 0xFF00–0xFF7F  →  I/O registers  ◄── ends up here
                                  │
                                  └── APU channel 1 length register
                                      (hardware reacts immediately)
```

Nothing is stored. The write is a **command to hardware**.

---

### The alternative — port-mapped I/O

The x86 architecture takes a different approach. It has dedicated `IN` and `OUT` instructions that talk to a separate **I/O address space**, completely isolated from memory:

```asm
; x86 port-mapped I/O
IN  AL, 0x60    ; read from keyboard controller port
OUT 0x43, AL    ; write to PIT timer port

; Game Boy / ARM memory-mapped I/O
LD  A, (0xFF00) ; read joypad — looks identical to reading RAM
LD  (0xFF06), A ; write timer modulo — looks identical to writing RAM
```

Port-mapped I/O requires the CPU to have extra pins and extra instructions. Memory-mapped I/O requires neither — the CPU is simpler and peripherals just sit on the same bus.

Most modern architectures (ARM, RISC-V, Game Boy SM83) use **pure MMIO**. x86 uses both.

---

### What this means for emulation

Every read and write in your emulator must be **intercepted by the MMU** before touching an array. When the CPU writes to `0xFF02`, it's not storing a byte — it's telling the serial controller to start a transfer:

```c
// mmu_write_byte — the address decoder
if (addr == 0xFF02) {
    mmu->io[0x02] = value | 0x7E;  // SC register — hardware behaviour, not plain storage
    return;
}
if (addr == 0xFF04) {
    mmu->io[0x04] = 0;             // writing ANY value to DIV resets it to 0
    timer->div_counter = 0;
    return;
}
```

The `io[]` array in your emulator is a convenient backing store, but the *real* job of `mmu_write_byte` is to trigger side effects — exactly what real hardware does when the address bus hits those addresses.

---

### MMIO vs port-mapped I/O — quick comparison

| | MMIO | Port-mapped I/O |
|---|---|---|
| Address space | Shared with RAM/ROM | Separate I/O space |
| Instructions needed | Standard load/store | Dedicated `IN`/`OUT` |
| CPU complexity | Lower | Higher (extra pins) |
| Used by | ARM, RISC-V, Game Boy | x86 (alongside MMIO) |

---

### One-line summary

> **MMIO = hardware registers disguised as memory addresses.**
> The CPU writes to an address, a peripheral reacts. No special instructions needed.
