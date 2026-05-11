---
tags:
  - linux-kernel
  - synchronization
  - interrupts
  - preemption
  - locking
  - softirq
  - hardirq
created: '2026-05-10'
---
# Kernel Context Preemption Matrix

The kernel has five execution contexts ordered by priority. A higher-priority context can preempt a lower-priority one on the same CPU; same-or-lower cannot. That single rule generates the whole locking discipline (when to use `spin_lock` vs `_irqsave` vs `_bh`) and the answer to "what's safe in an ISR."

```
NMI  >  Hardirq  >  Softirq  >  Process (preempt-disabled)  >  Process (preempt-enabled)
```

## What each context disables

The matrix is a direct consequence of what each context disables **on its own CPU** when it starts running.

| Context                        | Hardirqs on this CPU | Softirqs on this CPU              | Preemption (scheduler) |
| ------------------------------ | -------------------- | --------------------------------- | ---------------------- |
| **NMI**                        | Disabled             | Disabled                          | Disabled               |
| **Hardirq**                    | Disabled (IF=0)      | Disabled                          | Disabled               |
| **Softirq**                    | Enabled              | Disabled (in `__do_softirq` loop) | Disabled               |
| **Process — preempt disabled** | Enabled              | Enabled                           | Disabled               |
| **Process — preempt enabled**  | Enabled              | Enabled                           | Enabled                |

Notes:

- A hardirq enters through an *interrupt gate*; the CPU clears `RFLAGS.IF` automatically. Hardirqs stay disabled until `iretq`.
- Softirq processing happens in `__do_softirq` (called from `irq_exit` after a hardirq, or from `ksoftirqd`); during the loop, hardirqs are re-enabled but preemption is disabled.
- "Process — preempt disabled" covers any code with a non-zero preempt count: inside a `spin_lock`, `local_bh_disable()`, `preempt_disable()`, `rcu_read_lock()` (in non-preemptible RCU), etc.

## Preemption matrix (same CPU)

Rows = currently running. Columns = candidate preemptor. ✓ = can interrupt; ✗ = cannot.

| Running ↓  /  Preemptor → | NMI | Hardirq | Softirq | Other task (scheduler) |
|---|:---:|:---:|:---:|:---:|
| **NMI**                            | ✗ ¹ | ✗ | ✗ | ✗ |
| **Hardirq**                        | ✓   | ✗ | ✗ ² | ✗ |
| **Softirq**                        | ✓   | ✓ | ✗ ³ | ✗ |
| **Process — preempt-disabled**     | ✓   | ✓ | ✓ ⁴ | ✗ |
| **Process — preempt-enabled**      | ✓   | ✓ | ✓ ⁴ | ✓ |

¹ Nested NMIs are theoretically possible on x86 but the kernel's NMI entry path serializes them.
² Softirqs run only at `irq_exit`, *after* the hardirq has already returned — never *during*.
³ A second softirq is not entered on the same CPU while one is running; the same softirq cannot recurse on the same CPU.
⁴ Softirqs may run when the hardirq that fired during this code returns via `irq_exit` (assuming `local_bh_disable` is not in effect).

## Cross-CPU is independent

Everything above is **per-CPU**. Across CPUs, contexts run independently:

- CPU 0 in hardirq, CPU 1 takes a hardirq — both run.
- CPU 0 in softirq, CPU 1 in the same softirq number — allowed; softirqs are not serialized across CPUs.
- A spinlock taken on CPU 0 just makes CPU 1 spin briefly when contended; no deadlock.

That's why `spin_lock` (without `_irqsave` / `_bh`) is sufficient for **same-context cross-CPU** sharing. The IRQ-disabling variants are only needed when the contender is a *higher-priority same-CPU* context.

## What each context can do

| | Sleep | `mutex_lock` | `kmalloc(GFP_KERNEL)` | `kmalloc(GFP_ATOMIC)` | `spin_lock` | `printk` |
|---|:---:|:---:|:---:|:---:|:---:|:---:|
| NMI                                | ✗ | ✗ | ✗ | ✗ | ✗ ⁵ | ✓ ⁶ |
| Hardirq                            | ✗ | ✗ | ✗ | ✓ | ✓ | ✓ |
| Softirq                            | ✗ | ✗ | ✗ | ✓ | ✓ | ✓ |
| Process — preempt-disabled         | ✗ | ✗ ⁷ | ✗ | ✓ | ✓ | ✓ |
| Process — preempt-enabled          | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |

⁵ NMI handlers must use `arch_spin_lock` or NMI-safe primitives.
⁶ printk has an NMI-safe path — lockless ring buffer; console emission is deferred.
⁷ `mutex_lock` may sleep → forbidden whenever `preempt_disable`, `local_bh_disable`, or any IRQ disable is in effect.

## Locking implications (derived directly)

Pick the lock variant by looking up "who is the **highest-priority** context that may take this lock":

| Highest-priority taker is in… | All other takers must… |
|---|---|
| Hardirq                | Use `spin_lock_irqsave` if not already in hardirq |
| Softirq                | Use `spin_lock_bh` if not in softirq/hardirq |
| Process only           | Use `spin_lock` (or `mutex` if any taker may sleep) |

The reason is the deadlock risk: same-CPU preemption by a higher-priority context that needs the same lock. `_irqsave` defers hardirqs while you hold the lock; `_bh` defers softirqs. Inside hardirq context itself, plain `spin_lock` is correct because hardirqs are *already* disabled.

## Concrete failure modes

The rules above are derived from real deadlock and panic shapes. Knowing what the failure looks like makes the rule stick.

### Same-CPU deadlock: process holds `spin_lock`, hardirq wants it

```
CPU 0:
  Process P: spin_lock(&L);     ← takes L, preempt off
             /* in critical section */
             ↑
             ── hardirq fires on CPU 0 ──
             ISR: spin_lock(&L);    ← L is held → spin
                                    ← P can't run to release
                                    ← (ISR is on top of P's stack)
                                    ← CPU is wedged
```

Note the asymmetry: **the higher-priority task is the one that gets stuck**, even though it has priority. Priority gets you *scheduled*; it does not get you *the lock*. The lock is just a memory cell whose value says "held."

Outcome: the watchdog eventually fires an NMI and panics the box — `soft lockup` (default 20s) or `hard lockup` (default 10s).

**Cross-CPU is never this kind of deadlock.** The holder is still running on another CPU and will release. The spinner just waits a bounded time. The deadlock above is strictly a *same-CPU* hazard.

### What `spin_lock_irqsave` actually does

```
1. pushfq → save RFLAGS into flags     ← preserves caller's IF state
2. cli                                 ← clear IF on this CPU
3. preempt_disable()
4. atomic acquire of the lock
```

Step 2 is the load-bearing piece. With `IF=0`, the hardirq from the scenario above **cannot be delivered** to this CPU while the lock is held — it stays pending in the LAPIC's IRR and is dispatched the instant `local_irq_restore(flags)` re-enables `IF` at unlock. No deadlock, no lost interrupt.

`spin_lock_bh` is the softirq analogue: it bumps the SOFTIRQ section of `preempt_count`, which makes `local_softirq_pending()` checks at `irq_exit` skip softirq processing on this CPU until the matching `spin_unlock_bh`.


### Mutex with interrupts: the interruption itself is fine

A common confusion: it is **totally OK** to be interrupted while holding a mutex.

```
Process A: mutex_lock(&m);
           op1;              ← hardirq fires here — no problem
           op2;                if the ISR does NOT touch m's data
           mutex_unlock(&m);
```

Mutexes don't disable interrupts or preemption. The ISR runs, returns via `iretq`, execution resumes at `op2` with the mutex still held. Softirqs may run on the way out; the scheduler may even preempt and schedule you out — mutex supports this, and priority inheritance keeps you making progress. The critical section eventually resumes.

The bug only appears when the interrupting context *also touches the same data* — and then the bug is the **wrong primitive choice**, not the interruption.


---

## PREEMPT_RT changes the matrix

Under `CONFIG_PREEMPT_RT`:

- Most **hardirqs** run in **kthreads** (process context). Only `IRQF_NO_THREAD` handlers stay in hardirq context.
- Most **softirqs** are routed to per-CPU **kthreads** (`ksoftirqd`).
- `spin_lock` becomes a **sleeping mutex** (`rt_mutex`). Only `raw_spin_lock` retains busy-wait + hardirq-disable semantics.
- The strict no-sleep rules for hardirq/softirq above are relaxed because these contexts now run in process context.

For portable driver code: keep using `spin_lock`/`mutex_lock` — RT transforms them appropriately. Reach for `raw_spin_lock` only when you genuinely need to spin with hardirqs disabled even on RT.

## The single rule that generates everything

```
A higher-priority context can preempt a lower-priority one on the same CPU.
Same-or-lower priority cannot.
```

From this:

- **Why same-line ISR re-entrance is impossible**: hardirq → hardirq is "same priority," not allowed.
- **Why softirqs run at `irq_exit`**: hardirq is higher priority; softirq waits until hardirq returns.
- **Why `spin_lock_irqsave` is needed in process context that shares with an ISR**: process is lower priority than hardirq; without `_irqsave`, the ISR can preempt the lock holder and deadlock waiting for the same lock.
- **Why softirq-only sharing uses `spin_lock_bh`** in non-softirq context: process is lower priority than softirq; `_bh` defers softirqs while you hold the lock.

## References

| File | What's there |
|---|---|
| `kernel/softirq.c:441` | `do_softirq` — entry from `__do_softirq` loop |
| `kernel/softirq.c:510` | `__do_softirq` — softirq processing loop, sets in_softirq, hardirqs enabled |
| `kernel/softirq.c:654` | `irq_exit` — drains pending softirqs after hardirq |
| `arch/x86/include/asm/idtentry.h` | hardirq entry path — IF cleared by interrupt gate |
| `include/linux/preempt.h` | `preempt_count` layers (HARDIRQ / SOFTIRQ / NMI / preempt) |
| `include/linux/bottom_half.h` | `local_bh_disable` / `local_bh_enable` |
| `Documentation/locking/` | full locking documentation |

## Cross-links

- [[spin_lock Implementation Walkthrough]] — what `spin_lock` actually does and why
- [[Linux Kernel Locking]] — broader locking primitives map
- [[Interrupt Code Path - Hardware to Driver Handler]] — how hardirq context is entered
- [[IRQ Handler function prototype]] — the `irq_handler_t` you write
