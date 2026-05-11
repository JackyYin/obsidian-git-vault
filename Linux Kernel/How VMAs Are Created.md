---
tags:
  - linux-kernel
  - memory-management
  - vma
  - mmap
  - fork
  - execve
  - virtual-memory
created: '2026-05-11'
---
# How VMAs Are Created

A **VMA** (`struct vm_area_struct`) is the kernel's record of one contiguous mapped region in a process's virtual address space — `[vm_start, vm_end)` with permissions, backing file, and ops. Each `task->mm` owns a collection of VMAs stored in a **maple tree** (`mm->mm_mt`); older kernels used an rbtree + linked list, removed in 6.1.

```
task_struct  →  mm_struct  →  mm_mt  →  vma1, vma2, vma3, ...
                                          ↑
                                   each: vm_start, vm_end,
                                         vm_flags, vm_file,
                                         vm_ops, vm_pgoff
```

## The three-step lifecycle

Every VMA, no matter who creates it, is built by the same sequence:

```
                   ┌─────────────────────────┐
                   │ 1. vm_area_alloc()      │  kmem_cache_alloc(vm_area_cachep, GFP_KERNEL)
                   │    (kernel/fork.c:461)  │  → zeroed struct + vma_init
                   └────────────┬────────────┘
                                ↓
                   ┌─────────────────────────┐
                   │ 2. fill fields          │  vm_start, vm_end, vm_flags,
                   │                         │  vm_file, vm_ops, vm_pgoff
                   └────────────┬────────────┘
                                ↓
                   ┌─────────────────────────┐
                   │ 3. vma_link / store     │  insert into mm->mm_mt
                   │    into maple tree      │  under mmap_write_lock
                   └─────────────────────────┘
```

The interesting question is *who fills in the fields, and from what?* There are four entry points.

---

## 1. `execve()` — the first set of VMAs a process ever owns

When `execve()` runs, the existing `mm_struct` is torn down and a fresh one built from scratch. The new VMAs come from the ELF loader and the kernel:

```
do_execve()
  └─ bprm_mm_init()                  ← creates the new mm
  └─ exec_binprm()
       └─ search_binary_handler()
            └─ load_elf_binary()      fs/binfmt_elf.c
                 ├─ setup_arg_pages()           → VMA for the stack ([stack])
                 ├─ for each PT_LOAD segment:
                 │    └─ elf_map()              → VMA for .text/.rodata/.data/.bss
                 │         └─ vm_mmap()
                 │              └─ do_mmap()
                 │                   └─ mmap_region()
                 ├─ set_brk()                   → VMA for the heap ([heap])
                 ├─ load_elf_interp()           → VMAs for ld.so (PT_LOAD segments)
                 └─ arch_setup_additional_pages()
                      └─ vdso_mmap()            → VMA for [vdso] / [vvar]
```

A freshly-`exec`'d process already has roughly: `.text`, `.rodata`, `.data`, `.bss`, `[heap]`, `[stack]`, `[vdso]`, `[vvar]`, plus the same set for the dynamic linker. That's the initial contents of `/proc/$pid/maps`.

## 2. `fork()` / `clone()` — copying the parent's VMAs

`fork()` does not call `execve()`; the child needs a copy of the parent's address space. That's `dup_mmap`:

```
sys_clone / sys_fork
  └─ copy_process()
       └─ copy_mm()
            └─ dup_mm()
                 └─ dup_mmap()                 kernel/fork.c:626
                      └─ for each VMA in parent's mm_mt:
                           ├─ vm_area_dup(parent_vma)   ← kmem_cache_alloc + memcpy
                           ├─ adjust file refcount, anon_vma chains
                           ├─ copy_page_range()         ← page tables (COW marks)
                           └─ vma_iter_store(child_mt, new_vma)
```

Key detail: page tables are walked and PTEs marked **copy-on-write**. The VMAs are independent structs in the child, but their backing physical pages stay shared until one side writes. The maple tree of VMAs and the page tables are **two parallel structures** that must remain consistent.

`clone(CLONE_VM)` (used by pthreads) skips this entirely — the child's `task->mm` *is* the parent's `mm_struct`, refcount-incremented. No VMA duplication; threads share the address space.

## 3. `mmap()` — runtime creation, the canonical path

Every other VMA creation funnels through here:

```
sys_mmap (glibc → __NR_mmap)
  └─ ksys_mmap_pgoff()
       └─ vm_mmap_pgoff()
            └─ do_mmap()                       mm/mmap.c
                 ├─ get_unmapped_area()         ← find a free hole in mm_mt
                 ├─ calculate vm_flags from prot/flags
                 └─ mmap_region()               mm/mmap.c:2722
                      ├─ vma_merge() if adjacent to a compatible existing VMA
                      │    └─ if merged: no new struct, just extend → done
                      ├─ vm_area_alloc()        ← new VMA struct
                      ├─ fill in vm_start/end/flags/pgoff
                      ├─ if file-backed: call_mmap(file, vma)
                      │    └─ ext4_file_mmap (or whatever) sets vma->vm_ops
                      │       so future page faults route to filemap_fault
                      ├─ vma_link():
                      │    ├─ __vma_link_file()         ← add to address_space->i_mmap
                      │    ├─ vma_iter_store(&vmi, vma) ← insert into mm_mt
                      │    └─ mm->map_count++
                      └─ return vm_start
```

**`mmap()` does not allocate physical pages.** It only creates a VMA. The first time userspace touches an address inside that range, the CPU takes a page fault → `handle_mm_fault()` → walks the VMA, runs `vma->vm_ops->fault` (file-backed) or `do_anonymous_page` (anon) → allocates a page → installs a PTE. That's lazy allocation.

The `vma_merge` step is why `/proc/$pid/maps` does not have one entry per `mmap` call — adjacent compatible mappings collapse into a single VMA.

## 4. `brk()` / `sbrk()` — the heap

`malloc()` for small sizes calls `brk()` to grow the heap VMA in place:

```
sys_brk
  └─ do_brk_flags()                  mm/mmap.c
       ├─ if growing into a free range: vma_merge with the existing [heap] VMA
       │   → bumps vm_end, no new struct
       └─ else: vm_area_alloc + vma_link as in mmap_region
```

Large `malloc` calls (typically > 128 KB by glibc default) bypass `brk` and go straight to `mmap(MAP_ANONYMOUS)`. That's why `/proc/$pid/maps` of a long-running process has many anonymous regions in addition to `[heap]`.

---

## The struct itself

```c
struct vm_area_struct {                       /* include/linux/mm_types.h:619 */
    unsigned long vm_start;                   /* inclusive */
    unsigned long vm_end;                     /* exclusive */
    struct mm_struct *vm_mm;                  /* owning mm */
    pgprot_t vm_page_prot;                    /* derived from vm_flags */
    unsigned long vm_flags;                   /* VM_READ/WRITE/EXEC/SHARED/... */
    struct file *vm_file;                     /* NULL for anonymous */
    unsigned long vm_pgoff;                   /* file offset in pages */
    const struct vm_operations_struct *vm_ops;/* ->fault, ->open, ->close */
    struct anon_vma *anon_vma;                /* anon reverse mapping */
    /* maple-tree node bookkeeping, per-VMA lock, etc. */
};
```

Fields fall into three buckets:

| Bucket | Fields | Purpose |
|---|---|---|
| **Range**   | `vm_start`, `vm_end`, `vm_pgoff` | Where in the address space |
| **Policy**  | `vm_flags`, `vm_page_prot`       | What's allowed there |
| **Backing** | `vm_file` + `vm_ops`, or `anon_vma` | What to do on a page fault |

The page-fault handler reads exactly these three buckets to satisfy a fault.

## Locking

All four creation paths take `mmap_write_lock(mm)` before mutating `mm_mt`. Readers (the page-fault path, `/proc/$pid/maps`, `get_user_pages`) take `mmap_read_lock(mm)`.

Recent kernels added **per-VMA locking** (`vma->vm_lock`, allocated in `vma_lock_alloc()` inside `vm_area_alloc`) so page faults on *different* VMAs can run concurrently without serializing on `mmap_lock`. The big lock still guards structural changes (insert/remove); the per-VMA lock guards fault-time field reads.

## The single take-away

> **All VMAs are created by the same three-step sequence** — `vm_area_alloc()` → fill fields → `vma_link()` into the maple tree. The four entry points (`execve`, `fork`, `mmap`, `brk`) just differ in *where the field values come from*: ELF loader, parent VMA, syscall arguments, or heap-extension logic.

Every line in `/proc/$pid/maps` was produced by one of these four paths.

## References

| File | What's there |
|---|---|
| `include/linux/mm_types.h:619` | `struct vm_area_struct` definition |
| `kernel/fork.c:461`            | `vm_area_alloc` — kmem_cache + vma_init + per-VMA lock |
| `kernel/fork.c:626`            | `dup_mmap` — fork-time VMA cloning |
| `mm/mmap.c:2722`               | `mmap_region` — the canonical creation path |
| `mm/mmap.c`                    | `do_brk_flags` — heap extension |
| `fs/binfmt_elf.c`              | `load_elf_binary` — execve's VMAs |
| `mm/memory.c`                  | `handle_mm_fault` — consumer of VMA fields at fault time |

## Cross-links

- [[Linux Driver Model]] — mmap_lock contention is visible from driver code that pins user memory
- [[Kernel Context Preemption Matrix]] — `mmap_write_lock` is a sleeping lock (rwsem); forbids hardirq/softirq usage
- [[Interrupt Code Path - Hardware to Driver Handler]] — page-fault entry is symmetric: IDT vector 14 → `exc_page_fault` → `handle_mm_fault` reads the VMA
