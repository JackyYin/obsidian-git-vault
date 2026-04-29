---
tags:
  - linux-kernel
  - locking
  - concurrency
created: '2026-04-29'
---
# Linux Kernel Locking

## Lock Taxonomy

```
Linux Kernel Locks
│
├── SLEEPING LOCKS          (only in preemptible task context)
│   ├── mutex               — exclusive, owner must release
│   ├── rt_mutex            — mutex + priority inheritance (PI)
│   ├── semaphore           — counting, no owner (→ priority inversion risk)
│   ├── rw_semaphore        — multiple readers OR single writer
│   ├── ww_mutex            — wound/wait deadlock-avoidance mutex
│   └── percpu_rw_semaphore — scalable rw_semaphore for per-CPU fast path
│
├── CPU LOCAL LOCKS         (per-CPU concurrency only, not inter-CPU)
│   └── local_lock          — named scope for preempt/irq disable
│
└── SPINNING LOCKS          (busy-wait; implicitly disable preemption)
    ├── raw_spinlock_t      — always spinning, even on PREEMPT_RT
    └── bit spinlocks       — always spinning (too small for RT-mutex)
```

> On non-PREEMPT_RT kernels, `spinlock_t` and `rwlock_t` are also spinning locks.

---

## PREEMPT_RT: How Lock Categories Shift

```
Lock              non-PREEMPT_RT kernel        PREEMPT_RT kernel
──────────────────────────────────────────────────────────────────
raw_spinlock_t    SPINNING ──────────────────► SPINNING  (unchanged)
bit spinlock      SPINNING ──────────────────► SPINNING  (unchanged)

spinlock_t        SPINNING ──────────────────► SLEEPING  (via rt_mutex)
rwlock_t          SPINNING ──────────────────► SLEEPING  (via rt_mutex)
local_lock        CPU LOCAL ──────────────────► CPU LOCAL (per-CPU spinlock_t)

mutex             SLEEPING ──────────────────► SLEEPING  (unchanged)
rt_mutex          SLEEPING ──────────────────► SLEEPING  (unchanged)
semaphore         SLEEPING ──────────────────► SLEEPING  (unchanged, no PI!)
rw_semaphore      SLEEPING ──────────────────► SLEEPING  (via rt_mutex)
```

---

## local_lock Mapping

```
Operation                     non-PREEMPT_RT          PREEMPT_RT
─────────────────────────────────────────────────────────────────────
local_lock(&llock)            preempt_disable()        per-CPU spin_lock()
local_unlock(&llock)          preempt_enable()         per-CPU spin_unlock()
local_lock_irq(&llock)        local_irq_disable()      per-CPU spin_lock()  ← IRQ NOT disabled!
local_unlock_irq(&llock)      local_irq_enable()       per-CPU spin_unlock()
local_lock_irqsave(&llock)    local_irq_save()         per-CPU spin_lock()
local_unlock_irqrestore(...)  local_irq_restore()      per-CPU spin_unlock()
```

---

## Spinning Lock Suffixes

```
spin_lock(&l)           ┐
spin_lock_bh(&l)        │  + disables softirq (BH)
spin_lock_irq(&l)       │  + disables interrupts
spin_lock_irqsave(&l,f) ┘  + saves & disables interrupts

        preemption disabled ────────────────────────── always (on non-RT)
        softirq   disabled ──────────────────────────── _bh(), _irq(), _irqsave()
        hardirq   disabled ───────────────────────────────────── _irq(), _irqsave()
```

### `spin_lock_irq` vs `spin_lock_irqsave`

**`spin_lock_irq(&l)`** — unconditionally disables interrupts, then acquires the lock.

```c
spin_lock_irq(&l);
// interrupts are OFF
spin_unlock_irq(&l);
// interrupts are back ON (unconditionally)
```

**`spin_lock_irqsave(&l, flags)`** — saves the current interrupt state into `flags` first, then disables and acquires.

```c
spin_lock_irqsave(&l, flags);
// interrupts are OFF
spin_unlock_irqrestore(&l, flags);
// interrupts restored to whatever they were BEFORE (may stay OFF)
```

The key difference — were interrupts already disabled when you called it?

```
Caller context          spin_lock_irq()          spin_lock_irqsave()
──────────────────────────────────────────────────────────────────────
IRQs were enabled       disables → restores ON   saves ON  → restores ON
IRQs were disabled      disables → restores ON   saves OFF → restores OFF
                                   ^^^^^^^^^^^               ^^^^^^^^^^^
                                   BUG: silently             correct
                                   re-enables IRQs
```

**When to use which:**
- `spin_lock_irq()` — only when you **know** IRQs are enabled (e.g. process context top-level). Simpler, no flags variable needed.
- `spin_lock_irqsave()` — safe **anywhere**: interrupt handlers, nested locks, shared utility functions. Always prefer in library/shared code where the caller's IRQ state is unknown.

---

## Reader/Writer Lock Fairness

```
                    rw_semaphore              rwlock_t
                    ────────────              ────────
non-PREEMPT_RT      fair (no writer           fair (spinning,
                    starvation)               no writer starvation)

PREEMPT_RT          rt_mutex-based:           rt_mutex-based:
                    ┌──────────────────────────────────────────┐
                    │ Low-prio READER preempted                │
                    │  → holds lock → HIGH-PRIO WRITER starved │
                    │                                          │
                    │ Low-prio WRITER preempted                │
                    │  → priority BOOSTED by readers           │
                    │  → writer completes, readers unblocked   │
                    └──────────────────────────────────────────┘
                    Writer-starvation possible; reader-starvation prevented.
```

---

## Lock Nesting Rules

```
non-PREEMPT_RT      ┌─────────────────────────────────────────────┐
────────────────    │  1. SLEEPING locks                          │
                    │     (mutex, semaphore, rw_semaphore…)       │
                    │           ↓                                 │
                    │  2. CPU LOCAL locks  (local_lock)           │
                    │           ↓                                 │
                    │  3. SPINNING locks                          │
                    │     (spinlock_t, rwlock_t,                  │
                    │      raw_spinlock_t, bit spinlocks)         │
                    └─────────────────────────────────────────────┘

PREEMPT_RT          ┌─────────────────────────────────────────────┐
─────────────       │  1. SLEEPING locks                          │
                    │     (mutex, rt_mutex, semaphore,            │
                    │      rw_sem, spinlock_t*, rwlock_t*,        │
                    │      local_lock*)                           │
                    │           ↓                                 │
                    │  2. raw_spinlock_t                          │
                    │     bit spinlocks                           │
                    └─────────────────────────────────────────────┘
```

- Sleeping locks **cannot** nest inside spinning locks.
- Spinning locks **can** nest inside sleeping or CPU-local locks.
- Same-category locks can nest (respecting deadlock-avoidance ordering).

---

## spinlock_t Task State on PREEMPT_RT (blocking acquisition)

```
  task->state = TASK_INTERRUPTIBLE
  spin_lock(&l):
    block():
      saved_state = task->state          ← save INTERRUPTIBLE
      task->state = TASK_UNINTERRUPTIBLE ← force uninterruptible while blocked
      schedule()
                       ┌── non-lock wakeup → saved_state = TASK_RUNNING
                       └── lock wakeup    → task->state = saved_state
                                                          (RUNNING in this case)
```

Real wakeups are never lost even when blocked on a spinlock.

---

## References

- `Documentation/locking/locktypes.rst`
