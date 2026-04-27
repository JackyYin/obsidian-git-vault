---
tags:
  - linux-kernel
  - memory-management
  - page-fault
  - MMU
  - virtual-memory
---

# Linux Kernel Page Fault Mechanism

---

## What Triggers a Page Fault

When the CPU accesses a virtual address, the MMU walks the page table. If anything goes wrong, the MMU raises a **page fault exception** — trapping into the kernel:

```
CPU issues memory access (LOAD / STORE)
          │
          ▼
      MMU checks TLB
          │
    TLB hit? ──YES──▶ physical address found ──▶ access memory ✅
          │
          NO
          │
          ▼
    MMU walks page table
          │
    PTE valid? ──YES──▶ load into TLB ──▶ retry access ✅
          │
          NO (or access violation)
          │
          ▼
    MMU raises Page Fault exception
          │
          ▼
    CPU saves registers, jumps to fault handler
    (stvec on RISC-V, IDT vector 14 on x86)
```

---

## Three Root Causes of Page Faults

```
  ┌─────────────────────────────────────────────────────────────┐
  │              Why did the page fault happen?                 │
  │                                                             │
  │  1. Page not present      ← PTE present bit = 0             │
  │     (most common)           page never loaded, or swapped   │
  │                             out to disk                     │
  │                                                             │
  │  2. Permission violation  ← PTE present = 1, but            │
  │                             write to read-only page         │
  │                             execute from non-exec page      │
  │                             user access to kernel page      │
  │                                                             │
  │  3. No mapping at all     ← virtual address not in any VMA  │
  │                             → segfault                      │
  │                                                             │
  └─────────────────────────────────────────────────────────────┘
```

---

## Page Fault Handler — Full Flow

```
Page Fault Exception
        │
        ▼
  fault address saved in:
    x86:    CR2 register
    RISC-V: stval (supervisor trap value) register
        │
        ▼
  do_page_fault()  (arch-specific entry)
        │
        ▼
  __do_page_fault() / handle_mm_fault()  (generic MM layer)
        │
        ▼
  ┌─────────────────────────────────────────────────────────┐
  │           Find the VMA containing fault address         │
  │           find_vma(mm, address)                         │
  └───────────────────────┬─────────────────────────────────┘
                          │
              ┌───────────┴───────────┐
              │                       │
           VMA found               No VMA found
              │                       │
              ▼                       ▼
    check permissions            ┌──────────────┐
    match fault type             │  SIGSEGV     │
              │                  │  segfault    │
              │                  └──────────────┘
     ┌────────┴─────────┐
     │                  │
  bad perms         perms OK
     │                  │
     ▼                  ▼
  SIGSEGV          handle_mm_fault()
  (segfault)            │
                        │
           ┌────────────┼────────────┬──────────────┐
           │            │            │              │
           ▼            ▼            ▼              ▼
       demand        swap-in      CoW          stack
       paging        from disk    fault        growth
```

---

## Case 1 — Demand Paging (Most Common)

The kernel doesn't load all pages at program start — it maps them lazily:

```
  execve("./program")
        │
        ▼
  kernel sets up VMAs (virtual memory areas)
  but does NOT allocate physical pages yet
  PTEs all marked NOT PRESENT
        │
        ▼
  program starts running
        │
  first access to code page (.text)
        │
        ▼
  PAGE FAULT  ← PTE not present
        │
        ▼
  do_page_fault()
        │
        ▼
  allocate physical page frame
  read code from ELF file (or zero-fill for .bss)
  update PTE: set present bit, physical address
        │
        ▼
  return from fault handler
  CPU retries the instruction ✅

  ┌────────────────────────────────────────────────────────┐
  │  Virtual Address Space          Physical RAM           │
  │                                                        │
  │  ┌──────────────┐              ┌──────────────┐        │
  │  │  .text VMA   │──── fault ──▶│  page frame  │        │
  │  │  [not pres.] │    handle    │  (loaded now)│        │
  │  └──────────────┘              └──────────────┘        │
  │  ┌──────────────┐                                      │
  │  │  .data VMA   │──── not yet accessed, still empty    │
  │  │  [not pres.] │                                      │
  │  └──────────────┘                                      │
  └────────────────────────────────────────────────────────┘
```

---

## Case 2 — Swap-out and Swap-in (Major Fault)

### Swapping Out

Page evicted from physical memory to disk under memory pressure:

```
                                Swapping Out

  User Virtual Address Space       TLB            Physical Address Space
  ┌────────────────┐          ┌──────────┐        ┌────────────────┐
  │                │          │          │        │                │
  │  ┌──────────┐  │          │ ┌──────┐ │        │  ┌──────────┐  │
  │  │  page 0  │──┼──────────┼▶│ VPN0 │─┼────────┼─▶│ frame 0  │  │
  │  ├──────────┤  │          │ ├──────┤ │        │  ├──────────┤  │
  │  │  page 1  │──┼──────────┼▶│ VPN1 │─┼────────┼─▶│ frame 1  │  │
  │  ├──────────┤  │          │ ├──────┤ │        │  ├──────────┤  │
  │  │  ██████  │  │          │ │      │ │        │  │          │  │
  │  │  page 2  │──┼─────────▶│ │(none)│─┼────────┼─▶│  (free)  │──│────┐
  │  │  ██████  │  │erase TLB │ ├──────┤ │        │  │          │  │    │
  │  │(evicted) │  │          │ │ VPN3 │ |        |  ├──────────┤  │    │
  │  ├──────────┤  │          │ |      |─┼────────┼─▶│ frame 3  │  │    │
  │  │  page 3  │──┼──────────┼▶└──────┘ │        │  └──────────┘  │    │
  │  └──────────┘  │          └──────────┘        └────────────────┘    │
  └────────────────┘                                                    │ write
                                                                        │ to
                                                    ┌────────────────┐  │ disk
                                                    │  Swap Media    │  │
                                                    │  ┌──────────┐  │  │
                                                    │  │  ██████  │◀─┼──┘
                                                    │  │  page 2  │  │
                                                    │  │  ██████  │  │
                                                    │  └──────────┘  │
                                                    └────────────────┘
```

### Swapping In

Process accesses the evicted page — triggers a page fault, kernel reads page back from disk:

```
                                Swapping In

  User Virtual Address Space       TLB            Physical Address Space
  ┌────────────────┐          ┌──────────┐        ┌────────────────┐
  │                │          │          │        │                │
  │  ┌──────────┐  │          │ ┌──────┐ │        │  ┌──────────┐  │
  │  │  page 0  │──┼──────────┼▶│ VPN0 │─┼────────┼─▶│ frame 0  │  │
  │  ├──────────┤  │          │ ├──────┤ │        │  ├──────────┤  │
  │  │  page 1  │──┼──────────┼▶│ VPN1 │─┼────────┼─▶│ frame 1  │  │
  │  ├──────────┤  │          │ ├──────┤ │        │  ├──────────┤  │
  │  │  ██████  │  │  PAGE    │ │ VPN2 │ │        │  │  ██████  │  │
  │  │  page 2  │──┼──FAULT──▶│ │(new) │─┼────────┼─▶│ frame 2  │  │◀──┐
  │  │  ██████  │  │  present │ ├──────┤ │        │  │ (filled) │  │   │
  │  │(accessed)│  │  =0→1    │ │ VPN3 │ |        |  ├──────────┤  │   │
  │  ├──────────┤  │          │ │      │─┼────────┼─▶│ frame 3  │  │   │
  │  │  page 3  │──┼──────────┼▶└──────┘ │        │  └──────────┘  │   │
  │  └──────────┘  │          └──────────┘        └────────────────┘   │
  └────────────────┘                                                   │ disk I/O
                                                                       │ read
                                                    ┌────────────────┐ │
                                                    │  Swap Media    │ │
                                                    │  ┌──────────┐  │ │
                                                    │  │  ██████  │──┼─┘
                                                    │  │  page 2  │  │
                                                    │  │  ██████  │  │
                                                    │  └──────────┘  │
                                                    └────────────────┘
```

Swap-in steps:
1. CPU accesses page 2 → TLB miss → MMU walks page table → `present=0` → **page fault**
2. Kernel reads swap offset from PTE → issues **disk I/O** to read page 2 from swap media
3. Page loaded into a free physical frame
4. PTE updated: `present=1`, PFN set to new frame
5. New TLB entry inserted: VPN2 → frame
6. CPU retries instruction ✅

```
  Minor fault = page in memory, just not mapped  (fast, ~µs)
  Major fault = page on disk, needs I/O          (slow, ~ms)
```

---

## Case 3 — Copy-on-Write (CoW)

The most elegant optimization in the kernel — used by `fork()`:

```
  fork() called
        │
        ▼
  child process created
  DOES NOT copy parent's pages
  instead: shares ALL pages with parent
  all shared pages marked READ-ONLY in both PTEs
        │
        ▼
  parent or child writes to a shared page
        │
        ▼
  PAGE FAULT  ← write to read-only page
        │
        ▼
  kernel detects: this is a CoW page (not a real violation)
        │
        ▼
  allocate NEW page frame
  copy content from shared page to new page
  update faulting process PTE → new page, writable
  other process PTE unchanged → still points to original
        │
        ▼
  CPU retries the write ✅

  ┌─────────────────────────────────────────────────────────────┐
  │  Before write:                After write (child wrote):    │
  │                                                             │
  │  Parent VMA                   Parent VMA                    │
  │  PTE ──────────┐              PTE ─────────────────────┐    │
  │                │                                       │    │
  │                ▼                                       ▼    │
  │         ┌────────────┐                          ┌──────────┐│
  │         │ shared page│                          │ original ││
  │         │  (ro both) │                          │  page    ││
  │         └────────────┘                          └──────────┘│
  │                ▲                                            │
  │  Child VMA     │              Child VMA                     │
  │  PTE ──────────┘              PTE ──────────────────────┐   │
  │                                                         ▼   │
  │                                                  ┌─────────┐│
  │                                                  │new copy ││
  │                                                  │(writable)│
  │                                                  └─────────┘│
  └─────────────────────────────────────────────────────────────┘
```

---

## Case 4 — Stack Growth

The stack grows downward on demand:

```
  user stack starts small
        │
  function calls deepen → stack pointer moves down
        │
  access below current stack page
        │
        ▼
  PAGE FAULT  ← address below stack, no PTE
        │
        ▼
  kernel checks: is this address within stack VMA limit?
  (checked against RLIMIT_STACK)
        │
        ├── YES → expand stack VMA downward
        │         allocate new page
        │         update PTE ✅
        │
        └── NO  → SIGSEGV (stack overflow)
```

---

## Minor vs Major Fault

```
  Minor Fault (fast, no I/O)          Major Fault (slow, needs I/O)
  ──────────────────────────          ──────────────────────────────
  page is in RAM but not mapped       page is on swap disk
  e.g. shared lib first access        e.g. rarely used data evicted
  e.g. CoW fault                      e.g. hibernation restore
  e.g. demand paging (anon page)
                                      involves:
  cost: ~microseconds                 disk seek + read → ~milliseconds
                                      cost: 1000x slower than minor fault

  tracked in /proc/<pid>/stat:
    field 10 = minflt (minor faults)
    field 11 = majflt (major faults)
```

---

## Kernel vs Userspace Fault

```
  Fault in USERSPACE                  Fault in KERNEL
  ──────────────────                  ───────────────
  normal — handled by MM              usually means a bug
  demand paging, CoW, swap            kernel dereferenced bad pointer
  ends with: retry instruction        checked via fixup table
                                        (exception tables)

  Kernel has a fixup mechanism:
    copy_from_user() / copy_to_user()
        │
        │  deliberately accesses user memory
        │  might fault if user gave bad pointer
        │
        ▼
    if fault occurs → kernel looks up exception table
        │
        ├── fixup entry found → jump to error path, return -EFAULT
        └── no fixup entry   → kernel BUG / panic (oops)
```

---

## Data Structures Involved

```
  task_struct
    └── mm_struct                ← process memory descriptor
          ├── pgd                ← page global directory (root of page table)
          ├── mmap               ← linked list of VMAs
          └── mm_rb              ← red-black tree of VMAs (fast lookup)

  vm_area_struct (VMA)           ← one per contiguous mapping
    ├── vm_start / vm_end        ← virtual address range
    ├── vm_flags                 ← VM_READ, VM_WRITE, VM_EXEC, VM_SHARED
    ├── vm_pgoff                 ← offset into file (if file-backed)
    └── vm_ops                   ← fault(), map_pages(), etc.
          │
          └── vm_ops->fault()    ← called by handle_mm_fault()
                                    for file-backed pages (mmap'd files)

  Page Table Entry (PTE) bits:
    ┌──────┬──────┬──────┬──────┬──────┬───────┬───────────────┐
    │Dirty │Access│Global│ User │Write │Present│ Physical PFN  │
    └──────┴──────┴──────┴──────┴──────┴───────┴───────────────┘
```

---

## Full Summary

```
Virtual address accessed by CPU
        │
        MMU page table walk
        │
        fault (present=0 or permission denied)
        │
        ▼
  do_page_fault()
        │
        find_vma() ──── no VMA ────▶ SIGSEGV
        │
        check perms ─── mismatch ──▶ SIGSEGV
        │
        handle_mm_fault()
	        │
	        ├── anonymous page    → alloc page, zero-fill, map PTE
	        │   (demand paging)
	        │
	        ├── file-backed page  → read from file/ELF, map PTE
	        │   (mmap / execve)
	        │
	        ├── swap page         → read from disk, map PTE (major fault)
	        │
	        ├── CoW page          → alloc new page, copy, remap PTE
	        │   (fork + write)
	        │
		    └── stack growth      → expand VMA, alloc page, map PTE
        │
        ▼
  return from exception
  CPU retries faulting instruction ✅
```
