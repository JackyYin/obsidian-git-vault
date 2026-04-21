---
tags:
  - gameboy
  - C
---

# MBC1 Implementation Guide

Everything you need to know, and every pitfall that will burn you.

---

## Table of Contents

1. [Overview](#overview)
2. [ROM Header Fields](#rom-header-fields)
3. [Internal Registers](#internal-registers)
4. [Register Writes](#register-writes)
5. [ROM Banking](#rom-banking)
6. [RAM Banking](#ram-banking)
7. [Banking Mode (the Dual-Purpose Register)](#banking-mode-the-dual-purpose-register)
8. [MBC1M — The Multicart Variant](#mbc1m--the-multicart-variant)
9. [Pitfall Reference](#pitfall-reference)
10. [Test ROM Checklist](#test-rom-checklist)

---

## Overview

MBC1 is the first and most common Memory Bank Controller on the Game Boy. It
sits between the CPU address bus and the cartridge ROM/RAM chips, and lets games
address more ROM and RAM than the 16 KB + 8 KB the CPU's fixed address map
would otherwise allow.

The CPU sees this address space:

```
0x0000–0x3FFF   ROM bank 0  (always the first 16 KB of ROM)
0x4000–0x7FFF   ROM bank N  (switchable)
0x8000–0x9FFF   VRAM        (not handled by MBC)
0xA000–0xBFFF   External RAM (switchable, optional)
```

MBC1 exposes **four write-only registers** that live in the ROM address range.
Writing to ROM does not modify ROM — instead the MBC intercepts those writes and
updates its internal state.

---

## ROM Header Fields

Before the registers, understand what you parse out of the ROM header at load time.

| Offset  | Field          | Used for                                                    |
| ------- | -------------- | ----------------------------------------------------------- |
| `0x147` | Cartridge type | `0x01` = MBC1, `0x02` = MBC1+RAM, `0x03` = MBC1+RAM+BATTERY |
| `0x148` | ROM size code  | Number of ROM banks (see table below)                       |
| `0x149` | RAM size code  | Amount of external RAM (see table below)                    |

### ROM size codes

| Code   | ROM size | Banks |
| ------ | -------- | ----- |
| `0x00` | 32 KB    | 2     |
| `0x01` | 64 KB    | 4     |
| `0x02` | 128 KB   | 8     |
| `0x03` | 256 KB   | 16    |
| `0x04` | 512 KB   | 32    |
| `0x05` | 1 MB     | 64    |
| `0x06` | 2 MB     | 128   |


> **Pitfall — `rom_nr_bits` is a code, not a count.**
> `rom[0x148]` stores the code (0–6), not the number of banks. The number of
> banks is `2 << code`. The code happens to equal `log2(num_banks) - 1`. Keep
> this straight when you build your bank mask.

### RAM size codes

| Code | RAM size | Banks |
|------|----------|-------|
| `0x00` | 0 KB | 0 |
| `0x01` | 2 KB | — (no banking) |
| `0x02` | 8 KB | 1 |
| `0x03` | 32 KB | 4 |
| `0x05` | 64 KB | 8 (not reachable on real MBC1, see pitfall) |

> **Pitfall — 2 KB RAM (code `0x01`).**
> Only one bank exists, and only 2 KB of it is physically present. Accesses
> above `0xA7FF` within the bank are open bus on real hardware (return `0xFF`).
> Most emulators just allocate 8 KB and ignore the upper 6 KB; this is usually
> fine in practice.

---

## Internal Registers

MBC1 has four registers, all write-only and zero-initialised at power-on:

| Name | Bits | Reset value |
|------|------|-------------|
| `RAMG` — RAM Gate (enable) | 4 | `0x0` (disabled) |
| `BANK1` — ROM bank low bits | 5 | `0x01` |
| `BANK2` — ROM bank high / RAM bank | 2 | `0x00` |
| `MODE` — Banking mode | 1 | `0` (ROM mode) |

---

## Register Writes

### `0x0000–0x1FFF` — RAMG (RAM Enable)

```
write any value with lower nibble == 0x0A  →  RAM enabled
write anything else                        →  RAM disabled
```

```c
cart->ram_enabled = (value & 0x0F) == 0x0A;
```

> **Pitfall — only the lower nibble matters.**
> `0x0A`, `0x1A`, `0xAA`, `0xFA` all enable RAM. `0x00` is the canonical
> disable value but any value whose low nibble is not `0xA` disables it.

> **Pitfall — reads from `0xA000–0xBFFF` when RAM is disabled return `0xFF`.**
> Do not return `0x00` or the last written value. The open-bus value on real
> hardware is `0xFF`.

---

### `0x2000–0x3FFF` — BANK1 (ROM bank, lower 5 bits)

```c
uint8_t bank = value & 0x1F;
if (bank == 0) bank = 1;
cart->rom_bank = bank;
```

> **Pitfall — writing `0x00` selects bank `0x01`, not bank `0x00`.**
> The MBC1 hardware cannot select bank 0 in the switchable slot. Writing 0
> is silently promoted to 1. This also applies to multiples of 0x20:
> writing `0x20`, `0x40`, `0x60` (whose lower 5 bits are `0x00`) all
> select bank `0x01`, `0x21`, `0x41` respectively — not the bank you asked for.
> The "missing" banks are `0x00`, `0x20`, `0x40`, `0x60`.

---

### `0x4000–0x5FFF` — BANK2 (ROM bank upper bits, or RAM bank)

```c
cart->ram_bank = value & 0x03;  // stored as 2 bits
```

This 2-bit register is dual-purpose. Its effect depends on `MODE`:

- **MODE 0** — contributes bits 5–6 of the ROM bank number in the switchable
  slot. Also affects the fixed slot in mode 1 (see below). RAM is always
  locked to bank 0.
- **MODE 1** — selects which 8 KB RAM bank is mapped at `0xA000–0xBFFF`.

The value is *always stored*, regardless of mode. Changing mode later
immediately applies the stored value.

---

### `0x6000–0x7FFF` — MODE (Banking Mode)

```c
cart->bank_mode = value & 0x01;
```

| Value | Name | Effect |
|-------|------|--------|
| `0` | ROM mode | BANK2 extends ROM; RAM locked to bank 0 |
| `1` | RAM mode | BANK2 selects RAM bank; fixed ROM slot affected |

---

## ROM Banking

### Full bank number formula

The effective ROM bank number for each slot is:

```
For 0x4000–0x7FFF (switchable):
    bank = (BANK2 << 5) | BANK1
    bank = bank & bank_mask        // mask to actual ROM size
    // BANK1 = 0 → 1 promotion was already applied on write

For 0x0000–0x3FFF (fixed):
    if MODE == 0:
        bank = 0                   // always physical bank 0
    if MODE == 1:
        bank = (BANK2 << 5) & bank_mask   // BANK1 is ignored here
```

### Bank mask

Only address bits that correspond to real ROM banks should be used:

```c
// rom_nr_bits is the value at rom[0x148] (e.g. 0x04 for 512 KB)
// num_banks = 2 << rom_nr_bits
// bank_mask = num_banks - 1
uint8_t bank_mask = (uint8_t)((2 << cart->rom_nr_bits) - 1);
```

> **Pitfall — not masking bank bits to ROM size.**
> Without the mask, a large BANK2 value on a small ROM will compute an address
> past the end of the ROM data. You will read garbage or crash.

> **Pitfall — the fixed bank (0x0000–0x3FFF) is NOT always bank 0 in MODE 1.**
> This surprises almost everyone. In MODE 1, BANK2 also shifts the fixed slot:
> a game running in RAM banking mode with BANK2 = 2 will see bank 64 in both
> `0x0000–0x3FFF` AND `0x4000–0x7FFF` (with BANK1 providing the fine offset).
> Games that use RAM banking mode therefore cannot rely on the interrupt vectors,
> header, or any fixed-slot code remaining at their normal addresses once BANK2
> is nonzero. Almost no real game relies on this because it is dangerous — but
> test ROMs will verify it.

> **Pitfall — bank numbers 0x20 / 0x40 / 0x60 cannot be mapped into the
> switchable slot.**
> Because writing `0x00` to BANK1 is promoted to `0x01`, any target bank
> whose lower 5 bits are zero is unreachable in the switchable slot. For
> example, you cannot directly address bank 32 (`0x20`); accessing it always
> gives bank 33 (`0x21`). These banks are accessible only via the fixed slot
> in MODE 1.

---

## RAM Banking

### Address calculation

```c
uint8_t num_ram_banks = cart->ram_size >> 13;   // ram_size / 8 KB
uint8_t eff_bank      = (num_ram_banks > 0)
                        ? cart->ram_bank % num_ram_banks
                        : 0;
uint32_t ram_addr = (uint32_t)(cart->bank_mode
                        ? eff_bank * 0x2000
                        : 0)
                    + (addr - 0xA000);
```

> **Pitfall — RAM bank register must wrap to available banks.**
> MBC1 has a 2-bit register that can hold values 0–3. If the cartridge only
> has 8 KB of RAM (1 bank), selecting bank 3 must alias to bank 0, not
> access an address 24 KB past the start of RAM. Without wrapping, writes
> get silently dropped and reads return heap garbage.
>
> The mooneye `ram_64kb` test specifically verifies this: it selects bank 3,
> writes a pattern, then reads it back, expecting to see the same data because
> the write and read both aliased to bank 0.

> **Pitfall — in MODE 0, RAM is always bank 0 regardless of BANK2.**
> Even if BANK2 was set to 3 before switching to MODE 0, the RAM slot is
> locked to bank 0 while in ROM banking mode. Many games never use RAM mode
> and rely on this.

> **Pitfall — writes to RAM when disabled must be silently ignored.**
> Do not update the RAM array. The write has no effect on real hardware.

---

## Banking Mode (the Dual-Purpose Register)

The single MODE bit controls four behaviours simultaneously:

```
MODE 0 (ROM mode — default)              MODE 1 (RAM mode)
──────────────────────────────           ──────────────────────────────
Fixed slot  0x0000–0x3FFF = bank 0       Fixed slot uses BANK2 << 5
Switchable  0x4000–0x7FFF = BANK2:BANK1  Switchable = BANK2:BANK1 (same)
RAM slot    0xA000–0xBFFF = bank 0       RAM slot = BANK2
BANK2 role  Upper ROM bits               RAM bank select
```

This is the single most complicated part of MBC1. A summary table:

| MODE | Fixed ROM slot    | Switchable ROM slot        | RAM slot            |     |
| ---- | ----------------- | -------------------------- | ------------------- | --- |
| 0    | bank 0 always     | `(BANK2<<5)\|BANK1` masked | bank 0 always       |     |
| 1    | `BANK2<<5` masked | `(BANK2<<5)\|BANK1` masked | `BANK2 % num_banks` |     |



> **Pitfall — MODE 1 changes the fixed ROM slot without any obvious write.**
> When a game writes a new value to BANK2 while in MODE 1, the fixed slot
> immediately changes. Code executing in the fixed slot can only safely do
> this if it copies itself to WRAM or HRAM first, or never uses BANK2 > 0.

> **Pitfall — switching MODE retroactively applies the stored BANK2.**
> BANK2 is written and stored before MODE changes. The moment you write
> `0x01` to `0x6000`, the current BANK2 value starts selecting the RAM bank
> and remapping the fixed ROM slot. Order of writes matters.

---

## MBC1M — The Multicart Variant

Some cartridges pack 4 separate games into a single MBC1 chip (e.g. the
Nintendo Campus Challenge cart). These are called **MBC1M** (multi-game).

Key differences from standard MBC1:

| Property             | Standard MBC1           | MBC1M                   |
| -------------------- | ----------------------- | ----------------------- |
| ROM organisation     | 1 game, full bank space | 4 games × 16 banks each |
| BANK1 bits used      | 5 (bits 0–4)            | 4 (bits 0–3)            |
| BANK2 shift          | `5`                     | `4`                     |
| Fixed slot in MODE 1 | `BANK2 << 5`            | `BANK2 << 4`            |

```c
// Switchable slot, MBC1M:
bank = ((cart->ram_bank << 4) | (cart->rom_bank & 0x0F)) & bank_mask;

// Fixed slot, MBC1M, MODE 1:
bank = (cart->ram_bank << 4) & bank_mask;
```

### Detection

There is no header flag for MBC1M. Detect it by checking whether a valid
Nintendo logo appears at both `0x0104` (bank 0) and `0x40104` (bank 2 = second
game's header):

```c
static bool check_is_mbc1m(Cartridge *cart) {
    // Nintendo logo must appear in bank 0
    if (!logo_matches(cart->rom, 0x0104)) return false;
    // And again at the start of the second game (ROM offset 0x40104)
    if (cart->rom_size <= 0x40104)        return false;
    return logo_matches(cart->rom, 0x40104);
}
```

> **Pitfall — applying the 5-bit BANK1 mask to an MBC1M.**
> On a standard MBC1, the BANK1 register selects among 32 banks (0–31).
> On MBC1M each game only has 16 banks, so only 4 bits of BANK1 are
> meaningful. Using 5 bits will corrupt addresses and bleed into the wrong
> game's data.

> **Pitfall — detecting MBC1M by ROM size alone.**
> MBC1M carts always use a 1 MB ROM image, but not every 1 MB MBC1 cart is
> an MBC1M. Always check for the duplicate Nintendo logo.

---

## Pitfall Reference

A consolidated list, roughly in order of how often they bite people:

### P1 — Bank 0 write promotes to bank 1
Writing `0x00` (or any value whose low 5 bits are zero) to `0x2000–0x3FFF`
selects bank 1, not bank 0. The fixed slot (`0x0000–0x3FFF`) always shows
bank 0; the switchable slot simply cannot show bank 0.

### P2 — Disabled RAM reads return `0xFF`, not `0x00`
The open-bus value is `0xFF`. Games that check `0xFF` to detect absent save
data rely on this.

### P3 — RAM bank register must wrap to available banks
With 8 KB RAM (1 bank), `ram_bank % 1 = 0` always. With 32 KB (4 banks),
`ram_bank % 4 = ram_bank` (unchanged, already in range). Without this, writes
fail a bounds check and reads return garbage.

### P4 — Fixed ROM slot changes in MODE 1
`0x0000–0x3FFF` is not always bank 0. In MODE 1 it follows `BANK2 << 5`.
This is invisible in most games but tested explicitly by mooneye.

### P5 — ROM bank mask must match actual ROM size
Build the mask from `rom[0x148]`: `mask = (2 << code) - 1`. Without it,
large BANK2 values on small ROMs compute out-of-bounds addresses.

### P6 — BANK2 and MODE interact retroactively
Both registers are stored independently. Changing MODE immediately reinterprets
the stored BANK2. Writing BANK2 in MODE 1 immediately remaps both RAM and the
fixed ROM slot.

### P7 — Only the lower nibble of the RAM enable value matters
`0x0A` enables, everything else disables. `0xAA` also enables. `0x0B` disables.

### P8 — RAM mode locks ROM to the lower 2 Mbit
In MODE 1 (RAM mode), BANK1 can only address 32 ROM banks (0–31 after the
`0 → 1` promotion). BANK2 selects the RAM bank and simultaneously remaps
the fixed ROM slot. Games with > 512 KB ROM that also use RAM must manage
this carefully.

### P9 — MBC1M uses a 4-bit BANK1, not 5-bit
Applying the standard 5-bit mask to an MBC1M produces wrong bank numbers for
every bank ≥ 16 within a game.

### P10 — Writes to disabled RAM are silently discarded
No side effect, no update to the backing array. The RAM retains its previous
contents.

### P11 — `0x20` / `0x40` / `0x60` in the switchable slot always give `+1`
Bank `0x20` always resolves to `0x21`, `0x40` to `0x41`, `0x60` to `0x61`.
These banks are not missing from the ROM image — they exist and are accessible
via the fixed slot in MODE 1 — they just cannot be mapped into the switchable
slot.

### P12 — Allocate RAM from the header, not from assumptions
Code `0x00` → 0 bytes (no RAM), code `0x02` → 8 KB, `0x03` → 32 KB.
Allocating a flat 32 KB regardless of the header will appear to work but breaks
bank-wrapping tests and wastes memory on carts with no RAM.

### P13 — Battery-backed RAM must be saved and restored
Cart type `0x03` = MBC1+RAM+BATTERY. The RAM contents must be written to disk
when the emulator exits (or the cart is unloaded) and reloaded when the same
ROM is started again. This is the save file — losing it is equivalent to
pulling the battery.

---

## Test ROM Checklist

Run these mooneye tests in order. Each one targets a specific register or
interaction.

| Test                     | What it checks                             |
| ------------------------ | ------------------------------------------ |
| `mbc1/bits_ramg`         | Lower nibble of RAM enable register        |
| `mbc1/bits_bank1`        | BANK1 write behaviour and 0→1 promotion    |
| `mbc1/bits_bank2`        | BANK2 write behaviour                      |
| `mbc1/bits_mode`         | MODE register write behaviour              |
| `mbc1/ram_64kb`          | RAM bank aliasing with 8 KB (1-bank) cart  |
| `mbc1/ram_256kb`         | RAM banking with 32 KB (4-bank) cart       |
| `mbc1/rom_512kb`         | ROM banking, 32 banks, no BANK2 needed     |
| `mbc1/rom_1Mb`           | ROM banking with BANK2 used for upper bits |
| `mbc1/rom_2Mb`           | Same, larger                               |
| `mbc1/rom_4Mb`           | Fixed slot remapping in MODE 1             |
| `mbc1/rom_8Mb`           | Full 64-bank ROM, all pitfalls active      |
| `mbc1/multicart_rom_8Mb` | MBC1M detection and 4-bit BANK1            |

Pass all of these and your MBC1 is solid.