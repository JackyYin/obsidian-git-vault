---
tags:
  - C
  - preprocessor
  - embedded
---
1. What's the printed result in the following code segment (A)?
2. What's the printed result in the following code segment (B) ?

```C
#define MIN(A,B) ((A) <= (B) ? (A) : (B))

static int arr[5] = { 10, 11, 12, 13, 14 };

int main() {
    int b = 20;
    int *p = &arr[0];
    int least = MIN(*p++, b);
    
    // (A) printf("%d\n", least);
    // (B) printf("%d\n", *p);
    return 0;
}
```

3. Implement `offsetof(type, member)`
```c
#define offsetof(type, member) \
    (unsigned long)&(((type *)(0))->member)
```


4. Implement `container_of(ptr, type, member)`
```c
#define container_of(ptr, type, member) \
    (type *)((void *)(ptr) - offsetof(type, member))
```


Refs:
- [jserv](https://hackmd.io/@sysprog/c-preprocessor)