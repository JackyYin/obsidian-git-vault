---
tags:
  - career
  - interview
  - computer-architecture
  - RISC-V
  - CPU
---

# CPU Control Hazard — Deep Dive

---

## The Core Problem

A control hazard happens because the CPU uses a **pipeline** — it fetches and decodes future instructions *before* knowing the outcome of a branch.

```
  C code:
      if (x == 0)
          foo();
      else
          bar();

  Assembly:
      BEQ  x, zero, foo    ← branch instruction
      bar:  ...             ← falls through if x != 0
      foo:  ...             ← jumps here if x == 0
```

The CPU doesn't know which path to take until the branch is **resolved in the EX stage** — but by then it has already fetched 2 more instructions:

```
Cycle:    1       2       3       4       5       6
          ──────  ──────  ──────  ──────  ──────  ──────
BEQ       IF      ID      EX  ←─ branch resolved HERE
???               IF      ID      EX                    ← wrong fetch!
???                       IF      ID      EX            ← wrong fetch!
```

By cycle 3, the CPU has already fetched 2 wrong instructions into the pipeline. This is the **control hazard**.

---

## Solution 1 — Pipeline Stall (Bubble)

The simplest fix: do nothing and wait.

```
Cycle:    1       2       3       4       5       6       7
          ──────  ──────  ──────  ──────  ──────  ──────  ──────
BEQ       IF      ID      EX
                          │
                          └── branch resolved: jump to foo
NOP               IF      ID      NOP    (bubble inserted)
NOP                       IF      NOP    (bubble inserted)
foo:                              IF      ID      EX      WB
```

**2 wasted cycles per branch** — called the **branch penalty**.

Simple but slow. A modern CPU running at 3GHz with 15% branch frequency would waste enormous cycles this way.

---

## Solution 2 — Branch Prediction

The CPU **guesses** which path to take and continues fetching speculatively. If wrong, it flushes the pipeline.

### Static Prediction (simple)

```
Rule: always predict NOT TAKEN (keep fetching sequentially)

  BEQ  x, zero, foo
  bar: instr1          ← always fetch this (predict not taken)
       instr2          ← always fetch this

  If branch NOT taken → correct! no penalty ✅
  If branch IS taken  → flush pipeline, fetch foo → 2 cycle penalty ❌
```

Another static rule: **backward branches predict taken** (good for loops):

```
  loop:
    ...
    BNE  counter, zero, loop   ← backward branch → predict taken
                               → correct 99% of the time for tight loops
```

### Dynamic Prediction (modern CPUs)

Hardware maintains a **Branch History Table (BHT)** — a cache of recent branch outcomes.

```
Branch History Table (BHT)

  PC of branch    │  2-bit saturating counter  │  Prediction
  ──────────────  │  ──────────────────────────  │  ──────────
  0x1000 (BEQ)   │  11 (strongly taken)        │  TAKEN
  0x2040 (BNE)   │  00 (strongly not taken)    │  NOT TAKEN
  0x30A0 (BLT)   │  10 (weakly taken)          │  TAKEN
  0x4008 (BGE)   │  01 (weakly not taken)      │  NOT TAKEN


2-bit saturating counter states:

  00              01              10              11
  Strongly    →  Weakly      →  Weakly      →  Strongly
  Not Taken      Not Taken      Taken           Taken

  predict                        predict
  NOT TAKEN                      TAKEN

  ← branch taken          branch taken →
  ← branch not taken  branch not taken →

  "saturating" = stays at 00 or 11, doesn't wrap around
```

### Why 2-bit instead of 1-bit?

```
1-bit counter — breaks on loop exit:

  loop:  (taken 99 times, then not taken once at exit)

  iteration 98:  taken  → counter=1 → predict taken ✅
  iteration 99:  taken  → counter=1 → predict taken ✅
  exit:          NOT taken → counter=0 → predict not taken ✅ (but too late!)
  next loop entry: predict NOT taken ❌ (wrong! loop starts again)

  1-bit mispredicts TWICE per loop — once at exit, once at re-entry

2-bit counter — tolerates one anomaly:

  iteration 99:  taken  → counter=11 → predict taken ✅
  exit:          NOT taken → counter=10 → predict taken ❌ (one mispredict)
  next loop entry: taken → counter=11 → predict taken ✅

  2-bit mispredicts only ONCE per loop ✅
```

---

## Solution 3 — Branch Delay Slot (RISC classic)

Used in older RISC architectures (MIPS, early SPARC). The instruction **immediately after the branch always executes**, regardless of outcome.

```
  MIPS example:

      BEQ  x, zero, foo
      ADD  r1, r2, r3    ← DELAY SLOT — always executes!
      ...

  Compiler fills delay slot with useful work:
  → move an instruction from before the branch into the slot
  → if nothing useful available, insert NOP
```

**RISC-V deliberately removed delay slots** — they complicate the ISA and compilers. RISC-V relies purely on branch prediction + speculation instead.

---

## Solution 4 — Speculative Execution

Modern out-of-order CPUs go further — they don't just predict the branch, they **speculatively execute instructions down the predicted path**, buffer the results, and only commit them if the prediction was correct.

```
  BEQ  x, zero, foo   ← predict: NOT taken
  bar: LOAD r1, [mem]  ← speculatively execute
       ADD  r2, r1, 5  ← speculatively execute
       STORE r3, [mem] ← speculatively execute (buffered, not committed)
                              │
                        branch resolves: WRONG! x was 0
                              │
                        flush speculative results ← rolled back, never visible
                        fetch from foo:
  foo: ...             ← correct path executes
```

**This is exactly where Spectre came from** — speculative loads into cache leave measurable side effects even after being rolled back, which attackers can read via timing.

This is why **Retpoline** work is directly relevant — Retpoline is a software mitigation that prevents speculative execution from following indirect branch targets controlled by an attacker.

---

## Control Hazard in RISC-V Specifically

RISC-V defines **no branch delay slot** (unlike MIPS). The spec leaves branch prediction entirely to the microarchitecture:

```
RISC-V branch instructions:
  BEQ   rs1, rs2, offset    branch if equal
  BNE   rs1, rs2, offset    branch if not equal
  BLT   rs1, rs2, offset    branch if less than
  BGE   rs1, rs2, offset    branch if greater or equal
  BLTU  rs1, rs2, offset    unsigned less than
  BGEU  rs1, rs2, offset    unsigned greater or equal


RISC-V hint to hardware (from spec):
  backward branch (negative offset) → hint: likely TAKEN   (loops)
  forward  branch (positive offset) → hint: likely NOT TAKEN (if/else)

  Hardware is free to ignore this — just a software convention
```

---

## Summary

```
Solution              Penalty if wrong    Hardware cost    Used in
──────────────────    ────────────────    ─────────────    ────────────────
Pipeline stall        always N cycles     none             simple in-order
Static prediction     N cycles            none             simple CPUs
1-bit dynamic BHT     N cycles            small (BHT)      older CPUs
2-bit saturating BHT  N cycles            small (BHT)      most modern CPUs
Speculative exec      flush pipeline      large (ROB)      all modern OOO CPUs
Branch delay slot     none (by design)    compiler burden  MIPS, old SPARC
                                                           (NOT RISC-V)


N = pipeline depth between fetch and branch resolve
    typically 2–5 cycles in simple pipelines
    up to 15–20 cycles in deep out-of-order pipelines (Pentium 4 infamously)
```

> **For the SiFive interview:** Know that RISC-V has no delay slot, relies on dynamic branch prediction, and that deep speculation is what makes Spectre possible — directly tying Retpoline/CET experience to CPU architecture fundamentals.
