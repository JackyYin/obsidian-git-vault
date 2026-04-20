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

Other Detailed Solutions:
- [[LeetCode 148 - Sort List]]
- [[LeetCode 1836 - Remove Duplicates From an Unsorted Linked List]]

Other LeetCode Problems:
- [83. Remove Duplicates from Sorted List](https://leetcode.com/problems/remove-duplicates-from-sorted-list)
- [merge two sorted lists](https://leetcode.com/problems/merge-two-sorted-lists)
- [intersection of two linked lists](https://leetcode.com/problems/intersection-of-two-linked-lists)

Refs:
- [jserv](https://hackmd.io/@sysprog/c-linked-list)
