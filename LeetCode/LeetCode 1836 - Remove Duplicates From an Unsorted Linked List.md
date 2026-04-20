---
tags:
  - C
  - linked-list
  - leetcode
---

# [1836. Remove Duplicates From an Unsorted Linked List](https://leetcode.com/problems/remove-duplicates-from-an-unsorted-linked-list/)

### Key Idea : Delete by Frequency with a Pointer-to-Pointer (Pass 2)

  Use `ListNode **ptr` pointing at the current "slot" rather than the node itself.
  This lets you splice out a node without needing a separate `prev` pointer:

  ```
  freq[val] > 1  →  skip: *ptr = (*ptr)->next   (ptr stays, node is unlinked)
  freq[val] == 1 →  keep: ptr = &(*ptr)->next   (ptr advances forward)
  ```

  Walk through example:

  ```
  ptr→[1]→[2]→[3]→[2]→[1]→∅

  freq[1]=2 → skip:  ptr→[2]→[3]→[2]→[1]→∅
  freq[2]=2 → skip:  ptr→[3]→[2]→[1]→∅
  freq[3]=1 → keep:  [3]→... , ptr→[2]→[1]→∅
  freq[2]=2 → skip:  ptr→[1]→∅
  freq[1]=2 → skip:  ptr→∅

  result: [3]→∅
  ```

  ---

### Why pointer-to-pointer instead of `prev`?

  A `prev` pointer approach requires special-casing the head node.
  `ListNode **ptr = &head` treats `head` as just another slot — no special case needed.

| Operation             | Effect                                        |
| --------------------- | --------------------------------------------- |
| `*ptr = (*ptr)->next` | unlinks current node (ptr stays at same slot) |
| `ptr = &(*ptr)->next` | advances ptr to next slot                     |

  ---

### Complexity

| Time  | `O(n)` — two linear passes                           |
| ----- | ---------------------------------------------------- |
| Space | `O(1)` — fixed-size array, not proportional to input |
