---
tags:
  - gameboy
  - hardware
  - protocol
  - SPI
  - serial
created: '2026-04-21'
---
## SPI — Serial Peripheral Interface

The Game Boy link cable protocol is essentially **SPI**, and the mapping is almost one-to-one.

### Side-by-side comparison

| Property | Game Boy Serial | SPI |
|---|---|---|
| Clock | Synchronous, shared clock line | Synchronous, shared SCLK |
| Direction | Full-duplex, both sides shift simultaneously | Full-duplex, MOSI + MISO simultaneously |
| Role | Internal clock = master, external = slave | Dedicated master drives SCLK |
| Frame size | Fixed 8 bits | Configurable, typically 8 bits |
| Bit order | MSB first | Configurable, typically MSB first |
| Chip select | None (software-initiated via SC bit 7) | Hardware CS/SS pin |
| Speed | Fixed 8,192 Hz (DMG) | Configurable, MHz range |

### Why the mechanism is identical

Both protocols work by connecting two **shift registers** back-to-back in a loop. Every clock pulse, one bit falls off the end of register A into register B, and vice versa. After 8 clocks, the two devices have silently swapped their bytes:

```
  GAME BOY 1 (master)          GAME BOY 2 (slave)
  ┌─────────────────┐          ┌─────────────────┐
  │  SB: 0100 0010  │──SOUT──► │  SB: 1010 1111  │
  │                 │◄──SIN─── │                 │
  └────────┬────────┘          └────────┬────────┘
           │                            │
           └──────────SCK───────────────┘
             (master generates clock)

  After 8 clocks:
  Master SB = 0b10101111   (received slave's byte)
  Slave  SB = 0b01000010   (received master's byte)
```

This is exactly how SPI works on any STM32, AVR, or RP2040. You could hook a Game Boy link cable up to a microcontroller's SPI peripheral with three wires (MOSI, MISO, SCLK) and talk to it directly.

### The one real difference

SPI has a **chip select (CS)** line — the master pulls it low to address a specific slave before clocking. The Game Boy has no CS hardware; instead the software sets SC bit 7 to signal intent. On a real link cable this works because there are only ever two devices, so there is nothing to address.

### What it is NOT

| Protocol | Why it doesn't match |
|---|---|
| **UART** | Asynchronous — no shared clock, uses start/stop bits and baud rate |
| **I²C** | Half-duplex, open-drain bus, uses 7-bit device addresses, ACK bits |
| **CAN** | Multi-master bus with arbitration and error frames |

### Practical note

If you were building a Game Boy printer or accessory clone on a modern MCU, you would configure its SPI peripheral in **slave mode** (external clock), 8-bit frames, MSB first, and it would receive Game Boy transfers natively with no bit-banging required.

### Related registers

| Register | Address | Purpose |
|---|---|---|
| SB | `0xFF01` | Serial transfer data (byte to send / byte received) |
| SC | `0xFF02` | Serial control — bit 7: transfer start, bit 0: clock select (1 = internal/master) |

### Timing (DMG)

- Internal clock: **8,192 Hz** (one bit per 512 T-cycles)
- Full byte transfer: **4,096 T-cycles** (8 bits × 512 T-cycles)
- Transfer triggers serial interrupt (IF bit 3) on completion
