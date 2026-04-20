---
tags:
  - C
  - linked-list
  - leetcode
---

# LeetCode 234 — Palindrome Linked List

  **Problem:** Check if a singly linked list is a palindrome in `O(n)` time and `O(1)` space.

  Your solution uses a **3-step approach**: Split → Reverse → Compare.

  ---

### Key Idea 1: Find the Midpoint and Split (`split`)

  Same fast/slow pointer trick as LC 148.
  `prev` severs the first half cleanly by setting `prev->next = NULL`:

  ```
  1→2→2→1→∅

  slow stops at 2 (second), prev at 2 (first)
  prev->next = NULL

  left:  1→2→∅
  right: 2→1→∅
  ```

  For odd-length lists:

  ```
  1→2→3→2→1→∅

  left:  1→2→∅    (shorter)
  right: 3→2→1→∅
  ```

  The extra middle node lands in `right` — it doesn't affect the comparison
  since the `while (head && right)` loop stops when either pointer hits `∅`.

  ---

### Key Idea 2: Reverse the Second Half (`reverse`)

  Standard in-place linked list reversal using a trailing `ptr`:

  ```
  2→1→∅   becomes   1→2→∅

  step 1: ptr=∅,  node=2,  next=1  →  2→∅,  ptr=2
  step 2: ptr=2,  node=1,  next=∅  →  1→2→∅, ptr=1
  ```

  This avoids any extra memory — no stack or array needed.

  ---

### Key Idea 3: Compare Both Halves

  Walk both halves simultaneously and compare values:

  ```
  left:          1→2→∅
  right reversed: 1→2→∅

  1==1 ✓   2==2 ✓   both hit ∅ → true
  ```

  If any values differ → `false`.

  ---

### Full Picture

  ```
  Original:   1→2→2→1→∅
                  │
              split()
                  │
           1→2→∅  │  2→1→∅
                  │
             reverse() right
                  │
           1→2→∅  │  1→2→∅
                  │
             compare
                  │
                true ✓
  ```

  ---

### Complexity

| Time  | `O(n)` — one pass each for split, reverse, compare |
| ----- | -------------------------------------------------- |
| Space | `O(1)` — all in-place, no extra data structures    |
