---
tags:
  - linux-kernel
  - cpu-architecture
  - cache
  - mesi
---

# MESI Cache Coherency Protocol

MESI is a **cache coherency protocol** — it ensures multiple CPU cores don't disagree on the value of a memory location.

Each cache line carries one of four states:

---

## The Four States

```
M — Modified    dirty, only this core has it, RAM is stale
E — Exclusive   clean, only this core has it, matches RAM
S — Shared      clean, multiple cores have it, matches RAM
I — Invalid     this core's copy is garbage, don't use it
```

---

## State Transitions

### Scenario 1: First read (cold miss)
```
Core A reads address X
X is not in any cache → fetch from RAM
Core A: X → E (Exclusive, I'm the only one)
```

### Scenario 2: Another core reads the same line
```
Core B reads address X
Core A has X in E state → bus snoop detects this
Core A: E → S
Core B: I → S
Both cores now share a clean copy
```

### Scenario 3: Core A writes
```
Core A writes to X (currently in S)
Must invalidate all other copies first
→ sends "invalidate" broadcast on the bus
Core B: S → I  (forced out)
Core A: S → M  (now dirty, owns it exclusively)
```

### Scenario 4: Core B reads X while Core A has M
```
Core B reads X
Bus snoop hits → Core A must intervene
Core A: flushes dirty line to RAM (or directly to Core B)
Core A: M → S
Core B: I → S
```

---

## Full State Diagram

```
                   ┌─────────────────────────────┐
          other    │                             │ local read hit
          write    ▼                             │
         ┌──── Invalid (I) ◄────────────────────┤
         │         │                   other    │
         │    local│read                write   │
         │         ▼                            │
         │    Exclusive (E) ──────────────────► Shared (S)
         │         │           other read               │
         │    local│write                               │ other write
         │         ▼                                    │
         └───► Modified (M) ◄───────────────────────── ┘
                   │
              local write
              (stays M)
```

---

## Why This Matters in Practice

**False sharing** — the most common MESI-related bug:

```c
struct {
    int counter_a;  // used by Core 0
    int counter_b;  // used by Core 1
} shared;           // both fit in ONE cache line (64 bytes)
```

```
Core 0 writes counter_a → M, broadcasts invalidate
Core 1 writes counter_b → must fetch line again → M
Core 0 writes counter_a → must fetch line again → ...
```

Both cores are hammering **different variables** but they share a cache line — so they constantly invalidate each other. Fix: pad to separate cache lines.

```c
struct {
    int counter_a;
    char _pad[60];  // force counter_b onto next cache line
    int counter_b;
};
```

---

## MESI on Multi-Socket Systems

On multi-socket (NUMA) systems, snooping every write across all sockets is too expensive. Intel extends MESI to **MESIF** (adds Forward state) and AMD uses **MOESI** (adds Owned state) to reduce cross-socket traffic. But the core idea is the same.

| Variant | Extra State | Purpose |
|---|---|---|
| MESIF (Intel) | Forward | Designates one S-state holder to respond to requests, reducing duplicate responses |
| MOESI (AMD) | Owned | Allows dirty sharing — modified data forwarded directly to requester without writing back to RAM first |
