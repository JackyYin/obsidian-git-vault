---
tags:
  - C
---
1. Master the concept of const:

```C
int main() {
    int a = 10;
    const int b = 11;
    b = 20;

    int const c = 12;
    c = 20;

    const int *d = &a;
    *d = 20;
    d = &b;

    int * const e = &a;
    *e = 20;
    e = &b;

    const int * const f = &a;
    *f = 20;
    f = &b;
}
```


2. Master data declaration:
```C
(a) int a; // An integer

(b) int *a; // A pointer to an integer

(c) int **a; // A pointer to a pointer to an integer

(d) int a[10]; // An array of 10 integers

(e) int *a[10]; // An array of 10 pointers to integers

(f) int (*a)[10]; // A pointer to an array of 10 integers

(g) int (*a)(int); // A pointer to a function a that takes an integer argument and returns an integer

(h) int (*a[10])(int); // An array of 10 pointers to functions that take an integer argument and return an integer
```



[[Preprocessor]]
[[Bits Manipulation]]