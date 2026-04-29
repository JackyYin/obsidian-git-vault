---
tags:
  - gameboy
  - emulator
  - timer
  - hardware
---
# Game Boy Timer Implementation

## Overview

The Game Boy timer consists of four memory-mapped registers and an internal 16-bit counter that drives everything. The critical insight is that **DIV and TIMA are not independent** — they are both derived from the same internal counter.

---

## The Internal 16-bit Counter (`sys_counter`)

The hardware maintains a 16-bit counter that increments every T-cycle (at 4,194,304 Hz). All timer behavior is derived from this counter.

```
sys_counter (16-bit, increments every T-cycle)

Bit:   15 14 13 12 11 10 9 8      7  6  5  4  3  2  1  0
      ──────────────────────────────────────────────────
      │        DIV (FF04)       │                      │
      └──────── bits 15..8 ─────┘                      │
                                  TIMA clock sources ──┘
```

The **DIV register** (0xFF04) is simply the **upper 8 bits** of `sys_counter`:

```
DIV = sys_counter >> 8
```

Writing to 0xFF04 resets `sys_counter` to 0, which also resets DIV.

---

## How TIMA is Clocked

TIMA does **not** have its own independent counter. Instead, it is incremented on the **falling edge** of a specific bit of `sys_counter`, selected by TAC bits 1-0.

### Why "bit 9" for 1024 T-cycle period?

Bit N of `sys_counter` is 0 for the first `1<<N` counts, then 1 for the next `1<<N` counts, and so on. The full cycle is `1<<(N+1)` T-cycles, and the **falling edge** (1→0) occurs once per full cycle.

```
Bit 9 timeline:

sys_counter:  0       512      1024     1536     2048
              │        │        │        │        │
bit 9:    ────┐        ┌────────┐        ┌────────┐
         0    0        1   1   0    0    1   1   0
                                ▲                ▲
                             falling          falling
                             (T=1024)         (T=2048)
                             TIMA++            TIMA++

  0 –  511 : bit 9 = 0
512 – 1023 : bit 9 = 1
      1024 : bit 9 falls (1→0) → TIMA++   ← falling edge
1024 – 1535: bit 9 = 0
1536 – 2047: bit 9 = 1
      2048 : bit 9 falls again  → TIMA++

Toggle period = 1<<9  =  512 T  (bit changes state every 512 T)
Falling edge  = 1<<10 = 1024 T  (falls once every 1024 T) ✓
```

The bit number N is chosen so that `1<<(N+1)` equals the desired TIMA period:

| TAC bits 1-0 | sys_counter bit N | Toggle (1<<N) | Falling edge (1<<(N+1)) | TIMA frequency |
|:---:|:---:|---:|---:|---:|
| 00 | 9 | 512 T | **1024 T** | 4,096 Hz |
| 01 | 3 | 8 T | **16 T** | 262,144 Hz |
| 10 | 5 | 32 T | **64 T** | 65,536 Hz |
| 11 | 7 | 128 T | **256 T** | 16,384 Hz |

### The AND Gate Model

The hardware uses an AND gate to combine `TAC_enable` and the selected `sys_counter` bit. A falling edge of the **AND gate output** increments TIMA:

```
sys_counter[bit] ──┐
                   AND ──── falling edge detector ──── TIMA++
TAC_enable     ────┘
```

This means:
- **Disabling the timer** (clearing TAC bit 2) when `sys_counter[bit]` is 1 **causes an extra TIMA increment** (AND output falls from 1 → 0).
- **Writing to DIV** (resetting `sys_counter` to 0) when `sys_counter[bit]` is 1 **causes an extra TIMA increment**.

---

## Falling Edge Detection (Code)

In `timer_tick`, we step 4 T-cycles at a time and check for a 1 → 0 transition:

```c
uint16_t old = timer->sys_counter;
timer->sys_counter = (uint16_t)(timer->sys_counter + step);  // step = 4

if (timer_enabled) {
    if (BITS_SET(old, bit) && BITS_NOT_SET(timer->sys_counter, bit)) {
        tima_increment(bus);   // falling edge → TIMA++
    }
}
```

```
Example: TAC mode 10 (bit 5, falling edge every 64 T)

sys_counter:   60    64    68
               │     │     │
bit 5:     ────┘     └─────   (60: bit5=1, 64: bit5=0)
               ▲
           old=60, new=64
           BITS_SET(60,5)=true, BITS_NOT_SET(64,5)=true
           → tima_increment()
```

---

## DIV Write: Falling Edge Side Effect

When software writes to 0xFF04:

```
Before write:  sys_counter = 60 (0b00111100), bit 5 = 1

timer_div_write():
  1. Check: TAC enabled AND sys_counter[bit] == 1?  → YES
  2. tima_increment()                                → TIMA++
  3. sys_counter = 0
  4. DIV = 0
```

```
sys_counter:  ...  60   [WRITE]   0   4   8   ...
bit 5:        ...   1      ↓      0   0   0   ...
                           │
                       forced fall
                          TIMA++
```

This is why `div_write.gb` and `tim01_div_trigger.gb` / `tim10_div_trigger.gb` pass — they rely on the DIV write forcing a TIMA increment.

---

## TAC Write: Falling Edge Side Effect

When software writes to 0xFF07:

```c
timer_tac_write(value):
  old_and = TAC_enable_old AND sys_counter[old_bit]
  apply new TAC
  new_and = TAC_enable_new AND sys_counter[new_bit]
  if (old_and AND NOT new_and):
      tima_increment()
```

```
Case: timer was enabled, bit is high, now disabling

sys_counter:    ...  48  ...
TAC_enable:     1  → 0
bit 5 of 48:    1
old_and:        1 (enabled AND bit high)
new_and:        0 (disabled)
→ falling edge on AND output → TIMA++
```

This is why `rapid_toggle.gb` passes.

---

## TIMA Overflow and the 4T Reload Delay

When TIMA overflows (wraps from 0xFF → 0x00), the hardware does **not** immediately reload from TMA or fire the interrupt. There is a **4 T-cycle delay**:

```
T-cycle:     N        N+4
             │         │
TIMA:    0xFF│→ 0x00   │→ TMA
             │         │
IF bit 2:    0         1 ← interrupt fires here
             │         │
             overflow  reload + interrupt
             (window: TIMA=0 for 4T)
```

The 4T window is where TIMA reads as 0x00 and IF bit 2 is not yet set. After 4T, the reload fires and the interrupt is requested simultaneously.

### During the 4T Window

| Action        | Effect                                                        |
| ------------- | ------------------------------------------------------------- |
| Write to TIMA | Cancels reload; TIMA keeps written value; interrupt NOT fired |
| Write to TMA  | New value will be used when reload fires                      |
| No write      | Reload fires at N+4: TIMA = TMA, IF bit 2 set                 |


---

## Implementation: Reload Timer

`tima_reload_timer` counts down T-cycles until the reload fires:

```c
static void tima_increment(Bus *bus) {
    mmu->io[0x05]++;
    if (mmu->io[0x05] == 0) {
        timer->tima_reload_timer = 4;  // start 4T countdown
    }
}
```

In `timer_tick`, the reload countdown is processed **before** the falling-edge check. This is the key ordering that produces the correct 4T delay:

```c
// Step 1: check pending reload  ← runs BEFORE edge check
if (timer->tima_reload_timer > 0) {
    if (timer->tima_reload_timer <= step) {
        timer->tima_reload_timer = 0;
        mmu->io[0x05] = mmu->io[0x06];   // TIMA = TMA
        mmu->io[0x0F] |= 0x04;            // request interrupt (N+4)
    } else {
        timer->tima_reload_timer -= step;
    }
}

// Step 2: advance counter and check falling edge
uint16_t old = timer->sys_counter;
timer->sys_counter += step;
if (timer_enabled && BITS_SET(old, bit) && BITS_NOT_SET(sys_counter, bit))
    tima_increment(bus);   // sets reload_timer = 4 if overflow
```

```
Step A (T=N):   reload check → nothing (timer=0)
                edge check   → TIMA overflows → reload_timer = 4

Step B (T=N+4): reload check → 4 ≤ 4 → FIRES: TIMA=TMA, IF set  ✓
                edge check   → (continues normally)
```

Because the reload check runs **before** the edge check, a reload set during step A cannot fire until step B — giving exactly 4T of delay.

---

## Cancelling a Reload (TIMA Write During Window)

```c
void timer_tima_write(Bus *bus, uint8_t value) {
    timer->tima_reload_timer = 0;   // cancel pending reload
    mmu->io[0x05] = value;          // keep written value
    // interrupt is NOT fired
}
```

```
T=N:    TIMA overflows → 0x00, reload_timer = 4

        [CPU writes 0x42 to TIMA between N and N+4]
              │
        reload_timer = 0
        TIMA = 0x42

T=N+4:  reload check → timer=0, nothing (canceled) ✓
        IF bit 2 never set
```

---

## Complete Register Summary

| Register | Address | Read | Write |
|----------|---------|------|-------|
| DIV | 0xFF04 | `sys_counter >> 8` | Resets `sys_counter` to 0; may trigger TIMA |
| TIMA | 0xFF05 | Timer counter | Sets TIMA; cancels any pending reload |
| TMA | 0xFF06 | Modulo value | Stored; used at next reload |
| TAC | 0xFF07 | Timer control | Changes enable/clock; may trigger TIMA |

### TAC Bit Layout

```
Bit:  7  6  5  4  3  2  1  0
      ─  ─  ─  ─  ─  ┬  ┴──┘
                      │   └── clock select (00/01/10/11)
                      └────── timer enable (1 = enabled)
```

---

## Known Limitations

`tima_write_reloading.gb` and `tma_write_reloading.gb` fail because they test **sub-instruction timing**: the `LDH ($05), A` instruction is 12 T-cycles, and the SM83 performs the memory write in M3 (T=8–11). If the TIMA clock fires in M2 (T=4–7), the write in M3 would cancel the reload — but our per-instruction emulator applies the CPU write at T=0 (before `timer_tick`), so the write appears to happen before the clock. Fixing this requires sub-instruction (per-M-cycle) execution granularity in `cpu_step`.
