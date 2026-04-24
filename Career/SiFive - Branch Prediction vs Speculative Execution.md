---
tags:
  - career
  - interview
  - computer-architecture
  - CPU
  - RISC-V
---

# Branch Prediction vs Speculative Execution

---

## One-Line Distinction

> **Branch prediction** = *guessing which path to take*
> **Speculative execution** = *actually executing instructions before you know if they should run*

Branch prediction is the **decision**. Speculative execution is the **action** that follows.

---

## Analogy

```
You're driving to a fork in the road:

  Branch Prediction:
  "I think I should go LEFT"  ← just a guess, no action yet

  Speculative Execution:
  "I'll start driving LEFT now, and if I'm wrong I'll turn around"
  ← actually moving, doing work, before confirmed correct
```

---

## How They Work Together

```
  Branch instruction encountered
          │
          ▼
  ┌─────────────────────────┐
  │    Branch Predictor     │   ← makes the GUESS
  │                         │
  │  BHT lookup → TAKEN     │
  └────────────┬────────────┘
               │  "I predict TAKEN, go to foo"
               ▼
  ┌─────────────────────────┐
  │  Speculative Execution  │   ← acts on the guess
  │                         │
  │  fetch foo instructions │
  │  decode them            │
  │  execute them           │
  │  results held in ROB    │   ← Reorder Buffer, not committed yet
  │  (not written to regs   │
  │   or memory yet)        │
  └────────────┬────────────┘
               │
       branch resolves in EX stage
               │
       ┌───────┴────────┐
       │                │
   CORRECT          WRONG
       │                │
       ▼                ▼
  commit results    flush pipeline
  to registers      discard all
  and memory        speculative work
  ✅ no penalty     ❌ branch penalty
```

---

## Speculative Execution is Broader Than Branch Prediction

Branch prediction is just **one reason** to speculate. The CPU also speculates in other situations:

```
Triggers for Speculative Execution:
  ┌───────────────────────────────────────────────────────────────┐
  │                                                               │
  │  1. Branch Prediction          ← "I guess we take this path"  │
  │                                                               │
  │  2. Load Value Prediction      ← "I guess this load returns 5"│
  │     (rare, in some CPUs)                                      │
  │                                                               │
  │  3. Memory Disambiguation      ← "I guess this load doesn't   │
  │     (store-to-load forwarding)    depend on a pending store"  │
  │                                                               │
  │  4. Out-of-Order Execution     ← "I'll execute instruction B  │
  │                                   before A since B's data     │
  │                                   is ready and A's isn't"     │
  │                                                               │
  └───────────────────────────────────────────────────────────────┘
```

---

## Key Difference: What Each One Does

```
                   Branch Prediction         Speculative Execution
                   ─────────────────         ─────────────────────
What it does       predicts direction         executes instructions
                   of a branch                before they're confirmed

Output             a guess (taken/not taken)  actual computed results
                                              held in ROB

Hardware           Branch History Table (BHT) Reorder Buffer (ROB)
component          Return Stack Buffer (RSB)  reservation stations
                   Branch Target Buffer (BTB)

Can be wrong?      yes → triggers flush        yes → results discarded

Works without      no — needs a prediction     no — needs to know
the other?         to know what to execute     what path to execute
```

---

## The Reorder Buffer (ROB) — Heart of Speculative Execution

```
  In-order:    IF → ID → EX → MEM → WB   (results written immediately)

  Out-of-order with speculation:

  Instructions enter ROB in program order
  Execute out-of-order when operands ready
  Commit in-order only when confirmed not speculative

  ROB (circular buffer):
  ┌───────┬───────┬───────┬───────┬───────┬───────┐
  │ ADD   │ LOAD  │ BEQ   │ MUL   │ SUB   │ STORE │
  │ done  │ done  │pending│specul │specul │specul │
  └───────┴───────┴───────┴───────┴───────┴───────┘
    ▲ commit                               ▲ new entries
    head (oldest)                          tail (newest)

  BEQ still pending → MUL, SUB, STORE cannot commit
  BEQ resolves correct → MUL, SUB, STORE commit in order ✅
  BEQ resolves wrong   → MUL, SUB, STORE flushed         ❌
```

---

## Why This Led to Spectre

```
Spectre attack exploits BOTH:

  Step 1 — Branch Prediction:
    attacker trains the BHT to mispredict a bounds check

    if (index < array_size)       ← predictor trained to predict TAKEN
        value = array[index]      ← speculatively executed with
                                     out-of-bounds index!

  Step 2 — Speculative Execution:
    CPU speculatively executes the load with malicious index
    loads secret data into cache (side effect survives flush!)

  Step 3 — Cache timing side channel:
    attacker measures access time to probe array
    deduces what secret value was loaded

  Branch resolves: index >= array_size → speculative results flushed
  But cache state is NOT flushed → secret already leaked


Retpoline fixes the branch prediction part:
  replaces indirect branches (call *rax) with a trampoline
  that causes the predictor to always speculate at a safe
  infinite loop — attacker can no longer control the
  speculative execution target
```

---

## Summary

```
Branch Prediction
  └── a component inside the CPU frontend
  └── makes a directional guess (taken / not taken)
  └── feeds the target address to the fetch unit
  └── wrong guess → pipeline flush + refetch

Speculative Execution
  └── the broader mechanism that acts on predictions
  └── executes instructions before they are confirmed
  └── results buffered in ROB, not committed until safe
  └── wrong speculation → ROB flushed, state rolled back
  └── but microarchitectural side effects (cache, TLB) may persist
                                          ▲
                                    this is Spectre
```

> **One sentence:** Branch prediction tells the CPU *where* to speculate — speculative execution is the CPU *actually doing the work* before it knows if it should.
