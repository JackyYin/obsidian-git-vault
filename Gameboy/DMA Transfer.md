---
tags:
  - hardware
  - architecture
  - gameboy
  - emulation
  - DMA
created: '2026-04-27'
---
## DMA — Direct Memory Access

### The problem it solves

The CPU moves data one byte at a time. To copy 160 bytes of sprite data into OAM, the CPU would need to execute 160 read + 160 write instructions — burning ~1,280 T-cycles doing nothing but shuffling bytes. During that time it could have been running game logic.

DMA delegates the copying job to **dedicated hardware** that works in parallel, freeing the CPU for other things.

---

### How DMA works in general

A DMA controller is a simple piece of hardware wired directly to the memory bus. You give it three things and it does the rest:

```
CPU tells DMA controller:
  source address  = 0xC000
  destination     = 0xFE00 (OAM)
  length          = 160 bytes

DMA controller takes over the bus and copies
independently, without the CPU fetching each byte.
```

The CPU either **pauses** while the DMA runs (cycle-stolen), or the DMA and CPU **time-share** the bus and both run concurrently.

---

### Game Boy OAM DMA specifically

The Game Boy has one DMA channel dedicated to copying sprite attributes into OAM. It is triggered by writing to `0xFF46`:

```
Write 0xC0 to 0xFF46
        │
        └─► DMA copies 0xC000–0xC09F  →  OAM (0xFE00–0xFE9F)
            160 bytes, one per T-cycle group
            takes 160 M-cycles = 640 T-cycles
```

The value you write is the **high byte** of the source address. So:

| Write value | Source range copied |
|---|---|
| `0x80` | `0x8000–0x809F` |
| `0xC0` | `0xC000–0xC09F` |
| `0xDF` | `0xDF00–0xDF9F` |

Any address in `0x00–0xDF` is valid (ROM, WRAM, external RAM). Sourcing from `0xE0–0xFF` is undefined behaviour on real hardware.

---

### What the CPU can and cannot do during OAM DMA

During the 640 T-cycles of OAM DMA:

```
CPU can access:    0xFF80–0xFFFE  (HRAM only)
CPU cannot access: everything else (reads return 0xFF, writes ignored)
```

The bus is owned by the DMA controller. The standard Game Boy pattern is to copy a small routine into HRAM and jump to it before triggering DMA, then sit in a wait loop there until the transfer finishes:

```asm
; Routine copied to HRAM before DMA starts
dma_wait:
    ld   a, 0xC0
    ldh  (0xFF46), a   ; trigger DMA
    ld   a, 40         ; wait 40 × 4 = 160 M-cycles (≥ 640 T-cycles)
.loop:
    dec  a
    jr   nz, .loop
    ret
```

If your emulator doesn't restrict bus access during DMA, it will pass most games but fail tests that read from non-HRAM addresses mid-transfer.

---

### DMA in your emulator

A minimal correct implementation:

1. **Trigger**: on write to `0xFF46`, record the source base address and set a countdown of 160 M-cycles (640 T-cycles).
2. **Each tick**: if DMA is active, copy one byte per M-cycle from source to OAM and decrement the counter.
3. **Bus lockout** *(optional for basic correctness)*: block CPU reads/writes outside HRAM while active.

```c
// On write to 0xFF46:
dma->source    = value << 8;   // e.g. 0xC0 → 0xC000
dma->remaining = 160;

// Each M-cycle tick while remaining > 0:
uint16_t src = dma->source + (160 - dma->remaining);
mmu->oam[160 - dma->remaining] = mem_read(src);
dma->remaining--;
```

---

### Timing summary

| Property | Value |
|---|---|
| Trigger register | `0xFF46` |
| Bytes transferred | 160 |
| Duration | 160 M-cycles = 640 T-cycles |
| CPU accessible during transfer | HRAM (`0xFF80–0xFFFE`) only |
| Destination | OAM `0xFE00–0xFE9F` |
| Source | `(written value) << 8`, range `0x0000–0xDF9F` |

---

### One-line summary

> **DMA = a hardware shortcut that copies a block of memory without tying up the CPU.**
> On the Game Boy, writing to `0xFF46` copies 160 bytes into OAM in 640 T-cycles while the CPU is restricted to HRAM.
