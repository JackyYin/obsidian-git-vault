---
tags:
  - kernel
  - linux
---
# Comparison

```
  ─────────────────────────────────────────────────────────────────────────────
    WHEN TO USE WHICH
  ─────────────────────────────────────────────────────────────────────────────

    Use kmalloc when:                      Use vmalloc when:
    ────────────────                       ─────────────────
    ✅ DMA transfer needed                 ✅ no DMA needed
    ✅ performance critical path           ✅ one-time large buffer (e.g. module)
    ✅ physically contiguous required      ✅ physical contiguity not required
```


|                 | `GFP_KERNEL`    | `GFP_ATOMIC`             | `GFP_NOWAIT`      |
| --------------- | --------------- | ------------------------ | ----------------- |
| Sleeps          | Yes (if needed) | Never                    | Never             |
| Success rate    | Highest         | Higher chance of success | Fails sooner      |
| Usable in IRQ?  | ❌               | ✅                        | ✅                 |
| Failure warning |                 | warns on failure         | silent on failure |

| Use          | Situation                                                      |
| ------------ | -------------------------------------------------------------- |
| `GFP_ATOMIC` | Interrupt handler, softirq, atomic context — **must not fail** |
| `GFP_NOWAIT` | Non-sleeping context, failure is **tolerable and expected**    |

|                   | `kmalloc`             | `vmalloc (GFP_KERNEL)`  |
| ----------------- | --------------------- | ----------------------- |
| Interrupt context | ✅ (with `GFP_ATOMIC`) | ❌ Never                 |
| Process context   | ✅                     | ✅                       |
| May sleep?        | Depends on flag       | **Always may sleep**    |

---
# Memory Layout

```
  ┌────────────────────────────────────────────────────────────────────────────┐
  │                     LINUX VIRTUAL ADDRESS SPACE (64-bit)                   │
  │                                                                            │
  │  0xFFFFFFFFFFFFFFFF ┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄ │
  │                                                                            │
  │  ┌──────────────────────────────────────────────────────────────────────┐  │
  │  │                        KERNEL SPACE                                  │  │
  │  │                                                                      │  │
  │  │  ┌────────────────────────────────────────────────────────────────┐  │  │
  │  │  │                  Direct Mapping Region                         │  │  │
  │  │  │            (PAGE_OFFSET ~ 0xFFFF888000000000)                  │  │  │
  │  │  │                                                                │  │  │
  │  │  │  Virtual address = Physical address + PAGE_OFFSET              │  │  │
  │  │  │  1:1 mapped — every physical RAM page has a virtual alias      │  │  │
  │  │  │                                                                │  │  │
  │  │  │  ┌──────────────┐   ┌──────────────┐   ┌──────────────┐        │  │  │
  │  │  │  │ Physical RAM │   │ Physical RAM │   │ Physical RAM │        │  │  │
  │  │  │  │   Page 0     │   │   Page 1     │   │   Page 2     │ ...    │  │  │
  │  │  │  └──────┬───────┘   └──────┬───────┘   └──────┬───────┘        │  │  │
  │  │  │         │ direct map       │ direct map        │ direct map    │  │  │
  │  │  │         ▼                  ▼                   ▼               │  │  │
  │  │  │  ┌──────────────┐   ┌──────────────┐   ┌──────────────┐        │  │  │
  │  │  │  │ Virtual Page │   │ Virtual Page │   │ Virtual Page │        │  │  │
  │  │  │  │  (kernel)    │   │  (kernel)    │   │  (kernel)    │        │  │  │
  │  │  │  └──────────────┘   └──────────────┘   └──────────────┘        │  │  │
  │  │  │                                                                │  │  │
  │  │  │  ◀────────────────── kmalloc() lives here ──────────────────▶  │  │  │
  │  │  │                                                                │  │  │
  │  │  └────────────────────────────────────────────────────────────────┘  │  │
  │  │                                                                      │  │
  │  │  ┌────────────────────────────────────────────────────────────────┐  │  │
  │  │  │                    vmalloc Region                              │  │  │
  │  │  │            (VMALLOC_START ~ 0xFFFFC90000000000)                │  │  │
  │  │  │                                                                │  │  │
  │  │  │  Virtual pages are CONTIGUOUS here...                          │  │  │
  │  │  │  ...but physical pages can be SCATTERED anywhere in RAM        │  │  │
  │  │  │                                                                │  │  │
  │  │  │  ┌────────────────────────────────────────────────────────┐    │  │  │
  │  │  │  │  vmalloc virtual range  (contiguous virtual address)   │    │  │  │
  │  │  │  │  [vaddr]  [vaddr+1]  [vaddr+2]  [vaddr+3]  [vaddr+4]   │    │  │  │
  │  │  │  └────┬─────────┬──────────┬──────────┬──────────┬────────┘    │  │  │
  │  │  │       │ page    │ page     │ page     │ page     │ page        │  │  │
  │  │  │       │ table   │ table    │ table    │ table    │ table       │  │  │
  │  │  │       ▼         ▼          ▼          ▼          ▼             │  │  │
  │  │  │   ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐       │  │  │
  │  │  │   │ Phys   │ │ Phys   │ │ Phys   │ │ Phys   │ │ Phys   │       │  │  │
  │  │  │   │ Page   │ │ Page   │ │ Page   │ │ Page   │ │ Page   │       │  │  │
  │  │  │   │ 0x04   │ │ 0x91   │ │ 0x12   │ │ 0x55   │ │ 0x33   │       │  │  │
  │  │  │   │(random)│ │(random)│ │(random)│ │(random)│ │(random)│       │  │  │
  │  │  │   └────────┘ └────────┘ └────────┘ └────────┘ └────────┘       │  │  │
  │  │  │              (scattered across physical RAM)                   │  │  │
  │  │  │                                                                │  │  │
  │  │  │  ◀────────────────── vmalloc() lives here ──────────────────▶  │  │  │
  │  │  │                                                                │  │  │
  │  │  └────────────────────────────────────────────────────────────────┘  │  │
  │  │                                                                      │  │
  │  │  ┌────────────────────────────────────────────────────────────────┐  │  │
  │  │  │              Other Kernel Regions                              │  │  │
  │  │  │                                                                │  │  │
  │  │  │   kernel text (.text)   ← kernel code                          │  │  │
  │  │  │   kernel data (.data)   ← global variables                     │  │  │
  │  │  │   kernel bss  (.bss)    ← uninitialized globals                │  │  │
  │  │  │   struct page array     ← page metadata (mem_map)              │  │  │
  │  │  │   kernel stack          ← per-CPU interrupt stack              │  │  │
  │  │  └────────────────────────────────────────────────────────────────┘  │  │
  │  │                                                                      │  │
  │  └──────────────────────────────────────────────────────────────────────┘  │
  │                                                                            │
  │  ┌──────────────────────────────────────────────────────────────────────┐  │
  │  │                        USER SPACE                                    │  │
  │  │                                                                      │  │
  │  │   stack ▼           heap ▲          .text / .data / mmap             │  │
  │  │                                                                      │  │
  │  └──────────────────────────────────────────────────────────────────────┘  │
  │                                                                            │
  │  0x0000000000000000 ┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄    │
  └────────────────────────────────────────────────────────────────────────────┘
```


```
  ─────────────────────────────────────────────────────────────────────────────
    kmalloc vs vmalloc — HEAD TO HEAD
  ─────────────────────────────────────────────────────────────────────────────

    kmalloc                                vmalloc
    ──────────────────────────────         ──────────────────────────────
    Lives in:  Direct Mapping Region       Lives in:  vmalloc Region
    Virtual:   CONTIGUOUS                  Virtual:   CONTIGUOUS
    Physical:  CONTIGUOUS ← key diff       Physical:  SCATTERED  ← key diff
    Page table: no extra mapping needed    Page table: must set up per page
    Speed:     FAST (no TLB overhead)      Speed:     SLOWER (TLB misses)
    Max size:  limited (~4MB typical)      Max size:  very large (GBs)
    Use case:  small, frequent allocs      Use case:  large, infrequent allocs
               DMA-safe (if GFP_DMA)                  driver buffers, modules
    DMA safe?: YES (physically contig.)    DMA safe?: NO (not physically contig.)
    Sleepable?: NO (GFP_ATOMIC allowed)    Sleepable?: YES (may sleep)
```

```
  ─────────────────────────────────────────────────────────────────────────────
    PHYSICAL MEMORY VIEW
  ─────────────────────────────────────────────────────────────────────────────

    kmalloc(16KB)                          vmalloc(16KB)

    Physical RAM                           Physical RAM
    ┌──────────┐  0x00000                  ┌──────────┐  0x00000
    │          │                           │  page A  │  ◀── vmalloc page 1
    ├──────────┤  0x01000                  ├──────────┤  0x01000
    │  page 1  │  ◀── kmalloc block        │          │
    ├──────────┤  0x02000  (4 pages,       ├──────────┤  0x02000
    │  page 2  │  ◀── all contiguous)      │          │
    ├──────────┤  0x03000                  ├──────────┤  0x03000
    │  page 3  │                           │          │
    ├──────────┤  0x04000                  ├──────────┤  0x04000
    │  page 4  │                           │  page B  │  ◀── vmalloc page 2
    ├──────────┤  0x05000                  ├──────────┤  0x05000
    │          │                           │          │
    ├──────────┤  ...                      ├──────────┤  ...
    │          │                           │  page C  │  ◀── vmalloc page 3
    ├──────────┤                           ├──────────┤
    │          │                           │          │
    ├──────────┤                           ├──────────┤
    │          │                           │  page D  │  ◀── vmalloc page 4
    └──────────┘                           └──────────┘

    4 pages, physically adjacent           4 pages, physically scattered
    1 TLB entry can cover all              1 TLB entry needed per page
```


---

## Slab Allocator

```
  ┌─────────────────────────────────────────────────────────────────────────────┐
  │                           SLAB ALLOCATOR OVERVIEW                           │
  │                                                                             │
  │                         kmalloc(64)  called                                 │
  │                               │                                             │
  │                               ▼                                             │
  │                    ┌──────────────────────┐                                 │
  │                    │    size < 8KB ?      │                                 │
  │                    └──────────┬───────────┘                                 │
  │                               │                                             │
  │               ┌───────────────┴───────────────┐                             │
  │              YES                              NO                            │
  │               │                               │                             │
  │               ▼                               ▼                             │
  │  ┌────────────────────────┐     ┌──────────────────────────┐                │
  │  │     Slab Allocator     │     │     Page Allocator       │                │
  │  │   (kmalloc-N caches)   │     │    alloc_pages() direct  │                │
  │  │                        │     │    no slab involved      │                │
  │  │  size classes:         │     │                          │                │
  │  │  8, 16, 32, 64, 128,   │     │  8KB  → order=1 (2 pages)│                │
  │  │  256, 512, 1K, 2K, 4K  │     │  16KB → order=2 (4 pages)│                │
  │  └───────────┬────────────┘     │  32KB → order=3 (8 pages)│                │
  │              │                  └──────────────────────────┘                │
  │              ▼                                                              │
  │  ┌────────────────────────────────────────────────────────────────────┐     │
  │  │                         SLAB CACHE                                 │     │
  │  │                                                                    │     │
  │  │   kmem_cache (e.g. "kmalloc-64")                                   │     │
  │  │                                                                    │     │
  │  │   ┌──────────────────────────────────────────────────────────┐     │     │
  │  │   │  FULL SLABS — all objects in use                         │     │     │
  │  │   │                                                          │     │     │
  │  │   │  ┌────────────────────────────────────────────────────┐  │     │     │
  │  │   │  │ [USED][USED][USED][USED][USED][USED][USED][USED]   │  │     │     │
  │  │   │  └────────────────────────────────────────────────────┘  │     │     │
  │  │   └──────────────────────────────────────────────────────────┘     │     │
  │  │                                                                    │     │
  │  │   ┌───────────────────────────────────────────────────────────┐    │     │
  │  │   │  PARTIAL SLABS — mix of free and used  ◀── alloc from here│    │     │
  │  │   │                                                           │    │     │
  │  │   │  ┌────────────────────────────────────────────────────┐   │    │     │
  │  │   │  │ [USED][FREE][USED][FREE][FREE][USED][FREE][USED]   │   │    │     │
  │  │   │  └────────────────────────────────────────────────────┘   │    │     │
  │  │   │  ┌────────────────────────────────────────────────────┐   │    │     │
  │  │   │  │ [FREE][USED][FREE][USED][USED][FREE][USED][FREE]   │   │    │     │
  │  │   │  └────────────────────────────────────────────────────┘   │    │     │
  │  │   └───────────────────────────────────────────────────────────┘    │     │
  │  │                                                                    │     │
  │  │   ┌──────────────────────────────────────────────────────────┐     │     │
  │  │   │  EMPTY SLABS — all free (ready to return to page alloc)  │     │     │
  │  │   │                                                          │     │     │
  │  │   │  ┌────────────────────────────────────────────────────┐  │     │     │
  │  │   │  │ [FREE][FREE][FREE][FREE][FREE][FREE][FREE][FREE]   │  │     │     │
  │  │   │  └────────────────────────────────────────────────────┘  │     │     │
  │  │   └──────────────────────────────────────────────────────────┘     │     │
  │  │                                                                    │     │
  │  └────────────────────────────────────────────────────────────────────┘     │
  └─────────────────────────────────────────────────────────────────────────────┘
```

---
##  Decode `/proc/slabinfo`

```
  ┌─────────────────────────────────────────────────────────────────────────────┐
  │                          /proc/slabinfo  OUTPUT                             │
  │                                                                             │
  │  # name         <active_objs> <num_objs> <objsize> <objperslab<pagesperslab>│
  │  task_struct         512          512       7168         4            7     │
  │  inode_cache        8192         8192        624        26            4     │
  │  dentry            65536        65536        192        21            1     │
  │  kmalloc-64         4096         4096         64        64            1     │
  │  kmalloc-256        1024         1024        256        16            1     │
  │  sock_inode_cache   1024         1024       1216         6            2     │
  │                                                                             │
  └─────────────────────────────────────────────────────────────────────────────┘
                         │           │          │            │           │
                         ▼           ▼          ▼            ▼           ▼

                active_objs     num_objs    objsize   objperslab   pagesperslab
                ───────────    ─────────   ────────   ──────────   ──────────────
                currently      total       size of      how many     how many 4KB
                allocated      slots      1 object   objects fit   pages per slab
                  objects    created       (bytes)   in one slab
```


---

```
  ─────────────────────────────────────────────────────────────────────────────
    ALLOCATION FLOW
  ─────────────────────────────────────────────────────────────────────────────

    kmalloc(64) called
          │
          ▼
    partial slab has free slot?  ──YES──▶  return pointer instantly ✅
          │                               (no page allocator touched)
          NO
          │
          ▼
    empty slab available?  ──────YES──▶  move to partial, return slot ✅
          │
          NO
          │
          ▼
    ask page allocator for  ──────────▶  carve into 64B slots ──▶  return slot
    a new 4KB page
```

```
  ─────────────────────────────────────────────────────────────────────────────
    DE-ALLOCATION FLOW
  ─────────────────────────────────────────────────────────────────────────────

    kfree(ptr) called
          │
          ▼
    mark slot FREE in slab
          │
          ▼
    slab now empty?  ──YES──▶  optionally return page to page allocator
          │
          NO
          │
          ▼
    keep in partial list (ready for next alloc)
```


```    

  ─────────────────────────────────────────────────────────────────────────────
    SINGLE SLAB INTERNALS  (4KB page, object size = 64B → 64 objects per page)
  ─────────────────────────────────────────────────────────────────────────────

    ┌─────────────────────────────────────────────────────────────────────────┐
    │                         ONE SLAB  (= one 4KB page)                      │
    │                                                                         │
    │  ┌──────┬──────┬──────┬──────┬──────┬──────┬──────┬──────┐              │
    │  │ obj0 │ obj1 │ obj2 │ obj3 │ obj4 │ obj5 │ obj6 │ obj7 │  ...         │
    │  │ USED │ FREE │ USED │ FREE │ FREE │ USED │ FREE │ USED │              │
    │  └──────┴──────┴──────┴──────┴──────┴──────┴──────┴──────┘              │
    │     ▲                                                                   │
    │     └── each slot = 64B, pre-initialized, ready to hand out             │
    │                                                                         │
    │  freelist pointer:  obj1 → obj3 → obj4 → obj6 → NULL                    │
    │                     (linked list of free slots inside the slab)         │
    │                                                                         │
    └─────────────────────────────────────────────────────────────────────────┘
```

```
  ─────────────────────────────────────────────────────────────────────────────
    WHERE SLAB FITS IN THE MEMORY HIERARCHY
  ─────────────────────────────────────────────────────────────────────────────

    Hardware (Physical RAM)
          │
          ▼
    Buddy System / Page Allocator    ← allocates in PAGE units (4KB, 8KB...)
          │
          ▼
    Slab Allocator (SLUB default)    ← carves pages into small reusable slots
          │
          ├── named caches           ← task_struct, inode, socket, dentry...
          └── kmalloc-N caches       ← generic 8B ~ 4KB buckets
                │
                ▼
          kmalloc() / kmem_cache_alloc()
                │
                ▼
          Kernel subsystems          ← drivers, filesystems, networking...
```

##  Decode `/proc/buddinfo`

```
 $ cat /proc/buddyinfo

Node 0, zone      DMA      1    0    1    0    2    1    1    0    1    1    3
Node 0, zone    DMA32    760  552  183   75   33   13    5    2    1    1    3
Node 0, zone   Normal   1234  891  400  210   85   30   12    4    2    1    2
                           │    │    │    │    │    │    │    │    │    │    │
		                 2^0  2^1  2^2  2^3  2^4  2^5  2^6  2^7  2^8  2^9  2^10
	                     4K   8K   16K  32K  64K  128K 256K 512K  1M   2M   4M
                         pages free at each order

Each number = count of free physically contiguous blocks at that order.
```


```
  ─────────────────────────────────────────────────────────────────────────────
    NAMED CACHES vs GENERIC kmalloc CACHES
  ─────────────────────────────────────────────────────────────────────────────

    Generic (kmalloc)                   Named (per object type)
    ─────────────────────────────       ─────────────────────────────
    kmalloc-8                           task_struct cache
    kmalloc-16                          inode_cache
    kmalloc-32                          dentry cache
    kmalloc-64                          sock_inode_cache
    kmalloc-128                         ext4_inode_cache
    kmalloc-256                         anon_vma cache
    kmalloc-512                         vm_area_struct cache
    kmalloc-1024                        signal_cache
    kmalloc-2048                        files_cache
    kmalloc-4096
         │                                      │
         ▼                                      ▼
    any ad-hoc allocation              high-frequency same-type allocs
    no constructor                     supports ctor / dtor
    e.g. temporary buffers             e.g. every new process, file, socket
```

