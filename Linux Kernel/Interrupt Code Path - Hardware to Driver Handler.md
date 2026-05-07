---
tags:
  - linux-kernel
  - x86
  - interrupts
  - irq
  - idt
  - drivers
  - apic
created: '2026-05-07'
---
# Interrupt Code Path: Hardware to Driver Handler

End-to-end trace of what happens when a device asserts an interrupt line, from the moment the silicon raises a wire to the moment your `irq_handler_t` callback runs. Focus is x86_64 with APIC; the cross-arch convergence point is `handle_irq_event`.

## High-level pipeline

```
Device pin / MSI write
    ↓
IO-APIC / PCI MSI fabric
    ↓
LAPIC of target CPU
    ↓
CPU microcode: vector → idt_table[vector] → gate's RIP
    ↓
Per-vector asm stub in irq_entries_start
    ↓
asm_common_interrupt   (shared asm prologue)
    ↓
common_interrupt(regs, vector)         ← C entry
    ↓
handle_irq(desc, regs)
    ↓
desc->handle_irq(desc)                  ← per-irqchip flow handler
    ↓
handle_irq_event_percpu
    ↓
__handle_irq_event_percpu
    ↓
action->handler(irq, action->dev_id)    ← YOUR irq_handler_t
```

## Stage 1 — Hardware delivery

A modern PCIe device usually uses **MSI/MSI-X**, which means it doesn't toggle a wire — it issues a memory write to a magic address (`0xFEExxxxx` on x86) carrying the target APIC ID and vector. The IO-APIC / interrupt remapper consumes that, decides which CPU should take it, and sends an *Interrupt Message* on the system bus. The destination LAPIC receives it, marks the vector pending in its IRR, and (when allowed by IF, TPR, and current ISR priority) raises `INTR` to the core.

Legacy devices on a shared line still go through the IO-APIC: pin → IO-APIC redirection table → LAPIC of the target CPU → same delivery mechanism.

## Stage 2 — CPU microcode dispatch

The CPU finishes its current instruction, then microcode runs roughly:

1. Check `RFLAGS.IF`. If 0, defer.
2. Read `IDTR` → walk to `idt_table[vector]` → load the gate descriptor.
3. Push `SS, RSP, RFLAGS, CS, RIP` (and error code if applicable) onto the kernel stack. If the gate has a non-zero IST index, switch to that stack via TSS first; otherwise, on a privilege change, switch to the kernel stack from `TSS.RSP0`.
4. Set `RIP = gate.offset`, `CS = gate.selector`.
5. Begin executing at the per-vector stub.

There is no software `call`. The transfer is hardware-issued; the only matching return is `iretq`.

## Stage 3 — Per-vector asm stub

Inside `arch/x86/entry/entry_64.S`:

```asm
SYM_CODE_START(irq_entries_start)
    vector=FIRST_EXTERNAL_VECTOR
    .rept NR_EXTERNAL_VECTORS
0 :
    ENDBR                       ; CET landing pad
    .byte 0x6a, vector          ; push imm8 — vector number
    jmp   asm_common_interrupt  ; fan-in
    .fill 0b + IDT_ALIGN - ., 1, 0xcc
    vector = vector+1
    .endr
SYM_CODE_END(irq_entries_start)
```

Each stub is exactly `IDT_ALIGN` (16) bytes so `set_intr_gate(i, irq_entries_start + IDT_ALIGN*i)` works. The stub's only job is to push the vector number — without that, `asm_common_interrupt` would have no way to know which line fired.

## Stage 4 — Shared asm prologue

`asm_common_interrupt` (generated from the `DECLARE_IDTENTRY_IRQ` macro chain) does the heavy lifting:

- `cld` (clear direction flag, ABI requirement)
- `swapgs` if entering from user mode
- Build `pt_regs` on the kernel stack
- Switch to the per-CPU IRQ stack (so deeply nested handlers don't blow the thread stack)
- RCU / context-tracking entry hooks (`irqentry_enter`)
- Then: `call common_interrupt`

It bridges hardware ABI to the C ABI.

## Stage 5 — C entry: `common_interrupt`

`arch/x86/kernel/irq.c:247`:

```c
DEFINE_IDTENTRY_IRQ(common_interrupt)
{
    struct pt_regs *old_regs = set_irq_regs(regs);
    struct irq_desc *desc;

    RCU_LOCKDEP_WARN(!rcu_is_watching(), "IRQ failed to wake up RCU");

    desc = __this_cpu_read(vector_irq[vector]);
    if (likely(!IS_ERR_OR_NULL(desc))) {
        handle_irq(desc, regs);
    } else {
        apic_eoi();
        ...
    }

    set_irq_regs(old_regs);
}
```

Two responsibilities:

1. **Vector → `irq_desc` lookup**. `vector_irq[]` is per-CPU; each CPU may map the same vector to a different IRQ (e.g., MSI vectors are CPU-affine).
2. Call `handle_irq()` if the slot is populated; otherwise EOI a stray.

## Stage 6 — Generic-layer entry

`arch/x86/kernel/irq.c:234`:

```c
static __always_inline void handle_irq(struct irq_desc *desc, struct pt_regs *regs)
{
    if (IS_ENABLED(CONFIG_X86_64))
        generic_handle_irq_desc(desc);
    else
        __handle_irq(desc, regs);
}
```

`generic_handle_irq_desc` (`include/linux/irqdesc.h:159`):

```c
static inline void generic_handle_irq_desc(struct irq_desc *desc)
{
    desc->handle_irq(desc);
}
```

This is the **arch-neutral** join point. From here on, the path is the same on arm64, RISC-V, etc.

## Stage 7 — Flow handler

`desc->handle_irq` is one of:

| Handler | When |
|---|---|
| `handle_edge_irq` | Edge-triggered line (most MSI) |
| `handle_level_irq` | Level-triggered (legacy shared lines) |
| `handle_fasteoi_irq` | Modern IO-APIC / GIC where EOI is fast and unconditional |
| `handle_simple_irq` | Synthetic / chained |
| `handle_percpu_devid_irq` | Per-CPU devices (timers, IPIs) |

These live in `kernel/irq/chip.c`. They:

- Acknowledge the irqchip (e.g., write to LAPIC EOI, or mask + ack on IO-APIC)
- Mask the line if needed (level-triggered)
- Call `handle_irq_event(desc)`
- Unmask after action runs

## Stage 8 — Action dispatch

`kernel/irq/handle.c`:

```c
// handle.c:189
irqreturn_t handle_irq_event_percpu(struct irq_desc *desc)
{
    irqreturn_t retval = __handle_irq_event_percpu(desc);
    add_interrupt_randomness(desc->irq_data.irq);
    return retval;
}

// handle.c:139
irqreturn_t __handle_irq_event_percpu(struct irq_desc *desc)
{
    irqreturn_t retval = IRQ_NONE;
    unsigned int irq = desc->irq_data.irq;
    struct irqaction *action;

    for_each_action_of_desc(desc, action) {
        ...
        trace_irq_handler_entry(irq, action);
        res = action->handler(irq, action->dev_id);    // ← handle.c:158
        trace_irq_handler_exit(irq, action, res);
        ...
        switch (res) {
        case IRQ_WAKE_THREAD:
            __irq_wake_thread(desc, action);
            break;
        case IRQ_HANDLED:
            ...
        }
    }
    return retval;
}
```

This walks the **`desc->action` linked list**. Every driver that called `request_irq()` on this line has an `irqaction` linked here. They run in registration order.

## Stage 9 — Your handler runs

`action->handler(irq, action->dev_id)` is exactly the function pointer you passed to `request_irq()`:

```c
static irqreturn_t my_driver_isr(int irq, void *dev_id)
{
    struct my_dev *dev = dev_id;
    /* read status, ack hardware, schedule bottom half */
    return IRQ_HANDLED;
}

ret = request_irq(irq, my_driver_isr, IRQF_SHARED, "my_driver", dev);
```

Return values that matter:

- `IRQ_HANDLED` — I serviced this device.
- `IRQ_NONE` — Not mine (shared-line case).
- `IRQ_WAKE_THREAD` — Hardirq part done; please run my `thread_fn` in process context.

## Threaded handlers

If you registered with `request_threaded_irq(irq, hard, thread, …)`, the hardirq path wakes a per-IRQ kthread. Loop body in `kernel/irq/manage.c`:

```c
static int irq_thread(void *data)
{
    ...
    while (!irq_wait_for_interrupt(action)) {
        irq_thread_fn(desc, action);
    }
}
```

`irq_thread_fn` calls your `action->thread_fn(irq, dev_id)`. This runs with interrupts enabled, in process context, and can sleep.

## Putting it all together

```
[ Hardware ]
    MSI write / pin assert
        ↓
    IO-APIC / MSI routing fabric
        ↓  (vector + dest CPU)
    target LAPIC raises INTR

──────────────────────────────────────────────
[ CPU microcode ]
    IDTR → idt_table[vector]
    push SS:RSP:RFLAGS:CS:RIP
    jump to gate.offset

──────────────────────────────────────────────
[ Kernel asm ]
    irq_entries_start[vector]:
        push $vector
        jmp  asm_common_interrupt
    build pt_regs, swapgs, switch to IRQ stack

──────────────────────────────────────────────
[ Kernel C ]
    common_interrupt(regs, vector)
        desc = vector_irq[vector]
        ↓
    handle_irq(desc, regs)
        ↓
    generic_handle_irq_desc(desc)
        → desc->handle_irq(desc)
        ↓
    handle_edge_irq / handle_level_irq / …
        ack, mask, then handle_irq_event
        ↓
    handle_irq_event_percpu
        → __handle_irq_event_percpu
        ↓
    for_each_action_of_desc(desc, action):
        res = action->handler(irq, action->dev_id);

──────────────────────────────────────────────
[ Your driver ]
    irqreturn_t my_isr(int irq, void *dev_id)
    { ... return IRQ_HANDLED; }
```

## Hooking your driver in

```c
ret = request_irq(irq, my_isr, IRQF_SHARED, "my_dev", dev);
```

Internally:

```
request_irq()
  → request_threaded_irq(irq, my_isr, NULL, flags, name, dev)
    → __setup_irq()
        - kzalloc irqaction
        - action->handler = my_isr
        - action->dev_id  = dev
        - link onto desc->action list
        - if first action: enable line via irqchip
```

After this returns, every interrupt on `irq` flows through the chain above and ends in `my_isr`.

## Summary

- Hardware delivers the interrupt; the CPU is the one that "calls" the asm stub via IDT lookup, not software.
- The asm path is identical for every external IRQ — `irq_entries_start` is a fan-out, `asm_common_interrupt` is the fan-in.
- `common_interrupt` is the only x86-specific C function on the path; from `handle_irq_event` onward, the code is shared with arm64, RISC-V, etc.
- Your `irq_handler_t` runs at `kernel/irq/handle.c:158`, deep inside `__handle_irq_event_percpu`, with hardirqs disabled and on the per-CPU IRQ stack.

## References

| File | Role |
|---|---|
| `arch/x86/entry/entry_64.S:498` | `irq_entries_start` asm array |
| `arch/x86/include/asm/idtentry.h:505` | `jmp asm_common_interrupt` |
| `arch/x86/include/asm/idtentry.h:640` | `DECLARE_IDTENTRY_IRQ(common_interrupt)` |
| `arch/x86/kernel/irq.c:247` | `DEFINE_IDTENTRY_IRQ(common_interrupt)` |
| `arch/x86/kernel/irq.c:234` | `handle_irq()` |
| `include/linux/irqdesc.h:159` | `generic_handle_irq_desc()` |
| `kernel/irq/chip.c` | flow handlers (`handle_edge_irq`, `handle_level_irq`, …) |
| `kernel/irq/handle.c:139` | `__handle_irq_event_percpu` |
| `kernel/irq/handle.c:158` | call site of `action->handler` |
| `kernel/irq/manage.c` | `request_irq`, `__setup_irq`, `irq_thread` |
| `kernel/irq/irqdesc.c:379` | `irq_to_desc()` |

## Cross-links

- [[x86 IDT Table Construction]] — how `idt_table` is built and connected
- [[IRQ Handler function prototype]] — the `irq_handler_t` signature
- [[Linux Driver Model]] — where `request_irq` typically lives in a driver lifecycle
