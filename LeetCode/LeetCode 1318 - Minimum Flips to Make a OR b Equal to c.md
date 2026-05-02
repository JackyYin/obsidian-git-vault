---
tags:
  - leetcode
  - C
  - bits_operation
---

# LeetCode 1318 — Minimum Flips to Make a OR b Equal to c

## Problem

Given three integers `a`, `b`, `c`, return the minimum number of bit flips required (in `a` and/or `b`) so that `a | b == c`. Each flipped bit in either `a` or `b` counts as one operation.

## Implementation

```c
int minFlips(int a, int b, int c) {
    int xored = (a | b) ^ c;
    int anded = (a & b) & xored;

    int cnt = 0;
    while (xored) {
        xored &= (xored - 1);
        cnt++;
    }

    while (anded) {
        anded &= (anded - 1);
        cnt++;
    }

    return cnt;
}
```

## The per-bit cost table

For each bit position, compare `(a_i, b_i)` against `c_i`:

| a | b | a\|b | c | flips needed |
|---|---|------|---|--------------|
| 0 | 0 | 0    | 0 | 0 |
| 0 | 0 | 0    | 1 | **1** (flip either a or b to 1) |
| 0 | 1 | 1    | 0 | **1** (flip b → 0) |
| 1 | 0 | 1    | 0 | **1** (flip a → 0) |
| 1 | 1 | 1    | 0 | **2** (flip BOTH to 0) |
| —  | —  | =    | = | 0 |

**Key observation:**

- A bit disagrees with `c` → **at least 1 flip**.
- That bit *also* has `a=1 AND b=1 AND c=0` → **add 1 more flip** (must clear both).

## How the two masks capture exactly that

```c
int xored = (a | b) ^ c;       // disagreement bits — each costs ≥ 1
int anded = (a & b) & xored;   // among disagreements, where BOTH a,b are 1 — extra +1
```

- `xored` — bits where `a|b` differs from `c`. Each contributes **1 flip**.
- `anded` — the subset of disagreement bits where `a=1 AND b=1`. Since `a&b=1` forces `a|b=1`, the disagreement must mean `c=0`. These are precisely the "row 5" cases above. Each contributes **1 extra flip**.

**Total flips = popcount(`xored`) + popcount(`anded`).**

## The `while (n) { n &= n - 1; cnt++; }` loop

This is **Brian Kernighan's popcount**. `n & (n - 1)` clears the lowest set bit, so the loop runs once per 1-bit — O(popcount(n)) instead of O(bit-width).

```
n      = 1011 0100
n - 1  = 1011 0011
n & .. = 1011 0000   ← lowest 1 cleared
```

## Trace: `a = 2 (010)`, `b = 6 (110)`, `c = 5 (101)`

```
        bit:  2 1 0
        a  =  0 1 0
        b  =  1 1 0
       a|b =  1 1 0
        c  =  1 0 1

xored = (a|b) ^ c
      bit 2: 1^1 = 0
      bit 1: 1^0 = 1   ← disagree (a|b=1, c=0)
      bit 0: 0^1 = 1   ← disagree (a|b=0, c=1)
xored = 011  → popcount = 2

a&b = 010
anded = (a&b) & xored
      = 010 & 011 = 010 → popcount = 1
      (bit 1: a=1, b=1, c=0 — needs the extra flip)

total = 2 + 1 = 3 ✓
```

## Mental model

Think of `xored` as **"which bits are wrong"** and `anded` as **"which wrong bits cost double."** Sum the popcounts and you're done — no per-bit iteration over `a`, `b`, `c` needed.

## Complexity

- **Time:** O(popcount(xored) + popcount(anded)), bounded by O(bit-width) = O(32).
- **Space:** O(1).

## Related

- [[Bits Operation]]
- [[LeetCode 338 - Counting Bits]] — same Brian Kernighan trick used for popcount
