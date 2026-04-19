---
tags:
  - C
  - embedded
---

 ---
 ## [160. Intersection of Two Linked Lists](https://leetcode.com/problems/intersection-of-two-linked-lists/)


Solution - O(m+n)

  The cross-redirect guarantees both pointers travel the same total distance:

  skipA path: ──[A's nodes]──[B's nodes]──► intersection
  skipB path: ──[B's nodes]──[A's nodes]──► intersection

 #### **total distance each = lenA + lenB**

  They must meet at the intersection, or both hit NULL at the same time (no intersection case).

  ---



LeetCode:
- [merge two sorted lists](https://leetcode.com/problems/merge-two-sorted-lists)
- [intersection of two linked lists](https://leetcode.com/problems/intersection-of-two-linked-lists)

Refs:
- [jserv](https://hackmd.io/@sysprog/c-linked-list)