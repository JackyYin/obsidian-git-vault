---
tags:
  - leetcode
  - C
  - monotonic_stack
---

# LeetCode 901 — Online Stock Span

## Problem

Design a class `StockSpanner` that, for each new daily `price`, returns the **span**: the number of consecutive days (ending today, going backwards) where the price is **≤ today's price**. Stop at the first strictly larger day.

## Implementation

```cpp
class StockSpanner {
public:
    stack<pair<int,int>> stk;
    StockSpanner() {}

    int next(int price) {
        int cnt = 1;
        while (!stk.empty()) {
            auto p = stk.top();
            if (price < p.first) {
                break;
            }
            cnt += p.second;
            stk.pop();
        }
        stk.push({ price, cnt });
        return cnt;
    }
};
```

## Naïve approach and why it's wasteful

Walk backward through history while `prices[j] ≤ today`. **O(n²)** overall, and most of those comparisons are redundant: any past day shorter than today's price will be absorbed by today and **never matter again** for future queries (future queries first hit today, which already dominates them).

## The key insight

> Once a past day is dominated, we only ever need to know: "what's the next strictly greater day to the left, and how many days does it dominate?"

So we **collapse** dominated runs into a single entry: `(price, span)`. The `span` is "this price plus everything it already absorbed."

## The stack invariant

Reading bottom → top, prices are **strictly decreasing**. The top of the stack is always the most recent bucket — the one today must compare against first.

## What `next(price)` does

```
cnt = 1                          // today counts itself
while stack not empty:
    (p, s) = top
    if price < p: stop           // p is the strictly greater wall — done
    cnt += s                     // absorb the entire bucket
    pop
push (price, cnt)
return cnt
```

Each popped bucket says: "I'm dominated by today, so transfer my span to today."

## Trace: `[100, 80, 60, 70, 60, 75, 85]`

```
next(100):  stack empty → push (100,1)         span = 1
            stack: [(100,1)]

next(80):   top=100 > 80, stop → push (80,1)   span = 1
            stack: [(100,1), (80,1)]

next(60):   top=80 > 60, stop → push (60,1)    span = 1
            stack: [(100,1), (80,1), (60,1)]

next(70):   top=60 ≤ 70 → cnt=1+1=2, pop
            top=80 > 70, stop → push (70,2)    span = 2
            stack: [(100,1), (80,1), (70,2)]
                                    ↑ "70 absorbed yesterday's 60"

next(60):   top=70 > 60, stop → push (60,1)    span = 1
            stack: [(100,1), (80,1), (70,2), (60,1)]

next(75):   top=60 ≤ 75 → cnt=2, pop
            top=70 ≤ 75 → cnt=2+2=4, pop      (4 = 75 + 60 + 70 + 60)
            top=80 > 75, stop → push (75,4)    span = 4
            stack: [(100,1), (80,1), (75,4)]

next(85):   top=75 ≤ 85 → cnt=1+4=5, pop
            top=80 ≤ 85 → cnt=5+1=6, pop
            top=100 > 85, stop → push (85,6)   span = 6
            stack: [(100,1), (85,6)]

returns:  1, 1, 1, 2, 1, 4, 6  ✓
```

## Picture: the bucket idea

After `next(75)`, the stack looks like:

```
bottom                                    top
┌─────────┐  ┌────────┐  ┌─────────────────────┐
│ 100, 1  │  │ 80, 1  │  │ 75, span=4          │
└─────────┘  └────────┘  └─────────────────────┘
   day 0       day 1     day 5 (absorbs days 2,3,4)
```

The `(75, 4)` bucket means: "starting at day 5 going back, 75 dominates 4 consecutive days (itself + days 4, 3, 2)." We don't need to store days 2/3/4 individually — their info is **fused** into this bucket.

## Why the amortized cost is O(1) per call

Every price is **pushed once** and **popped at most once** across the lifetime of the spanner. Total work over n calls is O(n), so **amortized O(1) per `next`** — even though a single call can pop many buckets.

## Mental model

Think of the stack as a chain of **left-going walls**, each shorter than the one before it. A new price either:

- Hits a taller wall immediately → span = 1, becomes a new short wall.
- Knocks down one or more shorter walls, **inheriting their reach**, until it hits a taller wall — and then becomes the new top.

## Complexity

- **Time:** amortized O(1) per `next` call; O(n) total over n calls.
- **Space:** O(n) worst case (strictly decreasing input keeps everything on the stack).

## Pattern

This is the **monotonic stack** pattern, common to problems like:

- Next Greater Element (LC 496, 503)
- Largest Rectangle in Histogram (LC 84)
- Daily Temperatures (LC 739)
- Trapping Rain Water (LC 42)

The shared idea: maintain a stack whose contents satisfy a monotonic property; each element is pushed and popped at most once → amortized O(1).
