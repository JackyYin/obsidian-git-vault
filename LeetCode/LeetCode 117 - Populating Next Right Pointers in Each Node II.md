## Problem

Set each node's `next` to its right sibling on the same level (or `NULL` if none). The tree is **not** perfect — any child can be missing.

**Constraint that makes it interesting:** O(1) extra space. So no BFS queue. The traversal must lean on structure already built.

---

## Key insight

> Once level `L` has its `next` pointers set, you can **traverse level `L` left-to-right via those `next` pointers** — and use that traversal to set up level `L+1`'s `next` pointers.

Levels are built one at a time, and each completed level becomes the scaffolding for the next.

```
Level L (already linked):  A ──→ B ──→ C ──→ D ──→ NULL
                          /\    /     /\    /
                         a  b  c     e  f  g
Level L+1 (being linked): a ──→ b ──→ c ──→ e ──→ f ──→ g ──→ NULL
                          ↑                                 ↑
                          leftmost                          last node
```

While sweeping level `L`, we maintain two pieces of state for level `L+1`:

1. **`leftmost`** — the first non-null child encountered on level `L+1`. This becomes the entry point for the next outer iteration.
2. **`prev`** — the most recently linked child on level `L+1`. Each new child gets `prev->next = child`, then `prev` advances.

---

## The `process` helper (pointer-to-pointer trick)

```cpp
void process(Node *child, Node **prev, Node **leftmost) {
    if (child) {
        if (*prev)
            (*prev)->next = child;     // chain to previous child
        else
            *leftmost = child;          // first child seen on this level

        *prev = child;                  // advance prev
    }
}
```

The two `**` parameters let the helper mutate the caller's `prev` and `leftmost` (C++ passes by value). The helper encapsulates the "first child vs subsequent child" branch in one place — both `cur->left` and `cur->right` go through it.

```cpp
Node* connect(Node* root) {
    Node *leftmost = root;
    Node *prev;
    while (leftmost) {                      // outer: per level
        Node *cur = leftmost;
        leftmost = NULL;
        prev = NULL;
        while (cur) {                       // inner: walk current level
            process(cur->left,  &prev, &leftmost);
            process(cur->right, &prev, &leftmost);
            cur = cur->next;                // ← uses next built last round
        }
    }
    return root;
}
```

Note `cur = cur->next` — that's where we lean on last round's work. On level 0, `leftmost = root` and `root->next` is already `NULL`, so the inner loop runs once and bootstraps level 1.

---

## Walk-through

Tree (note the missing children — this is what makes LC 117 ≠ LC 116):

```
              1
            /   \
           2     3
          / \     \
         4   5     7
```

### Round 1 — sweeping level 0, building level 1

```
state at start:  leftmost = 1
                 prev = NULL,  cur = 1

cur = 1:
  process(1->left = 2):  prev was NULL → leftmost := 2,  prev := 2
  process(1->right = 3): prev was 2    → 2->next = 3,    prev := 3
  cur = 1->next = NULL  → exit inner
```

After round 1:

```
              1 ──→ NULL
            /   \
           2 ──→ 3
          / \     \
         4   5     7

leftmost (next round) = 2
```

### Round 2 — sweeping level 1, building level 2

```
state at start:  leftmost = 2 → reset to NULL
                 prev = NULL,  cur = 2

cur = 2:
  process(2->left = 4):  prev was NULL → leftmost := 4,  prev := 4
  process(2->right = 5): prev was 4    → 4->next = 5,    prev := 5
  cur = 2->next = 3                                  ← uses round 1's link!

cur = 3:
  process(3->left = NULL):  skip
  process(3->right = 7):    prev was 5 → 5->next = 7,    prev := 7
  cur = 3->next = NULL  → exit inner
```

When we reached node 3, its `left` was missing — `process` simply did nothing, and `prev` stayed at 5. Then 3's right child (7) chained directly off 5, **even though they have different parents**. That's exactly what level-order requires.

After round 2:

```
              1 ──→ NULL
            /   \
           2 ──→ 3 ──→ NULL
          / \     \
         4 → 5 ───→ 7

leftmost (next round) = 4
```

### Round 3 — sweeping level 2, building level 3 (which is empty)

```
cur = 4: no children, prev/leftmost untouched, cur = 4->next = 5
cur = 5: no children,                          cur = 5->next = 7
cur = 7: no children,                          cur = 7->next = NULL
```

`leftmost` was never assigned (stays NULL). Outer loop exits. Done.

---

## Why this is elegant

**Each node is visited exactly twice:**
- Once as `cur` in the outer-level sweep (to inspect its children).
- Once as a `child` passed to `process` (to be chained or made the new leftmost).

So the algorithm is **O(n) time, O(1) auxiliary space** — beating the obvious BFS-with-queue solution that needs O(width) space.

The `process` helper also cleanly handles the "skip missing child" case by short-circuiting on `if (child)` — no sentinel, no special-casing the first child of each level inline. The pointer-to-pointer arguments let it own the bookkeeping for both `prev` and `leftmost` without the caller having to inspect them.

---

## Key takeaways

1. **Reuse the structure you just built.** The `next` pointers on level `L` are the iterator over level `L` while you build `next` on level `L+1`. No queue needed.
2. **Two pieces of state per level**: `leftmost` (where the next sweep starts) and `prev` (the chain pointer being extended).
3. **`Node**` parameters** let a helper update the caller's local pointers — clean way to factor out the "first-child vs subsequent-child" branch.
4. **Cross-parent chaining** falls out naturally: the helper doesn't care whose child it just received; it only cares about `prev`. Missing children are handled by a single null-check.
