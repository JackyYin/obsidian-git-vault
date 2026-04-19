---
tags:
  - C
  - embedded
---

1. What's the difference between logical shift and arithmetic shift ?

2. What's the value of  x  and y ?
```C
int8_t x = -1;
uint8_t y = 0xFF;

x >>= 7;
y >>= 7;
```

3. What's the value of x and y ?
```C
int8_t x = 1;
uint8_t y = 1;

x <<= 7;
y <<= 7;
```

4. What's the value of x ?
```C
int8_t x = -1;

x <<= 1;
```


5. Write down a code paragraph to set & clear bits 3 of variable x:

```C
#define BITS_3 (1 << 3)

static int x;

static void set(void) { x |= BITS_3; }

static void clear(void) { x &= ~BITS_3; }

static void toggle(void) { x ^= BITS_3; }
```

6. What's the meaning of `X & (-X)` ?
```
  101100
& 010100
---------
  000100
```

7. What's the meaning of `X & (X-1)` ?
```
  101100
& 101011
---------
  101000
```


LeetCode:
- [number of 1 bits](https://leetcode.com/problems/number-of-1-bits/)
- single number i
- single number ii
- [single number iii](https://leetcode.com/problems/single-number-iii/)

Refs:
- [jserv](https://hackmd.io/@sysprog/c-bitwise)