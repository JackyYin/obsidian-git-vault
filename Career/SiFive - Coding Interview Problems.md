---
tags:
  - career
  - interview
  - coding
  - RISC-V
  - embedded-C
  - linux-kernel
interview_date: '2026-05-08'
---

# SiFive Coding Interview — Curated Problems (2026-05-08)

> Coding round for **System Software Engineer (Linux)**.
> Past candidate reports vary: some had **no coding test**, others say **"fundamentals of coding + your approach to a problem"**. Plan as if a real coding round happens, Zoom screen-share, language of choice (recommend **C**, fallback Python).
> SiFive software-engineer interviews are rated the **hardest tier** (alongside CPU DV) — Glassdoor 3.13/5.

---

## Tier 0 — Always-Asked Embedded C Fundamentals  ⭐⭐⭐⭐⭐

These are guaranteed to surface, either as standalone questions or inside a larger problem. Source: Nigel Jones' famous *"0x10 Best Questions for Would-be Embedded Programmers"* — referenced as the de-facto C test even today (RMB Consulting; embeddedrelated.com).

| #   | Problem                                                                                  | Why it matters at SiFive                            |
| --- | ---------------------------------------------------------------------------------------- | --------------------------------------------------- |
| 1   | Write `#define SECONDS_PER_YEAR` (catch the 16-bit overflow → use UL).                   | Kernel/firmware code runs on word sizes you choose. |
| 2   | Write the `MIN(a,b)` macro (parenthesize args + ternary).                                | Side-effect trap: `MIN(*p++, b)`.                   |
| 3   | Declare: pointer to fn returning int; array of 10 such pointers.                         | Driver `file_operations` style.                     |
| 4   | Three uses of `static` (function scope, file scope, fn linkage).                         | Kernel modules use all three.                       |
| 5   | Five uses of `const` (`const int *`, `int * const`, etc.).                               | Kernel APIs are `const`-heavy.                      |
| 6   | What is `volatile`? Can a var be both `const` and `volatile`?                            | MMIO + ISR — RISC-V CSR access.                     |
| 7   | Why `int square(volatile int *p)` is buggy.                                              | Kernel ISR pitfall.                                 |
| 8   | Set / clear bit 3 of an int (use `1<<3`, `&= ~`).                                        | Register manipulation, every driver.                |
| 9   | Write value to an absolute address `0x67a9`.                                             | Memory-mapped I/O.                                  |
| 10  | What's wrong with this ISR: returns a value, takes a parameter, calls `printf`, uses FP? | Kernel ISR rules.                                   |
| 11  | `unsigned a=6; int b=-20; (a+b>6)?` — integer promotion gotcha.                          | Common bug source in drivers.                       |
| 12  | `unsigned compzero = 0xFFFF;` — why `~0` is correct.                                     | Word-size portability.                              |
| 13  | `malloc(0)` — what does it return? (implementation defined).                             | Tests reasoning under ambiguity.                    |

**Source:** [RMB Consulting — A 'C' Test: 0x10 Best Questions](https://rmbconsulting.us/publications/a-c-test-the-0x10-best-questions-for-would-be-embedded-programmers/)

---

## Tier 1 — Reimplement Standard Library  ⭐⭐⭐⭐⭐

Classic SiFive-style "show me you understand pointers + edge cases."

| Problem                                                  | Edge cases to call out                               |
| -------------------------------------------------------- | ---------------------------------------------------- |
| `void *memcpy(void *dst, const void *src, size_t n)`     | overlap → undefined; alignment; word-at-a-time copy. |
| `void *memmove(void *dst, const void *src, size_t n)`    | handle overlap (forward vs backward).                |
| `size_t strlen(const char *s)`                           | NULL? UB. Talk about word-at-a-time scan.            |
| `char *strcpy(char *dst, const char *src)` and `strncpy` | strncpy doesn't always null-terminate (gotcha).      |
| `int strcmp(const char *a, const char *b)`               | sign of return value, unsigned char compare.         |
| `int atoi(const char *s)`                                | leading whitespace, sign, INT_MAX/INT_MIN overflow.  |
| `void *memset(void *s, int c, size_t n)`                 | `c` cast to unsigned char; word-at-a-time.           |
| `int itoa(int n, char *buf, int base)`                   | INT_MIN, base 2/10/16, reverse order.                |

**Be ready to optimize:** word-aligned bulk copy → tail bytes. Mention `restrict` keyword.

---

## Tier 2 — Bit Manipulation  ⭐⭐⭐⭐⭐

Highest signal for embedded/kernel roles. Per Manasi Rajan's *"Cracking the (embedded) Coding Interview"*: focus on **arrays, strings, and bit manipulation** — that's the embedded subset of leetcode.

| Problem                                                            | Hint                                                      |
| ------------------------------------------------------------------ | --------------------------------------------------------- |
| Count set bits in a 32-bit int (3 ways: naive, Kernighan, lookup). | `n & (n-1)` clears lowest set bit.                        |
| Reverse bits of a 32-bit int.                                      | bit-by-bit, divide-and-conquer (4 lines).                 |
| Check if a number is a power of two.                               | `n && !(n & (n-1))`.                                      |
| Round up to next power of two.                                     | shift-OR cascade (used in kernel for buffer sizing).      |
| Find the only non-duplicate in array of pairs.                     | XOR all elements.                                         |
| Swap two ints without temp.                                        | `a ^= b; b ^= a; a ^= b;` (mention real-code is worse).   |
| Extract bits `[hi:lo]` from an integer.                            | `(x >> lo) & ((1u << (hi-lo+1)) - 1)` — RISC-V CSR style. |
| Endian swap a 32-bit value (`__builtin_bswap32` mental model).     | Network drivers, device tree.                             |
| Detect endianness of host at runtime.                              | `union { uint32_t i; uint8_t b[4]; }`.                    |
| Pack/unpack a struct into a byte stream.                           | Discuss alignment, padding, `__attribute__((packed))`.    |


---

## Tier 3 — Kernel-Flavored Data Structures  ⭐⭐⭐⭐

These are arrays/strings/lists problems but framed in kernel idioms — your wheelhouse.

| Problem                                                                 | Kernel relevance                           |
| ----------------------------------------------------------------------- | ------------------------------------------ |
| Implement intrusive doubly-linked list (`struct list_head`).            | This is *the* Linux kernel pattern.        |
| Implement `container_of(ptr, type, member)` macro.                      | Cite `offsetof` + pointer arithmetic.      |
| Reverse a singly-linked list (iterative + recursive).                   | Pointer manipulation 101.                  |
| Detect a cycle in a linked list (Floyd's tortoise-and-hare).            | Memory-leak / loop detection mental model. |
| Find middle node in one pass.                                           | Two-pointer technique.                     |
| Merge two sorted linked lists.                                          | Merge-sort building block.                 |
| LRU cache (dlist + hashmap, O(1) ops).                                  | Page cache / TLB conceptual mapping.       |
| Implement a ring buffer (single-producer / single-consumer, lock-free). | kfifo, UART RX, kernel printk buffer.      |
| Implement a hash table with open addressing.                            | Compare to kernel's `hlist`.               |
| Binary search in a sorted array (bug-free — half the candidates fail).  | `lo + (hi-lo)/2` to avoid overflow.        |

---

## Tier 4 — Concurrency & Synchronization (Likely for Senior Role)  ⭐⭐⭐⭐

You're applying as **Senior Staff** track — they will probe locking.

| Problem                                                                                              | Concept                                              |
| ---------------------------------------------------------------------------------------------------- | ---------------------------------------------------- |
| Implement a spinlock from CAS / `atomic_xchg`.                                                       | Then ticket lock, then queued spinlock — evolution.  |
| Why must a spinlock disable preemption / interrupts?                                                 | Ties back to your [[CPU Control Hazard]] note.       |
| Reader/writer lock vs RCU — when to choose which.                                                    | RCU = read-heavy, no waiters block readers.          |
| Implement a thread-safe SPSC queue without locks.                                                    | Memory barriers — RVWMO `FENCE` instruction.         |
| Producer-consumer with bounded buffer (semaphore + mutex, or condition variable).                    | Classic.                                             |
| Dining philosophers — explain deadlock and resolution.                                               | Lock-ordering argument.                              |
| Why does `double-checked locking` need a memory barrier?                                             | Ties to RVWMO and `smp_mb()`.                        |

**Cross-ref your existing prep:** [[SiFive - Computer Architecture Prep]] §5 RVWMO + §4 MESI.

---

## Tier 5 — Algorithms with Embedded Context  ⭐⭐⭐

Less likely than Tier 0–2, but possible if interviewer is from US team.

| Problem                                                  | Twist                                                    |
| -------------------------------------------------------- | -------------------------------------------------------- |
| Two-sum on a sorted array.                               | Two-pointer, no extra space.                             |
| Sliding window maximum.                                  | Monotonic deque.                                         |
| Longest substring without repeating characters.          | Sliding window + hash.                                   |
| Find Kth largest in stream.                              | Min-heap of size K.                                      |
| Rotate an array in place by k.                           | 3-reverse trick — minimizes memory.                      |
| Spiral traversal of matrix.                              | Layer-by-layer with bounds.                              |
| Validate a balanced bracket string.                      | Stack.                                                   |

> Skip trees, graphs, DP unless time permits — embedded interviews rarely go there (per Manasi Rajan).

---

## Tier 6 — System / Driver Coding Scenarios  ⭐⭐⭐⭐ (Senior-bait)

These bridge "coding" and "design" — high signal for the seniority they want.

| Problem                                                                                          | What they're probing                                              |
| ------------------------------------------------------------------------------------------------ | ----------------------------------------------------------------- |
| Sketch a `platform_driver`'s `probe()` and `remove()` for a memory-mapped UART.                  | `ioremap`, `request_irq`, `devm_*` lifecycle.                     |
| Write the IRQ top-half + bottom-half (tasklet / threaded IRQ) for a packet-receive device.       | Latency, reentrancy, `IRQF_SHARED`.                               |
| Implement `read()` for a char device that returns one byte from a ring buffer (block if empty).  | `wait_event_interruptible`, `copy_to_user`.                       |
| Walk a `uint64_t` virtual address through Sv39 page tables.                                      | Reuses your [[SiFive - Computer Architecture Prep]] §6.           |
| Given two cores updating a shared counter, where can it go wrong on RISC-V vs x86?               | RVWMO vs TSO, where to put `FENCE` / `smp_mb()`.                  |
| Detect false sharing in a struct; how do you fix it?                                             | `____cacheline_aligned_in_smp`.                                   |
| Implement `kmalloc`-vs-`vmalloc` decision logic.                                                 | Cross-ref [[kmalloc() vs. vmalloc()]].                            |

---

## Tier 7 — Behavioral-Coding Hybrid  ⭐⭐⭐

SiFive cares about *"open, honest, direct communication."* A coding round here is also a culture round. Practice narrating:

- "Walk me through your debugging of a kernel panic." → frame your **ORCA crash-dump RCA** story.
- "How did you port this to s390x?" → endianness, atomic semantics, page size.
- "Have you submitted patches upstream?" → be honest if no; show you understand the workflow (`git format-patch`, LKML, maintainer trees).

---

## Day-Of Strategy

1. **Clarify before you code.** Restate the problem, ask about input range, ask about NULLs, ask about thread-safety. SiFive explicitly values "approach to a problem".
2. **Talk while you type.** Silent coding = bad signal.
3. **Use C if it's a low-level problem.** Use Python only for pure-algo problems (faster, less syntax tax).
4. **Always do at least one walk-through with a sample input** before declaring done.
5. **Test edge cases out loud:** NULL, empty, single element, overflow, max value, alignment.
6. **Discuss complexity** in time AND **memory** (embedded mindset).
7. **If stuck:** describe the brute-force first, then optimize. They'd rather see a working O(n²) than a broken O(n).

---

## Evidence — Sources Consulted

| Source                                                            | What it gave us                                              | Link                                                                                                                    |
| ----------------------------------------------------------------- | ------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------- |
| Glassdoor — SiFive Software Engineer interviews                  | Difficulty 3.13/5; SW Engineer is hardest tier; 25-day cycle | https://www.glassdoor.com/Interview/SiFive-Software-Engineer-Interview-Questions-EI_IE1922671.0,6_KO7,24.htm             |
| PTT Soft_Job — SiFive面試 (2020,2021)                              | "**之前沒有 [coding test]**" ← past, may not apply now         | https://www.ptt.cc/bbs/Soft_Job/M.1660653546.A.EB3.html                                                                 |
| GoodJob — SiFive SW Engineer 面試經驗 (2020 + 2023)                  | "**沒有考coding test**" — Linux + work-experience focus        | https://www.goodjob.life/experiences/63dcdb3ac36210001278d82e ; https://www.goodjob.life/experiences/5e57ccb7400ede0012cf8524 |
| Hyperbola Blog — Shopee/Amazon/SiFive/Foodpanda                  | DevOps role: OS + shell + DB; difficulty moderate            | https://blog.hyperbola.me/posts/other-interviews/                                                                       |
| RMB Consulting — 0x10 Best Questions for Embedded Programmers    | The Tier-0 C question bank                                   | https://rmbconsulting.us/publications/a-c-test-the-0x10-best-questions-for-would-be-embedded-programmers/                |
| Manasi Rajan — Cracking the (embedded) Coding Interview          | "**arrays, strings, bit manipulation**" focus                | https://www.embeddedrelated.com/showarticle/1503.php                                                                    |
| theEmbeddedNewTestament (GitHub)                                 | Comprehensive embedded interview topic map                   | https://github.com/theEmbeddedGeorge/theEmbeddedNewTestament.github.io                                                  |
| Linux kernel 面試考題 — HackMD                                       | Process/thread, sync primitives, memory layout in Chinese   | https://hackmd.io/@accdlab/SyCxqEGKO                                                                                    |
| Indeed (in.indeed.com) — Linux Device Driver interview Qs        | 28 device-driver questions                                   | https://in.indeed.com/career-advice/interviewing/linux-device-driver-interview-questions                                |
| TestBook — Top 55 Linux Device Driver interview questions        | Device tree, IRQ, DMA, kmalloc/vmalloc                       | https://testbook.com/interview/linux-device-driver-interview-questions                                                  |
| SiFive Blog — RISC-V Software Updates 2024 → 2025                | Topical (AIA driver upstream, RVA23, what they ship in 2025) | https://www.sifive.com/blog/risc-v-software-great-progress-in-2024-and-much-mo                                          |

---

## Cross-references in this vault

- [[SiFive - Interview Prep]] — phone-round notes, role context
- [[SiFive - Computer Architecture Prep]] — RVWMO, MESI, Sv39, CLINT/PLIC
- [[CPU Control Hazard]]
- [[MESI Cache Coherency Protocol]]
- [[kmalloc() vs. vmalloc()]]
- [[Linux Kernel Booting]]
