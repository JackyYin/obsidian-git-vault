---
tags:
  - C
  - language-spec
  - interview
  - embedded-C
spec: 'ISO/IEC 9899:2011 §6.3.1.1, §6.3.1.3, §6.3.1.8'
---

# C Integer Conversion Matrix

> Reference for **§6.3.1.1** (rank, integer promotions), **§6.3.1.3** (signed/unsigned), and **§6.3.1.8** (usual arithmetic conversions) of the C11 standard. Useful for SiFive / embedded interviews — see [[SiFive - Coding Interview Problems]] Tier 0 §11.

---

## Step 1 — Integer Conversion Rank (§6.3.1.1)

```
_Bool  <  char / signed char / unsigned char
       <  short / unsigned short
       <  int   / unsigned int
       <  long  / unsigned long
       <  long long / unsigned long long
```

A signed type and its corresponding unsigned type have **the same rank**.

---

## Step 2 — Integer Promotions (§6.3.1.1 ¶2)

Every operand of rank ≤ rank(`int`) is promoted **before** the binary rule runs:

| Original type                                                                     | Promotes to (assuming standard ≥32-bit `int`) |
| --------------------------------------------------------------------------------- | --------------------------------------------- |
| `_Bool`, `signed char`, `unsigned char`, `char`, `short`, `unsigned short`        | `int` *                                       |
| `int`, `unsigned int`, `long`, `unsigned long`, `long long`, `unsigned long long` | unchanged                                     |

\* `unsigned int` only when the small type's full value range cannot fit in `int` (e.g. 16-bit-int platforms with `unsigned short`). On modern Linux/RISC-V/x86-64 they all become plain `int`.

So **the small types collapse**: any pairing involving `_Bool`/`char`/`short` reduces to "`int` op X" after promotion. The interesting matrix is the 6×6 of post-promotion types.

---

## Step 3 — Usual Arithmetic Conversions (§6.3.1.8) — the algorithm

After promotions, given two operands of types T1, T2:

1. Same type → that type. **Done.**
2. Both signed **or** both unsigned → result = type of higher rank.
3. Mixed signedness:
   - **3a.** If `rank(unsigned) ≥ rank(signed)` → signed is converted to **the unsigned type**.
   - **3b.** Else if the signed type can represent **all values** of the unsigned type → unsigned is converted to **the signed type**.
   - **3c.** Else → both converted to the **unsigned version of the signed type**.

> **Subtle point about 3b vs 3c:** rule 3b only fires when the signed type is **strictly wider** than the unsigned type. On LP64, `long long` and `unsigned long` are *both 64 bits*, so `long long` cannot represent the upper half of `unsigned long`'s range — rule 3b fails and we fall through to 3c.

---

## Step 4 — The Matrix (LP64: Linux x86-64, RISC-V Linux, AArch64 Linux, macOS)

Sizes assumed: `int` = 32 bit, `long` = 64 bit, `long long` = 64 bit. The matrix is symmetric.

| ↓ \ →                    | `int`                  | `unsigned int`       | `long`                 | `unsigned long`        | `long long`            | `unsigned long long`   |
| ------------------------ | ---------------------- | -------------------- | ---------------------- | ---------------------- | ---------------------- | ---------------------- |
| **`int`**                | `int`                  | `unsigned int` ¹     | `long`                 | `unsigned long` ²      | `long long`            | `unsigned long long` ² |
| **`unsigned int`**       | `unsigned int` ¹       | `unsigned int`       | `long` ³               | `unsigned long`        | `long long` ³          | `unsigned long long`   |
| **`long`**               | `long`                 | `long` ³             | `long`                 | `unsigned long` ¹      | `long long`            | `unsigned long long` ² |
| **`unsigned long`**      | `unsigned long` ²      | `unsigned long`      | `unsigned long` ¹      | `unsigned long`        | `unsigned long long` ⁴ | `unsigned long long`   |
| **`long long`**          | `long long`            | `long long` ³        | `long long`            | `unsigned long long` ⁴ | `long long`            | `unsigned long long` ¹ |
| **`unsigned long long`** | `unsigned long long` ² | `unsigned long long` | `unsigned long long` ² | `unsigned long long`   | `unsigned long long` ¹ | `unsigned long long`   |

**Rule annotations** (which sub-rule of §6.3.1.8 fires):

- ¹ = **3a, equal rank** — signed converts to unsigned.
- ² = **3a, unsigned has higher rank** — signed converts to unsigned.
- ³ = **3b, signed has higher rank AND represents all unsigned values** — unsigned converts to signed.
- ⁴ = **3c, signed has higher rank but CANNOT represent all unsigned values** — both convert to the unsigned version of the signed type. (LP64-specific: same-width `long long` / `unsigned long`.)

Unmarked cells fall under rule **2** (same signedness → higher rank).

---

## Step 5 — Including the small types

For any small type *S* ∈ {`_Bool`, `char`, `signed char`, `unsigned char`, `short`, `unsigned short`}:

```
S  op  X     ≡     int  op  X         (look up "int" row above)
```

**Example:** `unsigned short × long` on LP64 → promote `unsigned short` to `int` → look up `int × long` → **`long`**. (Surprises people who expect `unsigned long`.)

---

## Step 6 — Other data models (the cells that flip)

The differences are concentrated where **`long` interacts with `unsigned int`**, because `long`'s width changes:

|                          | Linux/macOS **LP64** (long=64) | Windows x64 **LLP64** (long=32) | 32-bit **ILP32** (long=32) |
| ------------------------ | ------------------------------ | ------------------------------- | -------------------------- |
| `unsigned int × long`    | `long` ³                       | `unsigned long` ¹               | `unsigned long` ¹          |
| `int × unsigned long`    | `unsigned long` ²              | `unsigned long` ¹               | `unsigned long` ¹          |
| `long × unsigned long`   | `unsigned long` ¹              | `unsigned long` ¹               | `unsigned long` ¹          |

The `long long` columns are stable across all common data models because `long long` is always ≥ 64 bits.

The `long long × unsigned long` cell is **rule 3c on LP64** because they're both 64 bit; on a hypothetical 128-bit-`long long` platform it would flip to `long long` via rule 3b.

---

## Worked example — why `long long × unsigned long` ≠ `long long`

```c
long long      a = -1LL;
unsigned long  b = 0;
auto           r = a + b;        // type of r = ?
```

Walk §6.3.1.8:

1. Same type? No.
2. Same signedness? No.
3. Mixed signedness:
   - **3a:** rank(`unsigned long`) ≥ rank(`long long`)? No — `long long` outranks `long`.
   - **3b:** Can `long long` represent all `unsigned long` values? `long long` max = 2⁶³−1, `unsigned long` max = 2⁶⁴−1. **No** — values 2⁶³ … 2⁶⁴−1 don't fit.
   - **3c:** Both convert to **unsigned version of signed type** = `unsigned long long`.

So `r` is `unsigned long long` and `-1LL` becomes `0xFFFF'FFFF'FFFF'FFFF`. The intuition that "higher-rank wins" is wrong here because the signed type isn't *wide enough* to swallow the unsigned type's range.

---

## Classic gotchas to drill before SiFive

```c
// Gotcha 1 — the famous Nigel Jones #12
unsigned int a = 6;
int b = -20;
(a + b > 6) ? puts(">6") : puts("<=6");   // prints ">6"
// rule 3a: equal rank → b promoted to unsigned int → huge positive value
```

```c
// Gotcha 2 — sizeof returns size_t (unsigned)
int n = -1;
if (n < sizeof(int))          // rule 3a → n becomes huge size_t
    puts("smaller");          // never prints
else
    puts("not smaller");      // this prints
```

```c
// Gotcha 3 — sign extension across width AND sign
int            x = -1;
unsigned long  y = x;         // x is sign-extended to long, then reinterpreted as unsigned long
                              // y == ULONG_MAX on LP64
```

```c
// Gotcha 4 — the long long / unsigned long surprise
long long      s = -1LL;
unsigned long  u = 1UL;
if (s < u) puts("s < u");     // never prints
else       puts("s >= u");    // prints — both promoted to unsigned long long, s becomes huge
```

---

## Sources

- [ISO/IEC 9899:2011 (n1548 draft)](https://www.open-std.org/jtc1/sc22/wg14/www/docs/n1548.pdf) — §6.3.1.1, §6.3.1.3, §6.3.1.8
- [cppreference — Implicit conversions](https://en.cppreference.com/w/c/language/conversion) — Usual arithmetic conversions
- [Nigel Jones — A 'C' Test: 0x10 Best Questions for Embedded Programmers](https://rmbconsulting.us/publications/a-c-test-the-0x10-best-questions-for-would-be-embedded-programmers/) — Q12 illustrates rule 3a

## Cross-references

- [[SiFive - Coding Interview Problems]] — Tier 0 #11 (integer promotion gotcha)
- [[SiFive - Computer Architecture Prep]] — RISC-V is LP64 on Linux, so the matrix above applies directly
