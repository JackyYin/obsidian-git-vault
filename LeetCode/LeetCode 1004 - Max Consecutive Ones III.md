## Problem

Given a 0/1 array `nums` and a budget `k`, find the longest contiguous subarray you can produce if you may flip **at most `k` zeros** to ones.

Reframed: **find the longest window `[L, R]` containing at most `k` zeros.**

---

## Approach — Variable-size sliding window (two pointers)

Track an invariant: `kleft` = remaining flips allowed inside the current window.

```
kleft = k − (number of zeros in nums[L..R])
```

The window is **valid** iff `kleft ≥ 0`.

- **`R` extends greedily** by 1 each iteration. If `nums[R] == 0`, decrement `kleft`.
- If the window becomes invalid (`kleft < 0`), **shrink from the left**: walk `L` rightward; each `0` we step over restores one flip (`kleft++`).
- After the inner repair loop, `[L..R]` is the longest valid window ending at `R`. Update `ans`.

Both pointers move monotonically forward → each index is touched at most twice → **O(n)**.

### Why this is correct

**Monotone validity:** if `[L, R]` is valid, then `[L', R]` is valid for any `L' ≥ L`. So the optimal `L` for a given `R` is the smallest one keeping the window valid — exactly what the inner shrink loop converges to.

---

## Implementation

```cpp
class Solution {
public:
    int longestOnes(vector<int>& nums, int k) {
        int kleft = k;
        int n = nums.size();
        int left = 0, right = 0;
        int ans = 0;

        while (right < n) {
            if (nums[right] == 0)
                kleft--;

            while (kleft < 0) {
                if (nums[left] == 0)
                    kleft++;
                left++;
            }

            ans = max(ans, right - left + 1);
            right++;
        }
        return ans;
    }
};
```

---

## Step-by-step walk-through

Example: `nums = [1,1,1,0,0,0,1,1,1,1,0]`, `k = 2`

**Legend:** `L` = left pointer, `R` = right pointer, `═` = cells inside the current window.

### Frame 0 — Initial state

```
idx:      0  1  2  3  4  5  6  7  8  9 10
nums:     1  1  1  0  0  0  1  1  1  1  0
          ↑
         L,R   (loop about to start)

kleft=2,  ans=0
```

### Frame 1 — Iters 1–3: R sees three 1's, kleft unchanged

```
idx:      0  1  2  3  4  5  6  7  8  9 10
nums:     1  1  1  0  0  0  1  1  1  1  0
          L     R
          ═══════

kleft=2,  len=3,  ans=3
```

### Frame 2 — Iter 5 (R=4): two 0's eaten — budget exhausted but still valid

```
idx:      0  1  2  3  4  5  6  7  8  9 10
nums:     1  1  1  0  0  0  1  1  1  1  0
          L           R
          ═════════════

kleft=0,  len=5,  ans=5
```

This is the largest window so far. The next `0` will tip us over.

### Frame 3 — Iter 6, BEFORE shrink: R=5 is a 0, kleft = −1, INVALID

```
idx:      0  1  2  3  4  5  6  7  8  9 10
nums:     1  1  1  0  0  0  1  1  1  1  0
          L              R
          ════════════════
                                       ⚠ kleft = −1 → must shrink
```

### Shrink trace

The inner loop reads `nums[left]` *before* incrementing `left`:

```
iter | L (read)  nums[L] | effect              | after iter: L  kleft
─────────────────────────────────────────────────────────────────────
  1  |    0         1    | skip a 1, no gain   |             1    -1
  2  |    1         1    | skip a 1, no gain   |             2    -1
  3  |    2         1    | skip a 1, no gain   |             3    -1
  4  |    3         0    | recover one flip!   |             4     0   STOP
```

`L` ends up **just past** the zero it recovered (index 4). That zero is now outside the window; the other two zeros (indices 4, 5) remain inside, exactly fitting the budget.

### Frame 4 — Iter 6, AFTER shrink

```
idx:      0  1  2  3  4  5  6  7  8  9 10
nums:     1  1  1  0  0  0  1  1  1  1  0
                      L  R
                      ════

kleft=0,  len=2,  ans=5  (unchanged — shrunken window is small)
```

### Frame 5 — Iters 7–10: R sweeps four 1's, no shrink needed

```
idx:      0  1  2  3  4  5  6  7  8  9 10
nums:     1  1  1  0  0  0  1  1  1  1  0
                      L              R
                      ════════════════

kleft=0,  len=6,  ans=6   ← new best!
```

The window `[4..9]` contains exactly 2 zeros (indices 4, 5) and 4 ones — flipping the two zeros yields a run of 6.

### Frame 6 — Iter 11 (R=10): hits another 0, shrink slides L by 1

**Before shrink:**

```
idx:      0  1  2  3  4  5  6  7  8  9 10
nums:     1  1  1  0  0  0  1  1  1  1  0
                      L                 R
                      ═══════════════════
                                              ⚠ kleft = −1
```

`L=4` is a `0`, so a single iteration recovers a flip immediately.

**After shrink:**

```
idx:      0  1  2  3  4  5  6  7  8  9 10
nums:     1  1  1  0  0  0  1  1  1  1  0
                         L              R
                         ════════════════

kleft=0,  len=6,  ans=6   (tied, no improvement)
```

Window slid from `[4..9]` to `[5..10]`, same length.

`R++` → `R=11 ≥ n`, loop ends. **Return 6.**

---

## The "shape" of the algorithm

```
                R sweeps right →
   L stays put for a while (window grows)
   ────────────────────────────────────
   …then a 0 pushes kleft below 0…
   L jumps forward to repair        (L only goes →, never ←)
   ────────────────────────────────────
   …window grows again from the new L…
```

Both pointers march **monotonically right**:
- `R` makes 1 full pass over the array.
- `L` also makes at most 1 full pass (it never rewinds).

Total work: **O(n)** time, **O(1)** extra space.

---

## Sliding window vs two pointers — naming nit

People use the names interchangeably, but there's a useful distinction:

| flavor                     | window size           | inner loop                                          |
|----------------------------|-----------------------|-----------------------------------------------------|
| **fixed sliding window**   | constant `k`          | none — slide both pointers in lockstep              |
| **variable / two-pointer** | varies by invariant   | yes — `R` extends, `L` repairs when invariant breaks |

LC 1004 is the **variable** kind: window size depends on the `kleft ≥ 0` invariant, with `L` and `R` moving asynchronously but both monotonically forward.

---

## Key takeaways

1. **Reframe the problem** — "flip ≤ k zeros to maximize a run of 1's" becomes "longest window with ≤ k zeros."
2. **Track an invariant** rather than recomputing from scratch — `kleft` deltas as `R` enters or `L` exits.
3. **Monotone pointers** give O(n) by amortization: each index is processed at most twice.
4. **The shrink loop is `while`, not `if`** — in general you may need many shrink steps, though here at most one zero is uncovered before the invariant is restored.
