---
tags:
  - C
  - embedded
---

## [141. Linked List Cycle](https://leetcode.com/problems/linked-list-cycle/)

### Key Idea: Floyd's Cycle Detection (Fast & Slow Pointers) 

  Use two pointers starting at head:
  - slow moves 1 step at a time
  - fast moves 2 steps at a time

  If no cycle: fast hits NULL → return false
  If cycle exists: fast laps slow inside the cycle — they must meet

  Once both pointers are inside the cycle, look at the gap between them each step:

  Each iteration: fast gains 1 step on slow
  gap shrinks by 1 every iteration → eventually gap = 0 → they meet

  They can never "skip over" each other because the gap decreases by exactly 1 each step.

### Solution - O(n)
  
```c
  while (fast && fast->next) {   // fast hits NULL → no cycle
      slow = slow->next;         // +1
      fast = fast->next->next;   // +2
      if (slow == fast)          // met → cycle found
          return true;
  }
  return false;
```
  
  The check fast && fast->next guards against null dereference on fast->next->next.

---

## [160.Intersection of Two Linked Lists](https://leetcode.com/problems/intersection-of-two-linked-lists/)

## KEY IDEA: total distance each = lenA + lenB

  The cross-redirect guarantees both pointers travel the same total distance:

  skipA path: ──[A's nodes]──[B's nodes]──► intersection
  skipB path: ──[B's nodes]──[A's nodes]──► intersection


> KEY POINT: 兩邊加起來 traverse 總距離一樣

  They must meet at the intersection, or both hit NULL at the same time (no intersection case).
  
### [Solution - O(m+n)](https://github.com/JackyYin/algorithms/blob/master/leetcode/160_intersection_of_two_linked_list.c)

  ---

## [148. Sort List](https://leetcode.com/problems/sort-list)

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


### Complexity

| Time  | `O(n log n)` — log n levels, O(n) merge work per level |
| ----- | ------------------------------------------------------ |
| Space | `O(log n)`   — recursion stack depth                   |
  
---


Other LeetCode:
- [83. Remove Duplicates from Sorted List](https://leetcode.com/problems/remove-duplicates-from-sorted-list)
- [merge two sorted lists](https://leetcode.com/problems/merge-two-sorted-lists)
- [intersection of two linked lists](https://leetcode.com/problems/intersection-of-two-linked-lists)

Refs:
- [jserv](https://hackmd.io/@sysprog/c-linked-list)