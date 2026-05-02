---
tags:
  - linked-list
  - leetcode
  - C
---

# LeetCode 82 — Remove Duplicates from Sorted List II

## Problem

Given a sorted linked list, delete **all** nodes that have duplicate numbers, leaving only distinct numbers from the original list.

Example: `1 → 2 → 3 → 3 → 4 → 4 → 5` becomes `1 → 2 → 5`.

## Implementation

```cpp
class Solution {
public:
    ListNode* deleteDuplicates(ListNode* head) {
        ListNode* cur = head;
        ListNode** ptr = &head;

        while (cur) {
            bool shifted = 0;
            while (cur && cur->next && cur->val == cur->next->val) {
                shifted = 1;
                cur = cur->next;
            }

            if (shifted) {
                *ptr = cur->next;
            }
            else {
                ptr = &(*ptr)->next;
            }
            cur = cur->next;
        }
        return head;
    }
};
```

## The two roles

- **`cur`** — the scanner. Walks through the list one node at a time.
- **`ptr`** — a *pointer to a pointer*. It points at whichever `next` field would need to be rewritten if the upcoming run turns out to be a duplicate. Starts as `&head` because the very first kept node might *be* the head.

## Each outer-loop iteration asks one question

> "Starting at `cur`, is `cur->val` part of a duplicate run?"

The **inner while** scans forward as long as `cur->val == cur->next->val`. After it exits, `cur` sits on the **last** node of the run (or just on itself if there was no run).

Then:

| `shifted` | meaning | action |
|---|---|---|
| `true` | run of duplicates found | `*ptr = cur->next` — splice the whole run out |
| `false` | `cur` is unique, keep it | `ptr = &(*ptr)->next` — advance the "rewrite slot" |

Finally `cur = cur->next` moves past the run / past the kept node.

## Trace: `1 → 2 → 3 → 3 → 4 → 4 → 5`

```
Start:    head → [1] → [2] → [3] → [3] → [4] → [4] → [5]
          ptr=&head, cur=[1]

iter 1: [1] unique. ptr advances to &[1].next, cur=[2]
          head → [1] → [2] → [3] → [3] → [4] → [4] → [5]
                  ↑ptr points at this arrow

iter 2: [2] unique. ptr advances to &[2].next, cur=[3]
          head → [1] → [2] → [3] → [3] → [4] → [4] → [5]
                        ↑ptr

iter 3: inner loop: 3==3, cur slides to second [3]. shifted=1.
        *ptr = cur->next = [4]   ← rewrites [2].next
          head → [1] → [2] ─────────────→ [4] → [4] → [5]
                        ↑ptr (still here)
        cur = cur->next = [4]

iter 4: inner loop: 4==4, cur slides to second [4]. shifted=1.
        *ptr = cur->next = [5]   ← rewrites [2].next AGAIN
          head → [1] → [2] ───────────────────────→ [5]
                        ↑ptr
        cur = [5]

iter 5: [5] unique (next is null). ptr advances. cur=null. exit.

Result: 1 → 2 → 5 ✓
```

## Why `ptr` is a `ListNode**` (the key insight)

When we find duplicates, we need to rewrite the `next` field of the **last kept node** — but at the moment of rewriting, we no longer hold a reference to that node directly. By keeping `ptr` as the *address of that next field*, `*ptr = ...` rewrites the right pointer regardless of whether it's `head` itself or some interior `node->next`.

### Edge case: duplicates at the head — `1 → 1 → 1 → 2`

- iter 1: run of 1s detected, `*ptr = [2]` — but `ptr == &head`, so **`head` itself becomes `[2]`**.

Same code path, no special case needed. That's the win over the typical dummy-node approach.

## Complexity

- **Time:** O(n) — each node visited once.
- **Space:** O(1) — no dummy node, no auxiliary structures.

## Related

- [[LeetCode 1836 - Remove Duplicates From an Unsorted Linked List]]
- [[Linked List]]
