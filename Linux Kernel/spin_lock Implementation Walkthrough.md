---
tags:
  - linux-kernel
  - synchronization
  - spinlock
  - memory-barriers
  - atomic
  - arm64
  - x86
  - qspinlock
created: '2026-05-05'
---
# `spin_lock()` Implementation Walkthrough

> Tracing what `spin_lock()` and `spin_unlock()` actually compile to, and why they are guaranteed to block both **compiler reordering** and **CPU reordering**.

The motivating context: writing a thread-safe ring buffer kernel module. The natural question becomes "do I need a spinlock, and if so, what does it actually guarantee?"

---

## 1. Picking a Synchronization Model

| Model | Right tool |
|---|---|
| **SPSC** (1 producer, 1 consumer) | Lock-free with `smp_load_acquire`/`smp_store_release`. This is what `kfifo` does. |
| **MPSC / SPMC** | Lock on the "many" side, barriers on the "one" side. |
| **MPMC** (many producers, many consumers) | Just take a `spinlock_t` around the whole thing. |

Acquire/release is enough only when **each shared variable has exactly one writer**. The moment two threads can write the same variable (e.g. two producers updating `tail`), you need either a spinlock or atomic RMW (`cmpxchg`, `atomic_fetch_inc`).

### Lock-free SPSC ring buffer

```c
struct ringbuffer {
    char *data;
    u32 capacity;       // power of two
    u32 head;           // consumer writes, producer reads
    u32 tail;           // producer writes, consumer reads
};

int ringbuffer_put(struct ringbuffer *rb, char c)
{
    u32 tail = rb->tail;                                  // we own it
    u32 head = smp_load_acquire(&rb->head);               // see consumer
    if (tail - head == rb->capacity)
        return -EAGAIN;

    rb->data[RB_MASK(tail, rb->capacity)] = c;
    smp_store_release(&rb->tail, tail + 1);               // publish data via index
    return 0;
}

int ringbuffer_get(struct ringbuffer *rb, char *res)
{
    u32 head = rb->head;                                  // we own it
    u32 tail = smp_load_acquire(&rb->tail);               // see producer
    if (head == tail)
        return -EAGAIN;

    *res = rb->data[RB_MASK(head, rb->capacity)];
    smp_store_release(&rb->head, head + 1);
    return 0;
}
```

The pattern: **publish data through an index**. Producer uses release on the index it owns; consumer uses acquire on the index the producer owns.

### MPMC version with a spinlock

The `full`/`empty` check **must be inside** the critical section, otherwise a TOCTOU race lets two producers both pass the check then both write:

```c
spin_lock(&rb->lock);
if (ringbuffer_full(rb)) {
    spin_unlock(&rb->lock);
    return -EAGAIN;
}
rb->data[RB_MASK(rb->tail, rb->capacity)] = c;
rb->tail++;
spin_unlock(&rb->lock);
```

When using a spinlock, you do **not** need explicit `smp_load_acquire`/`smp_store_release` — `spin_lock`/`spin_unlock` already imply the necessary barriers (acquire on lock, release on unlock).

---

## 2. Why `spin_lock`/`spin_unlock` Block Reordering

There are **two layers** to deal with:

```
your code
   │
   │   ← compiler can reorder (layer 1)
   ▼
generated machine code
   │
   │   ← CPU can reorder (layer 2)
   ▼
visible memory operations
```

### Layer 1 — Compiler: `"memory"` clobber

`include/linux/compiler.h:85`
```c
# define barrier() __asm__ __volatile__("": : :"memory")
```

The `"memory"` clobber tells GCC: *"assume all memory was read and written here"*. The compiler must materialise pending writes, reload register-cached values, and not move memory ops across this point. Every kernel sync primitive ultimately uses this trick.

### Layer 2 — CPU: architecture-specific instructions

This is what differs by architecture:

| Architecture                | Lock instruction                                 | Unlock instruction     | Why it works                                                                         |
| --------------------------- | ------------------------------------------------ | ---------------------- | ------------------------------------------------------------------------------------ |
| **x86 (TSO)**               | `LOCK cmpxchg`                                   | plain `mov`            | `LOCK` prefix = full memory fence; TSO already orders prior stores → release is free |
| **arm64 (weakly ordered)**  | `LDAXR` (load-acquire exclusive) or `CASA` (LSE) | `STLR` (store-release) | These are dedicated ARMv8 acquire/release instructions                               |
| **RISC-V (weakly ordered)** | AMO with `.aq` suffix or explicit `fence`        | AMO with `.rl` suffix  | `.aq`/`.rl` bits implement acquire/release                                           |

### Acquire/release as one-way barriers

From `Documentation/memory-barriers.txt:475-511`:

```
─── spin_lock (ACQUIRE) ──────────────  ← can leak in
       │
       │  critical section: ops are pinned here,
       │  cannot escape outward
       │
─── spin_unlock (RELEASE) ────────────  ← can leak in
```

Operations *inside* cannot escape outward (the dangerous direction). Operations *outside* can drift inward (harmless). That asymmetry is why two consecutive critical sections are *not* a full barrier — and why `smp_mb__after_spinlock()` exists for the rare case you need one (`arch/arm64/include/asm/spinlock.h:12`).

---

## 3. The Full Call Chain on arm64

```
       spin_lock(spinlock_t *)
              │
              ▼
     ┌─── include/linux/spinlock.h:349 ─────────────────────┐
     │ raw_spin_lock(&lock->rlock);                         │
     └──────────────────────────────────────────────────────┘
              │
              ▼
     raw_spin_lock(raw_spinlock_t *)   [macro → _raw_spin_lock]
              │
              ▼
     ┌─── kernel/locking/spinlock.c:154 ────────────────────┐
     │ void __lockfunc _raw_spin_lock(raw_spinlock_t *lock) │
     │ { __raw_spin_lock(lock); }                           │
     └──────────────────────────────────────────────────────┘
              │
              ▼
     ┌─── include/linux/spinlock_api_smp.h:130 ─────────────┐
     │ static inline void __raw_spin_lock(...)              │
     │ {                                                    │
     │     preempt_disable();                               │
     │     spin_acquire(&lock->dep_map, ...);   /* lockdep */│
     │     LOCK_CONTENDED(lock, do_raw_spin_trylock,        │
     │                          do_raw_spin_lock);          │
     │ }                                                    │
     └──────────────────────────────────────────────────────┘
              │
              ▼
     ┌─── include/linux/spinlock.h:184 ─────────────────────┐
     │ static inline void do_raw_spin_lock(...)             │
     │ {                                                    │
     │     __acquire(lock);                                 │
     │     arch_spin_lock(&lock->raw_lock);                 │
     │     mmiowb_spin_lock();                              │
     │ }                                                    │
     └──────────────────────────────────────────────────────┘
              │
              ▼   #define arch_spin_lock(l)  queued_spin_lock(l)
              │   (include/asm-generic/qspinlock.h:146)
              ▼
     ┌─── include/asm-generic/qspinlock.h:107 ──────────────┐
     │ static __always_inline void                          │
     │ queued_spin_lock(struct qspinlock *lock)             │
     │ {                                                    │
     │     int val = 0;                                     │
     │     if (likely(atomic_try_cmpxchg_acquire(           │
     │             &lock->val, &val, _Q_LOCKED_VAL)))       │
     │         return;                                      │
     │     queued_spin_lock_slowpath(lock, val);            │
     │ }                                                    │
     └──────────────────────────────────────────────────────┘
              │
              ▼   instrumentation wrapper (KASAN/KCSAN)
              │   include/linux/atomic/atomic-instrumented.h:1296
              ▼
     ┌─── include/linux/atomic/atomic-arch-fallback.h:2155 ─┐
     │ raw_atomic_try_cmpxchg_acquire(...)                  │
     │ {                                                    │
     │ #if defined(arch_atomic_try_cmpxchg_acquire)         │
     │     return arch_atomic_try_cmpxchg_acquire(...);     │
     │ #elif defined(arch_atomic_try_cmpxchg_relaxed)       │
     │     bool ret = arch_atomic_try_cmpxchg_relaxed(...); │
     │     __atomic_acquire_fence();                        │
     │     return ret;                                      │
     │ ...                                                  │
     │ }                                                    │
     └──────────────────────────────────────────────────────┘
              │
              ▼   on arm64:
              │   #define arch_cmpxchg_acquire(...) __cmpxchg_wrapper(_acq, ...)
              │   (arch/arm64/include/asm/cmpxchg.h:186)
              ▼
     ┌─── arch/arm64/include/asm/cmpxchg.h:21-43 ───────────┐
     │ asm volatile(ARM64_LSE_ATOMIC_INSN(                  │
     │ /* LL/SC path: */                                    │
     │ "  prfm   pstl1strm, %2\n"                           │
     │ "1: ld" #acq "xr" #sfx "  %0, %2\n"  ← ldaxr (acq=a) │
     │ "  st" #rel "xr" #sfx "  %w1, %3, %2\n"  ← stxr      │
     │ "  cbnz   %w1, 1b\n",                                │
     │ /* LSE atomics path (CPU has FEAT_LSE): */           │
     │ "  cas" #acq_lse #rel #sfx "  %3, %0, %2\n")         │
     │     ...                                              │
     │     : "memory");          ← compiler barrier         │
     └──────────────────────────────────────────────────────┘
              │
              ▼
       ┌────────────────────────────────────────┐
       │  Hardware instruction actually emitted │
       ├────────────────────────────────────────┤
       │  CPU without LSE: LDAXR + STXR loop    │
       │  CPU with LSE:    CASA (compare-and-   │
       │                   swap, acquire)       │
       └────────────────────────────────────────┘
```

### What each layer contributes

| Layer | What it adds |
|---|---|
| `spin_lock` (`spinlock.h:349`) | Type wrapper around `raw_spin_lock` |
| `_raw_spin_lock` (`spinlock.c:154`) | Out-of-line stub when not inlined; `EXPORT_SYMBOL` host |
| `__raw_spin_lock` (`spinlock_api_smp.h:130`) | `preempt_disable()` + lockdep tracking + `LOCK_CONTENDED` |
| `do_raw_spin_lock` (`spinlock.h:184`) | Sparse `__acquire` annotation + MMIO write barrier |
| `arch_spin_lock` → `queued_spin_lock` (`qspinlock.h:107`) | The actual lock algorithm: cmpxchg fast path, else MCS queue slow path |
| `atomic_try_cmpxchg_acquire` | KASAN/KCSAN instrumentation wrapper |
| `raw_atomic_try_cmpxchg_acquire` (`atomic-arch-fallback.h:2155`) | Picks strongest available arch primitive |
| `arch/arm64/.../cmpxchg.h:21-43` | Inline asm with `"memory"` clobber → emits `LDAXR` (or LSE `CASA`) |

---

## 4. The x86 Path (Shorter at the Bottom)

Same path through `queued_spin_lock` → `atomic_try_cmpxchg_acquire`, but the arch backend lands in:

```
arch/x86/include/asm/cmpxchg.h:109
   asm volatile(lock "cmpxchgl %2,%1" : ... : "memory");
                      ▲
              LOCK prefix = full hw barrier
```

x86 doesn't have a separate "acquire cmpxchg" because every `LOCK`-prefixed RMW is already a full memory fence — acquire is free under TSO.

---

## 5. Two Barriers Per Layer, Restated

Every link in the chain carries **both** kinds of protection:

1. **Compiler reorder protection** — every `asm volatile (... : "memory")` along the path. The `volatile` prevents elision; the `"memory"` clobber prevents reads/writes from being moved across.

2. **CPU reorder protection** — only at the very last step.
   - arm64: the `a` in `ldaxr` (or LSE `casa`) makes the load architecturally an acquire.
   - x86: the `LOCK` prefix makes the RMW a full fence.

Everything between `spin_lock()` at the top and that final inline asm at the bottom is plumbing — type wrappers, lockdep, preempt counting, instrumentation. **The actual ordering semantics are delivered by one instruction at the bottom of the stack.**

---

## 6. The `unlock` Side, Briefly

`include/asm-generic/qspinlock.h:123-129`
```c
static __always_inline void queued_spin_unlock(struct qspinlock *lock)
{
    /* unlock() needs release semantics: */
    smp_store_release(&lock->locked, 0);
}
```

On arm64 that resolves to `STLR` (`arch/arm64/include/asm/barrier.h:123-148`):
```c
asm volatile ("stlr %w1, %0"
        : "=Q" (*__p)
        : "rZ" (*(__u32 *)__u.__c)
        : "memory");
```

On x86 it's a plain `mov` plus `"memory"` clobber — TSO already gives release semantics for free.

---

## 7. Quick Reference: When to Use What

| Need | Use |
|---|---|
| One thread total | nothing |
| 1 writer + N readers per variable (SPSC, SPMC index) | `smp_load_acquire` / `smp_store_release` |
| Multiple writers to the same variable, simple critical section | `spin_lock` / `spin_unlock` |
| Multiple writers, lock-free | atomic RMW (`cmpxchg`, `atomic_fetch_*`) with appropriate ordering |
| Existing ring buffer | `kfifo` (SPSC lock-free) or `kfifo_in_spinlocked()` (MPMC) |

---

## References

| Claim | Source |
|---|---|
| `barrier()` macro definition | `include/linux/compiler.h:85` |
| `spin_lock` → `raw_spin_lock` | `include/linux/spinlock.h:349` |
| `_raw_spin_lock` out-of-line | `kernel/locking/spinlock.c:154` |
| `__raw_spin_lock` (preempt + lockdep) | `include/linux/spinlock_api_smp.h:130` |
| `do_raw_spin_lock` → `arch_spin_lock` | `include/linux/spinlock.h:184` |
| qspinlock fast path uses `atomic_try_cmpxchg_acquire` | `include/asm-generic/qspinlock.h:107` |
| qspinlock unlock = `smp_store_release` | `include/asm-generic/qspinlock.h:123-129` |
| Atomic fallback chain | `include/linux/atomic/atomic-arch-fallback.h:2155` |
| arm64 cmpxchg → `LDAXR`/`STXR`/`CASA` | `arch/arm64/include/asm/cmpxchg.h:21-43` |
| arm64 `STLR` for release | `arch/arm64/include/asm/barrier.h:123-148` |
| arm64 `LDAR` for acquire | `arch/arm64/include/asm/barrier.h:158-183` |
| x86 `LOCK cmpxchg` | `arch/x86/include/asm/cmpxchg.h:85-124` |
| ACQUIRE/RELEASE semantics doc | `Documentation/memory-barriers.txt:475-511` |
| `smp_mb__after_spinlock` | `arch/arm64/include/asm/spinlock.h:12` |

---

## Related Notes

- [[CPU Pipeline and Memory Barriers]] — `READ_ONCE`, `WRITE_ONCE`, `smp_rmb`, fence instructions, load/store tearing.
- [[CPU Cache Addressing]] — VIVT/PIPT/VIPT, cache hierarchy, maintenance APIs across x86/arm64/RISC-V.
- [[Why Kernel Code Patching Goes Through D-cache]] — the asymmetric I/D cache model and self-modifying code dance.
