---
tags:
  - linux-kernel
  - device-driver
  - irq
created: 2026-04-29
---

## `irq_handler_t` — The IRQ Handler Function Pointer Type

```c
typedef irqreturn_t (*irq_handler_t)(int, void *);
```

Breaking it down:

```
typedef   irqreturn_t   (*irq_handler_t)  (int,  void *);
│         │              │                 │      │
│         │              │                 └──────┴── parameter types
│         │              │                            (irq number, dev_id)
│         │              └─ the type name being defined
│         │                 (* ) means it's a pointer to a function
│         └─ return type of the function being pointed to
└─ creates a type alias
```

`irq_handler_t` is a type representing **a pointer to any function with this exact signature**:

```c
irqreturn_t my_handler(int irq, void *dev_id);
```

### Why `(*name)` syntax?

Without the `*`, this would be a function type, not a pointer-to-function type. The parentheses around `*irq_handler_t` are required because `*` binds more loosely than `()` — without them, `*` would be parsed as part of the return type.

### Parameters

| Parameter | Type       | Meaning                                                                       |
| --------- | ---------- | ----------------------------------------------------------------------------- |
| `int`     | irq number | The IRQ line that fired — useful if one handler serves multiple IRQs          |
| `void *`  | `dev_id`   | Whatever pointer you passed to `request_irq()` — typically your device struct |

The `void *` is the kernel's way of passing arbitrary context into the handler without knowing its type at the call site.

### Usage

```c
// request_irq takes an irq_handler_t as 2nd argument
int request_irq(unsigned int irq,
                irq_handler_t handler,   // ← the type in action
                unsigned long flags,
                const char *name,
                void *dev_id);

// your handler matches the signature automatically
irqreturn_t my_gpio_handler(int irq, void *dev_id) {
    // handle interrupt
    return IRQ_HANDLED;
}

// pass it directly — no cast needed because the type matches
request_irq(irq_num, my_gpio_handler, IRQF_TRIGGER_RISING, "my_gpio", NULL);
```
