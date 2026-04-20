---
tags:
  - C
  - linked-list
  - leetcode
---

# [148. Sort List](https://leetcode.com/problems/sort-list)

### Key Idea 1: Find the Midpoint with Fast/Slow Pointers (`split`)

  To divide the list in half without knowing its length:

  ```
  slow moves +1, fast moves +2 → when fast hits end, slow is at the middle
  ```

  `prev` tracks one step behind `slow` so you can sever the list:

  ```
  1→2→3→4→5→∅

  slow stops at 3, prev at 2  →  prev->next = NULL

  left:  1→2→∅
  right: 3→4→5→∅
  ```

  ---

### Key Idea 2: Base Case via `left == right`

  After `split(head)`, if `right == head` the list had only 1 node — nothing left to divide.

  ```c
  if (left == right)
      return left;
  ```

  ---

### Key Idea 3: Merge Two Sorted Lists (LC 21's pointer-to-pointer trick)

  ```
  ptr  → open slot in the merged list
  node → &l or &r (whichever list to pull from)

  *ptr = *node          — fill the slot
  ptr  = &(*node)->next — advance ptr to next open slot
  *node = (*node)->next — advance the chosen list
  ```

  Then attach whichever tail remains (`l` or `r`).

---
### Full Picture

  ```
  sortList([4,2,1,3])
           │
      split: [4,2] / [1,3]
           │
      split: [4]/[2]    [1]/[3]
           │
     merge: [2,4]      [1,3]
           │
        merge: [1,2,3,4]
  ```

---

### Complexity

| Time  | `O(n log n)` — log n levels, O(n) merge work per level |
| ----- | ------------------------------------------------------ |
| Space | `O(log n)`   — recursion stack depth                   |
