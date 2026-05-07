---
tags:
  - linux-kernel
  - x86
  - interrupts
  - idt
  - traps
  - irq
  - boot
created: '2026-05-07'
---
# x86 IDT Table Construction

The `idt_table` is the single 256-entry Interrupt Descriptor Table that the CPU consults on every exception, fault, trap, and external interrupt. Linux builds it in **stages** during boot, drawing from several source tables. Understanding which builder fills which range is the key to seeing how *traps* and *external IRQs* coexist in the same array.

## The one table

```
arch/x86/kernel/idt.c

gate_desc idt_table[IDT_ENTRIES] __page_aligned_bss;   // 256 entries

struct desc_ptr idt_descr = {
    .size    = IDT_TABLE_SIZE - 1,
    .address = (unsigned long) idt_table,
};
```

`idt_descr` is what the `lidt` instruction loads into the CPU's `IDTR` register. After `lidt`, the CPU knows where to look on every interrupt: `idt_table[vector]`.

The table is later remapped read-only at a fixed virtual address in the *cpu entry area* (`CPU_ENTRY_AREA_RO_IDT`) so that `sidt` cannot leak the kernel base and arbitrary writes cannot corrupt it (`idt_map_in_cea()`).

## Source tables (where entries come from)

All defined in `arch/x86/kernel/idt.c`:

| Source table               | Purpose                                     | Examples                                                                                          |
| -------------------------- | ------------------------------------------- | ------------------------------------------------------------------------------------------------- |
| `early_idts[]`             | Minimum needed before TSS/IST is ready      | `INTG(X86_TRAP_DB, asm_exc_debug)`, `SYSG(X86_TRAP_BP, asm_exc_int3)`                             |
| `early_pf_idts[]` (x86_64) | Page-fault before IST is ready              | `INTG(X86_TRAP_PF, asm_exc_page_fault)`                                                           |
| `def_idts[]`               | Full set of architecture-defined exceptions | DE, NMI, OF, BR, UD, NM, **DF (ISTG)**, **DB (ISTG)**, MC, TS, NP, SS, GP, PF, MF, AC, XF, VC, CP |
| `ia32_idt[]`               | 32-bit syscall gate when ia32 emu enabled   | `SYSG(IA32_SYSCALL_VECTOR, …)`                                                                    |
| `apic_idts[]`              | APIC system vectors (IPIs, timer, error)    | `RESCHEDULE_VECTOR`, `CALL_FUNCTION_VECTOR`, `LOCAL_TIMER_VECTOR`                                 |
| `irq_entries_start` (asm)  | The 224 external device-IRQ stubs           | One stub per vector in `[FIRST_EXTERNAL_VECTOR, FIRST_SYSTEM_VECTOR)`                             |

The macros `INTG`, `SYSG`, `ISTG` differ in DPL and IST index:

- `INTG` — interrupt gate, DPL=0, no IST
- `SYSG` — system gate, DPL=3 (allows `int3` from userspace)
- `ISTG` — interrupt gate that switches to an IST stack on entry

## Builders (when entries get installed)

Boot order, simplified:

```
start_kernel()
  ├── idt_setup_early_traps()         ← installs early_idts[]   (DB, BP)
  │     load_idt(&idt_descr)          ← lidt — table goes live
  │
  ├── idt_setup_early_pf()            ← installs early_pf_idts[] (PF)
  │
  ├── trap_init()
  │     idt_setup_traps()             ← installs def_idts[]
  │                                     OVERWRITES early DB with the IST variant
  │     idt_setup_ist_traps()         ← (further IST refinements)
  │
  └── apic_intr_mode_init() / smp boot
        idt_setup_apic_and_irq_gates() ← apic_idts[] + irq_entries_start array
```

Key insight: **later builders can overwrite earlier entries**. `exc_debug` is installed twice — once early (plain `INTG`) so the kernel can take a `#DB` even before TSS is set up, then again from `def_idts[]` as `ISTG(X86_TRAP_DB, asm_exc_debug, IST_INDEX_DB)` once IST routing is available. Same vector, same C handler, different stack-switch policy.

## How traps connect to the table

Each trap has a **dedicated** asm stub. The macro chain in `arch/x86/include/asm/idtentry.h`:

```
DECLARE_IDTENTRY*       declares C function exc_FOO + asm symbol asm_exc_FOO
DEFINE_IDTENTRY*        defines the C body (lives in arch/x86/kernel/traps.c, etc.)
idtentry asm macro      emits the asm stub asm_exc_FOO that:
                          - pushes (or fakes) error code
                          - builds pt_regs
                          - swapgs / cld
                          - calls exc_FOO(regs[, error_code])
                          - returns via iretq
```

The corresponding `INTG`/`ISTG` entry in `def_idts[]` then writes `idt_table[VECTOR]` to point straight at the dedicated stub. **No shared trampoline.**

Variant flavors (selecting prologue shape):

| Macro | Used for |
|---|---|
| `DECLARE_IDTENTRY` | Vanilla, no error code |
| `DECLARE_IDTENTRY_ERRORCODE` | Hardware pushes error code (PF, GP, AC, …) |
| `DECLARE_IDTENTRY_RAW` | Hand-rolled, regs-only — handler does its own setup |
| `DECLARE_IDTENTRY_IST` / `_DEBUG` | Uses an IST stack (separate stack on entry) |
| `DECLARE_IDTENTRY_MCE` / `_DF` / `_NMI` / `_VC` | Special asm prologues for the gnarly ones |
| `DECLARE_IDTENTRY_SYSVEC` | APIC system vectors — each gets its own stub |
| `DECLARE_IDTENTRY_IRQ` | **Only used by `common_interrupt`** — funnel target for all device IRQs |

## How external IRQs connect to the table

External device interrupts share a single fan-in stub: `asm_common_interrupt`. Per-vector stubs live in a packed asm array in `arch/x86/entry/entry_64.S`:

```asm
SYM_CODE_START(irq_entries_start)
    vector=FIRST_EXTERNAL_VECTOR
    .rept NR_EXTERNAL_VECTORS
0 :
    ENDBR
    .byte 0x6a, vector              ; push imm8 (vector number)
    jmp   asm_common_interrupt      ; converge on shared tail
    .fill 0b + IDT_ALIGN - ., 1, 0xcc
    vector = vector+1
    .endr
SYM_CODE_END(irq_entries_start)
```

Each stub is `IDT_ALIGN` bytes (16 on x86_64) and does only two things: push the vector number and jump to the shared tail. `idt_setup_apic_and_irq_gates()` then computes each stub's address by simple arithmetic and installs it:

```c
for_each_clear_bit_from(i, system_vectors, FIRST_SYSTEM_VECTOR) {
    entry = irq_entries_start + IDT_ALIGN * (i - FIRST_EXTERNAL_VECTOR);
    set_intr_gate(i, entry);
}
```

The `for_each_clear_bit_from(i, system_vectors, …)` loop **skips** vectors already taken by `apic_idts[]` (timer, IPIs) — those system vectors get their own dedicated stubs, just like traps.

## Summary diagram

```
idt_table[256]   (single array, loaded by lidt)
      │
      ├── vectors 0..31 — architecture traps
      │     source:   early_idts[] / def_idts[]
      │     stub:     dedicated asm_exc_FOO per vector
      │     C entry:  exc_debug, exc_page_fault, …
      │
      ├── APIC system vectors
      │     source:   apic_idts[]
      │     stub:     dedicated asm_sysvec_FOO per vector
      │     C entry:  sysvec_apic_timer_interrupt, …
      │
      └── external IRQ vectors  (FIRST_EXTERNAL..FIRST_SYSTEM)
            source:   irq_entries_start[] asm array
            stub:     16-byte slot — push vector; jmp asm_common_interrupt
            C entry:  SHARED → common_interrupt(regs, vector)
                          ↓
                      vector_irq[vector] → irq_desc
                          ↓
                      desc->handle_irq(desc)
                          ↓
                      action->handler(irq, dev_id)
```

## Why two different layouts

Traps and IRQs need fundamentally different dispatch:

- **Traps**: the vector number tells you everything — vector 14 is *always* a page fault, with specific error-code semantics, IST policy, and a single C handler. No demultiplexing needed beyond the vector itself. So one stub per vector, each going straight to its C function.
- **External IRQs**: vectors 0x20–0xFE are dynamically allocated to device interrupts. The vector tells you *which line fired*, but the kernel still needs to walk `vector_irq[]` → `irq_desc` → flow handler → driver action. So the prologue is uniform; only the vector number varies. Sharing `asm_common_interrupt` saves ~3.5 KB of stub bloat (224 vectors × 16 bytes vs one shared epilogue).

## References

| File | What's there |
|---|---|
| `arch/x86/kernel/idt.c` | All builders + source tables |
| `arch/x86/kernel/idt.c:281` | `idt_setup_apic_and_irq_gates()` |
| `arch/x86/kernel/idt.c:63` | `early_idts[]` |
| `arch/x86/kernel/idt.c:84` | `def_idts[]` |
| `arch/x86/kernel/idt.c:133` | `apic_idts[]` |
| `arch/x86/entry/entry_64.S:498` | `irq_entries_start` asm array |
| `arch/x86/include/asm/idtentry.h` | `DECLARE_IDTENTRY*` family |
| `arch/x86/include/asm/idtentry.h:640` | `DECLARE_IDTENTRY_IRQ(X86_TRAP_OTHER, common_interrupt)` |

## Cross-links

- [[Linux Kernel Booting]] — `start_kernel` ordering
- [[Page Fault Mechanism]] — what `exc_page_fault` does after entry
