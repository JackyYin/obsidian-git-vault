---
tags:
  - linux-kernel
  - cpu
  - memory-barrier
  - pipeline
  - RISC-V
  - x86
  - arm64
  - computer-architecture
  - compiler
---
# CPU Pipeline and Memory Barriers

---

## Why Memory Barriers Exist — It's All Because of the Pipeline

The core insight: **memory barriers are software's way of talking to the CPU pipeline.**

But there is a second, equally important layer: **the compiler**. The compiler can reorder, tear, merge, refetch, or invent memory accesses long before the CPU sees them. So in Linux there are *two* kinds of "barrier":

- **Compiler barriers** — affect what instructions the compiler emits (e.g. `READ_ONCE`, `WRITE_ONCE`, `barrier()`).
- **CPU / memory barriers** — affect how the CPU pipeline orders execution at runtime (e.g. `smp_rmb`, `smp_wmb`, `smp_mb`).

Confusing the two is the most common bug in this area.

---

## What the Pipeline Does to Memory Operations

You write this in C:
```c
a = 1;
b = 2;
```

You expect this order in hardware. But the pipeline doesn't guarantee it.

```
What YOU wrote:          What the CPU MAY do:
──────────────           ───────────────────
STORE a = 1              STORE b = 2   ← reordered!
STORE b = 2              STORE a = 1   ← reordered!
```

This is called **memory reordering** — and it happens for real, legitimate performance reasons inside the pipeline.

---

## Why the Pipeline Reorders Memory Operations

### Reason 1 — Store Buffer

The CPU doesn't write to cache immediately. It uses a **store buffer** to decouple the pipeline from slow cache/memory:

```
  CPU Core
  ┌──────────────────────────────────────────────────────┐
  │   Pipeline                                           │
  │   ────────                                           │
  │   STORE a=1 ──▶  Store Buffer  ──▶ (drains later)    │
  │   STORE b=2 ──▶  Store Buffer  ──▶ (drains later)    │
  │                  ┌──────────┐                        │
  │                  │ a=1      │                        │
  │                  │ b=2      │                        │
  │                  └────┬─────┘                        │
  │                       │  drains to cache when free   │
  └───────────────────────┼──────────────────────────────┘
                          ▼
                    L1 D-cache
```

The store buffer drains **out of order** based on cache availability. So even if you wrote `a=1` before `b=2`, `b` might reach cache first if `b`'s cache line happens to be hot.

### Reason 2 — Load Speculation

The CPU speculatively executes loads **before** knowing if an earlier store will affect them:

```
  STORE a = 1       ← store goes to store buffer (slow)
  LOAD  r1 = b      ← CPU doesn't wait — speculatively loads b NOW
                       even though the store hasn't committed yet
```

### Reason 3 — Out-of-Order Execution

The pipeline executes instructions when their **operands are ready**, not in program order:

```
  LOAD  r1 = [addr_a]   ← cache miss, will take 200 cycles
  ADD   r2, r3, r4      ← r3, r4 already in registers → execute NOW
  STORE [addr_b] = r2   ← r2 now ready → execute NOW
```

---

## The 4 Types of Reordering

```
  Type          What gets reordered       Example
  ──────────    ───────────────────       ───────────────────────────
  StoreStore    store A before store B    a=1; b=2  →  b=2; a=1
  LoadLoad      load A before load B      r=a; s=b  →  s=b; r=a
  LoadStore     load before store         r=a; b=1  →  b=1; r=a
  StoreLoad     store before load         a=1; r=b  →  r=b; a=1
                ← MOST dangerous —
                  even x86 TSO allows this one
```

---

## Two Layers of Reordering: Compiler vs CPU

Before any CPU sees your code, the **compiler** has already had a chance to mangle memory accesses. It can:

- **Reorder** independent loads/stores
- **Refetch** a value (re-issue a load that was already done)
- **Merge** two loads of the same location into one
- **Invent** loads or stores that don't appear in source
- **Tear** a single access into multiple smaller ones (load/store tearing)
- **Eliminate** "dead" stores

Compiler barriers fix that layer. CPU barriers fix the pipeline layer. **They are not interchangeable.**

```
   Source code
        │
        ▼
   ┌────────────────────────┐
   │  Compiler              │ ← READ_ONCE / WRITE_ONCE / barrier()
   │  may reorder/tear/...  │   stop here
   └──────────┬─────────────┘
              │ machine code
              ▼
   ┌────────────────────────┐
   │  CPU pipeline          │ ← smp_rmb / smp_wmb / smp_mb
   │  may reorder at runtime│   stop here
   └────────────────────────┘
```

---

## `READ_ONCE()` / `WRITE_ONCE()` — Compiler Barriers Only

These are **purely compiler-level** primitives. They prevent compiler optimisations on a single access. **They do not emit any hardware fence and do not stop CPU reordering.**

### Definition

`include/asm-generic/rwonce.h:44`:
```c
#define __READ_ONCE(x)   (*(const volatile __unqual_scalar_typeof(x) *)&(x))
```

`include/asm-generic/rwonce.h:55`:
```c
#define __WRITE_ONCE(x, val)                       \
do {                                               \
    *(volatile typeof(x) *)&(x) = (val);           \
} while (0)
```

These are pure `volatile` casts. **No `asm volatile`, no `fence`, no `dmb`, no `mfence`.**

### What the kernel docs say

`Documentation/memory-barriers.txt:1834-1835`:

> "Please note that these compiler barriers have no direct effect on the CPU, which may then reorder things however it wishes."

The barrier table at `Documentation/memory-barriers.txt:1841-1848` lists `READ_ONCE()` only under **ADDRESS DEPENDENCY** — *not* among the read/write/general CPU barriers.

### What `READ_ONCE`/`WRITE_ONCE` actually prevent

| Behaviour                                                                            | Prevented? | Why                                                 |
| ------------------------------------------------------------------------------------ | ---------- | --------------------------------------------------- |
| Compiler reordering of *this* access against other `READ_ONCE`/`WRITE_ONCE`/barriers | ✅          | `volatile` semantics                                |
| Load tearing / store tearing                                                         | ✅          | `volatile` access must be a single memory-reference |
| Compiler refetching / merging multiple accesses                                      | ✅          | `volatile`                                          |
| Compiler inventing extra loads/stores                                                | ✅          | `volatile`                                          |
| **CPU load-load / store-store reordering**                                           | ❌          | No fence emitted                                    |
| **CPU speculative loads**                                                            | ❌          | No fence emitted                                    |
| **Inter-CPU visibility ordering**                                                    | ❌          | No fence emitted                                    |

### The arm64 + LTO exception (and why it confirms the rule)

`arch/arm64/include/asm/rwonce.h:8` overrides `__READ_ONCE` to emit `ldar`/`ldapr` (acquire-load) — but **only when `CONFIG_LTO=y`**. The comment (`arch/arm64/include/asm/rwonce.h:26-35`) explains:

> "When building with LTO, there is an increased risk of the compiler converting an address dependency headed by a `READ_ONCE()` invocation into a control dependency..."

So this is a **compiler workaround**, not a memory-model promise. Without LTO, arm64 falls through to the generic plain-`volatile` definition (line 71). The rule still holds: `READ_ONCE()` is fundamentally a compiler-level construct.

---

## Load / Store Tearing

**Load tearing**: the compiler implements one C-level load as **multiple machine-level loads**.
**Store tearing**: same, for stores.

Both can produce values that were never written by the source code, because a concurrent writer can change the location between the partial accesses.

Reference: `Documentation/memory-barriers.txt:1781-1825`.

### Example 1 — Store tearing on a large constant

```c
p = 0x00010002;     /* 32-bit store */
```

Quoting `Documentation/memory-barriers.txt:1784-1787`:

> "given an architecture having 16-bit store instructions with 7-bit immediate fields, the compiler might be tempted to use two 16-bit store-immediate instructions to implement the following 32-bit store"

The 32-bit constant `0x00010002` won't fit into a single 16-bit store-immediate, but its two halves (`0x0001` and `0x0002`) each fit in a 7-bit immediate. So the compiler may emit:

```
   STORE16 [p+0], 0x0001     ; high half
   STORE16 [p+2], 0x0002     ; low half
```

A concurrent reader can observe the half-updated state — `0x00010000` or `0x00000002` — values **never assigned** by the source code.

`WRITE_ONCE(p, 0x00010002)` forces a single 32-bit store, eliminating the tear. `Documentation/memory-barriers.txt:1795-1796`:

> "In fact, a recent bug (since fixed) caused GCC to incorrectly use this optimization in a volatile store."

### Example 2 — Load and store tearing via packed struct

```c
struct __attribute__((__packed__)) foo {
    short a;     /* offset 0, 2 bytes */
    int   b;     /* offset 2, 4 bytes — STRADDLES a 32-bit boundary */
    short c;     /* offset 6, 2 bytes */
};                /* sizeof = 8 bytes */

struct foo foo1, foo2;
...
foo2.a = foo1.a;
foo2.b = foo1.b;
foo2.c = foo1.c;
```

`Documentation/memory-barriers.txt:1816-1820`:

> "the compiler would be well within its rights to implement these three assignment statements as a pair of 32-bit loads followed by a pair of 32-bit stores. This would result in load tearing on 'foo1.b' and store tearing on 'foo2.b'."

The compiler is copying the whole 8-byte struct, not the three named fields. It is faster to issue two aligned 32-bit accesses than three unaligned ones:

```
   foo1 layout (8 bytes total):
   ┌─────────┬─────────────────────┬─────────┐
   │  a (2)  │       b (4)         │  c (2)  │
   └─────────┴─────────────────────┴─────────┘
   bytes  0-1     2-3   |   4-5      6-7
                        ↑
                    32-bit boundary

   Compiler emits:
     LOAD32  r1, [foo1 + 0]    ; bytes 0..3 = a + low half of b
     LOAD32  r2, [foo1 + 4]    ; bytes 4..7 = high half of b + c
     STORE32 [foo2 + 0], r1
     STORE32 [foo2 + 4], r2
```

`foo1.b` is **reconstructed from two halves coming from two independent loads**. If another CPU writes `foo1.b = 0xAAAABBBB` between the two 32-bit loads, the consumer can read a value that splices the old high half with the new low half (or vice-versa) — never `0xAAAABBBB` itself, never the previous value, but a phantom value that was never stored.

The fix (`Documentation/memory-barriers.txt:1823-1825`):

```c
foo2.a = foo1.a;
WRITE_ONCE(foo2.b, READ_ONCE(foo1.b));
foo2.c = foo1.c;
```

`READ_ONCE(foo1.b)` forces a single 32-bit load specifically of `b`; `WRITE_ONCE(foo2.b, ...)` forces a single 32-bit store specifically of `b`. The compiler is no longer allowed to fold the field-level access into the wider whole-struct copy.

### Why `volatile` prevents tearing

A `volatile` access must correspond to a single memory-reference instruction; the compiler may not split, merge, or duplicate it. Provided the type is naturally aligned and fits a native word — the kernel asserts this via `compiletime_assert_rwonce_type` (`include/asm-generic/rwonce.h:35`) — the resulting machine instruction is **single-copy atomic** on every architecture Linux supports, so tearing is impossible.

The same property is what stops invented loads/stores, load merging, load fusing across loops, and dead-store elimination.

---

## CPU Memory Barriers — Telling the Pipeline to Stop Reordering

A memory barrier instruction **forces the store buffer to drain** and **prevents the pipeline from reordering across it**:

```
  WITHOUT barrier:                WITH barrier:

  STORE a = 1                     STORE a = 1
  STORE b = 2                     ───── smp_wmb() ─────  ← FENCE w,w
                                  STORE b = 2

  Pipeline free to reorder        Pipeline MUST complete
  b=2 might be visible            a=1 before making b=2
  before a=1                      visible to other cores
```

### Linux's CPU memory barriers

From `Documentation/memory-barriers.txt:1841-1848`:

```
  TYPE                MANDATORY     SMP-CONDITIONAL
  ==================  ============  ===============
  GENERAL             mb()          smp_mb()
  WRITE               wmb()         smp_wmb()
  READ                rmb()         smp_rmb()
  ADDRESS DEPENDENCY                READ_ONCE()
```

| Barrier         | Prevents              | Pipeline effect                                                      |
|-----------------|-----------------------|----------------------------------------------------------------------|
| `smp_wmb()`     | StoreStore reorder    | drain store buffer; all prior stores commit before any later store   |
| `smp_rmb()`     | LoadLoad reorder      | invalidate speculative loads; later loads cannot start before this   |
| `smp_mb()`      | all reordering        | full pipeline drain — most expensive                                 |

`READ_ONCE()` appears in this table only as an **address-dependency barrier** — meaning that on every architecture Linux supports (after Alpha workarounds inside `READ_ONCE`), `b = READ_ONCE(a); x = *b;` is naturally ordered by the pointer dependency, with no fence emitted. This is a hardware property of dependency chains, not something `READ_ONCE` itself emits.

`Documentation/memory-barriers.txt:1851-1852`:

> "All memory barriers except the address-dependency barriers imply a compiler barrier. Address dependencies do not impose any additional compiler ordering."

---

## `READ_ONCE()` vs `smp_rmb()` — Different Layers

The two operate on **different layers**, and you typically need both:

| | `READ_ONCE(x)` | `smp_rmb()` |
|---|---|---|
| Layer | Compiler | CPU pipeline |
| Mechanism | `volatile` cast | Hardware fence (`FENCE r,r` / `lfence` / `dmb ishld`) |
| Prevents tearing? | **Yes** | No |
| Prevents compiler reordering? | Of `*x` itself | Implies compiler barrier for *all* memory |
| Prevents compiler refetching/merging? | Yes | No |
| Prevents **CPU** load-load reorder? | **No** | **Yes** |
| Prevents **CPU** speculation? | No | Yes |
| Scope | One specific access | A point in the instruction stream |
| Cost on RISC-V | 0 instructions | 1 `FENCE r,r` |
| Cost on x86 | 0 instructions | 0 instructions (TSO already orders LL) |

A correct producer/consumer needs both:

```c
/* Producer (CPU 0) */
WRITE_ONCE(data, 42);   // (1) compiler must emit this exact store
smp_wmb();              // (2) CPU must finish (1) before (3) is visible
WRITE_ONCE(flag, 1);    // (3) compiler must not tear / hoist

/* Consumer (CPU 1) */
while (!READ_ONCE(flag))  // (4) compiler must reload each iteration
    cpu_relax();
smp_rmb();                // (5) CPU must not load (6) before (4)
val = READ_ONCE(data);    // (6) compiler must not tear / refetch
```

What each piece protects:
- `WRITE_ONCE`/`READ_ONCE` — stop the **compiler** from caching `flag` in a register, tearing the access, or hoisting `data` out of the loop.
- `smp_wmb`/`smp_rmb` — stop the **CPU** from reordering the data and flag accesses at runtime.

Drop either layer and the code is broken on a sufficiently relaxed machine.

---

## Real Example — Linux Kernel Producer/Consumer

```c
/* Producer (CPU 0) */
WRITE_ONCE(data, 42);   // (1) write data
smp_wmb();              // (2) StoreStore barrier
WRITE_ONCE(flag, 1);    // (3) signal ready

/* Consumer (CPU 1) */
while (!READ_ONCE(flag))   // (4) wait for signal
    cpu_relax();
smp_rmb();                  // (5) LoadLoad barrier
use(READ_ONCE(data));       // (6) guaranteed to see 42
```

Without `smp_wmb()`:
```
CPU 0 pipeline might reorder:
  flag = 1      ← visible first!
  data = 42     ← visible second

CPU 1 sees flag=1, reads data → gets 0 (stale!)  ← data race bug
```

Without `READ_ONCE` on `flag`, the compiler is free to hoist the load out of the loop entirely (`while (!flag)` becomes `if (!flag) for(;;);`).

---

## x86 vs RISC-V — Why RISC-V Needs More Barriers

```
  x86 (TSO — Total Store Order)
  ─────────────────────────────
  Hardware prevents StoreStore reordering automatically
  Hardware prevents LoadLoad reordering automatically
  Hardware prevents LoadStore reordering automatically
  Only StoreLoad can reorder (mfence needed)

  → smp_rmb() / smp_wmb() are NOPs on x86!
     only smp_mb() compiles to a real instruction
       (mfence, or "lock addl $0,(%rsp)")


  RISC-V (RVWMO — Relaxed Memory Order)
  ──────────────────────────────────────
  ALL four reordering types are allowed by hardware
  → every barrier compiles to a real FENCE instruction
  → kernel driver code on RISC-V needs MORE explicit barriers
  → missing a barrier on RISC-V = silent data corruption

  smp_rmb()  →  FENCE r,r    (real instruction)
  smp_wmb()  →  FENCE w,w    (real instruction)
  smp_mb()   →  FENCE rw,rw  (real instruction)
```

Note: `READ_ONCE` / `WRITE_ONCE` cost the same on every arch — **zero extra instructions**, because they are compiler-only.

---

## Summary — The Chain of Causation

```
Compiler optimisations
  │
  ├── Reorder / merge / refetch / invent / tear stores and loads
  │
  ├── fixed by:
  │     ├── READ_ONCE / WRITE_ONCE   (per-access compiler barrier)
  │     └── barrier()                (whole-program compiler barrier)
  │
Out-of-Order Pipeline
  │
  ├── Store Buffer          → causes StoreStore, StoreLoad reordering
  ├── Load Speculation      → causes LoadLoad, LoadStore reordering
  └── Out-of-Order Exec     → causes all types of reordering
  │
  ├── fixed by:
  │     ├── smp_wmb() / FENCE w,w     → StoreStore
  │     ├── smp_rmb() / FENCE r,r     → LoadLoad
  │     └── smp_mb()  / FENCE rw,rw   → all four
  │
  ▼
Correct multi-core behaviour
(driver code, lock-free data structures, ring buffers...)
```

> **Key insight:** `READ_ONCE` / `WRITE_ONCE` and `smp_rmb` / `smp_wmb` solve different problems. The first stops the compiler from rewriting your code; the second stops the CPU from rewriting it at runtime. Real lock-free code almost always needs both.

---

## References

| Claim | File:line |
|---|---|
| `READ_ONCE`/`WRITE_ONCE` are pure `volatile` casts | `include/asm-generic/rwonce.h:44, 55` |
| Header documents purpose as "prevent compiler from..." | `include/asm-generic/rwonce.h:1-19` |
| Compile-time assert that the access is a native word | `include/asm-generic/rwonce.h:35` |
| Compiler barriers have no direct effect on the CPU | `Documentation/memory-barriers.txt:1834-1835` |
| Tearing definition — single large access split into smaller ones | `Documentation/memory-barriers.txt:1781-1787` |
| Constant-split tearing example (`p = 0x00010002`) | `Documentation/memory-barriers.txt:1789-1799` |
| Packed-struct example: 2× 32-bit loads + 2× 32-bit stores | `Documentation/memory-barriers.txt:1801-1825` |
| Kernel barrier categorisation table | `Documentation/memory-barriers.txt:1841-1848` |
| Address-dependency barriers don't add compiler ordering | `Documentation/memory-barriers.txt:1851-1852` |
| arm64 LTO override is a compiler workaround | `arch/arm64/include/asm/rwonce.h:26-35, 71` |
