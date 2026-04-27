---
tags:
  - x86
  - assembly
  - stack
  - calling-convention
---

# x86 Function Prologue & Epilogue

---

## Overview

Every x86-64 function call follows a predictable stack discipline. The prologue sets up a new stack frame; the epilogue tears it down and returns to the caller.

```
  (example: caller at 0x401040, foo at 0x401100, stack base ~0x7fffffffe038)
```

---

## Prologue

### STEP 0 — caller executes `call foo` (at 0x401040)
(pushes return address 0x401045, jumps to foo at 0x401100)

```
  addr              Stack                              Registers
                ┌──────────────────┐
  0x7fffffffe038│  caller's frame  │
                ├──────────────────┤
  0x7fffffffe030│  return address  │  = 0x401045     RSP = 0x7fffffffe030
                │  (caller RIP+5)  │◀── RSP          RIP = 0x401100  (foo entry)
                └──────────────────┘                 RBP = 0x7fffffffe038
```

### STEP 1 — `push rbp` (at 0x401100)
(RSP -= 8, save caller's RBP value 0x7fffffffe038)

```
  addr              Stack                              Registers
                ┌──────────────────┐
  0x7fffffffe038│  caller's frame  │
                ├──────────────────┤
  0x7fffffffe030│  return address  │  = 0x401045
                ├──────────────────┤
  0x7fffffffe028│  saved RBP       │  = 0x7fffffffe038
                │                  │◀── RSP          RSP = 0x7fffffffe028
                └──────────────────┘                 RBP = 0x7fffffffe038 (unchanged)
```

### STEP 2 — `mov rbp, rsp` (at 0x401101)

```
  addr              Stack                              Registers
                ┌──────────────────┐
  0x7fffffffe038│  caller's frame  │
                ├──────────────────┤
  0x7fffffffe030│  return address  │  = 0x401045
                ├──────────────────┤
  0x7fffffffe028│  saved RBP       │  = 0x7fffffffe038
                │                  │◀── RSP, RBP     RSP = 0x7fffffffe028
                └──────────────────┘                 RBP = 0x7fffffffe028
```

### STEP 3 — `sub rsp, 32` (at 0x401104)
(allocate 32 bytes for locals)

```
  addr              Stack                              Registers
                ┌──────────────────┐
  0x7fffffffe038│  caller's frame  │
                ├──────────────────┤
  0x7fffffffe030│  return address  │  = 0x401045
                ├──────────────────┤
  0x7fffffffe028│  saved RBP       │◀── RBP          RBP = 0x7fffffffe028
                ├──────────────────┤
  0x7fffffffe020│  local var a     │  [RBP -  8]
                ├──────────────────┤
  0x7fffffffe018│  local var b     │  [RBP - 16]
                ├──────────────────┤
  0x7fffffffe010│  local var c     │  [RBP - 24]
                ├──────────────────┤
  0x7fffffffe008│  local var d     │  [RBP - 32]
                │                  │◀── RSP          RSP = 0x7fffffffe008
                └──────────────────┘
```

---

## Epilogue

### STEP 4 — `mov rsp, rbp` (at 0x401130)
(collapse locals — RSP snaps back to RBP)

```
  addr              Stack                              Registers
                ┌──────────────────┐
  0x7fffffffe038│  caller's frame  │
                ├──────────────────┤
  0x7fffffffe030│  return address  │  = 0x401045
                ├──────────────────┤
  0x7fffffffe028│  saved RBP       │◀── RSP, RBP     RSP = 0x7fffffffe028
                ├ ─ ─ ─ ─ ─ ─ ─ ─ ┤
  0x7fffffffe020│  (dead locals)   │
                  ...
  0x7fffffffe008│                  │  no longer accessible
                └ ─ ─ ─ ─ ─ ─ ─ ─ ┘
```

### STEP 5 — `pop rbp` (at 0x401133)
(restore caller's RBP = 0x7fffffffe038; RSP += 8)

```
  addr              Stack                              Registers
                ┌──────────────────┐
  0x7fffffffe038│  caller's frame  │
                ├──────────────────┤
  0x7fffffffe030│  return address  │  = 0x401045
                │                  │◀── RSP          RSP = 0x7fffffffe030
                └──────────────────┘                 RBP = 0x7fffffffe038 (restored)
```

### STEP 6 — `ret` (at 0x401134)
(pop 0x401045 into RIP; RSP += 8; jump to 0x401045)

```
  addr              Stack                              Registers
                ┌──────────────────┐
  0x7fffffffe038│  caller's frame  │◀── RSP          RSP = 0x7fffffffe038
                └──────────────────┘                 RBP = 0x7fffffffe038
                                                     RIP = 0x401045  ← back in caller
```

---

## Code Layout (text segment)

```
  0x401040  call foo          ← caller issues call (encodes target 0x401100)
  0x401045  mov eax, ...      ← return address (instruction after call)
  ...
  0x401100  push rbp          ← foo entry (prologue start)
  0x401101  mov  rbp, rsp
  0x401104  sub  rsp, 32
  ...
  0x401130  mov  rsp, rbp     ← epilogue start
  0x401133  pop  rbp
  0x401134  ret               ← jumps back to 0x401045
```

---

## Summary

| Phase    | Instructions                          | Effect                              |
|----------|---------------------------------------|-------------------------------------|
| Prologue | `push rbp`                            | Save caller's frame pointer         |
|          | `mov rbp, rsp`                        | Establish new frame base            |
|          | `sub rsp, N`                          | Allocate space for locals           |
| Epilogue | `mov rsp, rbp` (or `leave`)           | Deallocate locals                   |
|          | `pop rbp`                             | Restore caller's frame pointer      |
|          | `ret`                                 | Pop return address → jump to caller |

> `call` = `push RIP` + `jmp`  
> `ret`  = `pop RIP` (implicit jump)  
> The return address is always `call` instruction address + 5 bytes (`call rel32` encoding).
