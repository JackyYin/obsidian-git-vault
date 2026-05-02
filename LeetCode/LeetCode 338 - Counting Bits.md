---
tags:
  - leetcode
  - C
  - bits_operation
---

# LeetCode 338 — Counting Bits

## Problem

Given an integer `n`, return an array `ans` of length `n + 1` where `ans[i]` is the number of `1`s in the binary representation of `i`, for every `i` in `[0, n]`.

**Goal:** O(n) time, O(n) space (the output itself).

## Key Idea

**Reuse a previously computed answer for a strictly smaller number.**

The popcount of `i` differs from the popcount of some smaller `j` by exactly **one bit**, so we can build the table in a single forward pass:

```
bits[0] = 0
bits[i] = bits[smaller_i] + (0 or 1)
```

Two clean recurrences express this:

### A. Drop the lowest set bit (cleanest)

```cpp
bits[i] = bits[i & (i - 1)] + 1;
```

- `i & (i - 1)` clears `i`'s **lowest 1-bit**.
- So the subproblem has exactly one fewer `1`. Add it back: `+ 1`.

Example: `i = 12 = 1100`, `i & (i-1) = 1000 = 8`. `bits[12] = bits[8] + 1`.

### B. Drop the lowest bit via shift

```cpp
bits[i] = bits[i >> 1] + (i & 1);
```

- `i >> 1` removes `i`'s lowest bit (whatever it was).
- `i & 1` re-adds it if it was a `1`.

Example: `i = 13 = 1101`, `i >> 1 = 110 = 6`, `i & 1 = 1`. `bits[13] = bits[6] + 1`.

## Implementation

```cpp
class Solution {
public:
    vector<int> countBits(int n) {
        vector<int> bits(n + 1, 0);
        for (int i = 1; i <= n; ++i) {
            bits[i] = bits[i & (i - 1)] + 1;
            // or: bits[i] = bits[i >> 1] + (i & 1);
        }
        return bits;
    }
};
```

## Why O(n)

Each `bits[i]` is computed in **O(1)** by referencing one already-filled slot. No inner loop, no `popcount` per number, no per-bit iteration.

The naive approach (count bits for each number independently) is O(n log n).

## Complexity

- **Time:** O(n)
- **Space:** O(n) for the output (no auxiliary structures)

## Related

- [[Bits Operation]]
