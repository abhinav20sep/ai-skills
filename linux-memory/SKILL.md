---
name: linux-memory
description: Linux memory management for kernel and userspace in C, Rust, and Zig. Covers kernel allocators (kmalloc, vmalloc, slab caches, mempools, GFP flags), managed allocations (devm_*), DMA mapping API, userspace mmap, memory barriers, and OOM handling. Use when working with device drivers, DMA buffers, shared memory, NUMA, or diagnosing memory issues.
---

# Linux Memory Management

Kernel and userspace memory management patterns for C, Rust, and Zig.

## Step 1: Detect context

- Kernel module code (includes `linux/module.h`) → kernel workflow
- Userspace code → userspace workflow
- Mixed → run each workflow for its files

## Step 2: Kernel memory workflow

### kmalloc / kfree — small allocations (< 128KB, physically contiguous)

```c
#include <linux/slab.h>

/* Basic allocation */
void *buf = kmalloc(1024, GFP_KERNEL);
if (!buf)
	return -ENOMEM;

/* Zeroed allocation */
void *buf = kzalloc(1024, GFP_KERNEL);

/* Array allocation with overflow check */
int *arr = kmalloc_array(count, sizeof(int), GFP_KERNEL);
if (!arr)
	return -ENOMEM;

/* Free */
kfree(buf);
buf = NULL;  /* prevent use-after-free */
```

### GFP flags — allocation context rules

| Flag | Context | Behavior |
|---|---|---|
| `GFP_KERNEL` | Process context only | Can sleep, can do I/O, can reclaim |
| `GFP_ATOMIC` | Any context (IRQ, spinlock) | Never sleeps, uses emergency reserves |
| `GFP_NOIO` | Process context, I/O path | Can sleep, cannot do I/O (avoids recursion) |
| `GFP_NOFS` | Process context, FS path | Can sleep, cannot do FS operations |
| `GFP_DMA` | Any | Allocate from DMA zone (< 16MB ISA) |
| `GFP_DMA32` | Any | Allocate from DMA32 zone (< 4GB) |
| `GFP_USER` | Process context | For userspace pages, can be swapped |
| `__GFP_NOFAIL` | Process context | Never fails, retries forever (use sparingly) |
| `GFP_NOWAIT` | Any | Never sleeps, no I/O, no reclaim |

**Decision tree**:
```
In IRQ / spinlock / atomic context?
  YES → GFP_ATOMIC
  NO → Holding a mutex? In I/O path?
    YES in I/O path → GFP_NOIO
    YES in FS path → GFP_NOFS
    NO → GFP_KERNEL
Need DMA-capable memory?
  Add GFP_DMA or GFP_DMA32
```

### vmalloc — large allocations (not physically contiguous)

```c
#include <linux/vmalloc.h>

/* Allocate virtual memory (pages need not be physically contiguous) */
void *buf = vmalloc(1024 * 1024);  /* 1MB */
if (!buf)
	return -ENOMEM;

/* Zeroed */
void *buf = vzalloc(1024 * 1024);

vfree(buf);
```

**When to use vmalloc vs kmalloc**:

| Criterion | kmalloc | vmalloc |
|---|---|---|
| Physically contiguous | Yes | No |
| Max size | ~128KB (order-5 page) | Limited by virtual address space |
| DMA-compatible | Yes | No |
| Performance | Better (TLB-friendly) | Worse (page table walks) |
| Use case | Small buffers, DMA | Large buffers, code loading |

### Slab caches — frequently allocated structs

```c
/* Create a cache for a frequently allocated struct */
struct kmem_cache *my_cache = kmem_cache_create(
	"my_struct",           /* name (visible in /proc/slabinfo) */
	sizeof(struct my_struct),  /* object size */
	0,                     /* alignment (0 = default) */
	SLAB_HWCACHE_ALIGN,    /* flags: align to cache line */
	NULL                   /* constructor */
);

/* Allocate from cache */
struct my_struct *obj = kmem_cache_alloc(my_cache, GFP_KERNEL);

/* Free back to cache */
kmem_cache_free(my_cache, obj);

/* Destroy cache (on module exit) */
kmem_cache_destroy(my_cache);
```

### mempool — guaranteed allocation pool

```c
#include <linux/mempool.h>

/* Create a pool that pre-allocates emergency objects */
mempool_t *pool = mempool_create(32,              /* min_nr pre-allocated */
                                  mempool_kmalloc, /* alloc function */
                                  mempool_kfree,   /* free function */
                                  (void *)1024);   /* pool_data (size for kmalloc) */

/* Allocate (will not fail if pool has objects) */
void *buf = mempool_alloc(pool, GFP_NOIO);

/* Free (returns to pool, may wake waiters) */
mempool_free(buf, pool);

/* Destroy */
mempool_destroy(pool);
```

**Use mempool when**: allocation failure is unacceptable (e.g., block layer I/O requests).

### devm_* — managed allocations (auto-freed on device unbind)

```c
#include <linux/device.h>

/* Allocated and automatically freed when device is removed */
void *buf = devm_kzalloc(dev, 1024, GFP_KERNEL);
if (!buf)
	return -ENOMEM;

/* Managed I/O memory mapping */
void __iomem *regs = devm_ioremap_resource(dev, res);
if (IS_ERR(regs))
	return PTR_ERR(regs);

/* Managed interrupt */
int ret = devm_request_irq(dev, irq, my_handler, 0, "mydev", priv);

/* Managed clock */
struct clk *clk = devm_clk_get(dev, NULL);

/* All devm_* resources are freed automatically on:
   - device unbind
   - probe() failure (partial probe cleanup)
   - module unload
   No explicit free needed in remove() or error paths.
```

**Rule**: If using `devm_*`, you do NOT need to call `kfree`, `iounmap`, `free_irq`, etc. in your `remove()` function or error paths.

### Kernel allocation summary

```
Small (< 128KB), need physical contiguity? → kmalloc/kzalloc
Small, frequently allocated struct? → kmem_cache_create + kmem_cache_alloc
Large (> 128KB), physical contiguity not needed? → vmalloc/vzalloc
Must not fail? → mempool_create + mempool_alloc
Device-managed? → devm_kzalloc (auto-cleanup)
DMA-capable? → kmalloc + GFP_DMA/DMA32, or dma_alloc_coherent
```

## Step 3: Userspace memory workflow

### mmap — memory-mapped I/O

```c
#include <sys/mman.h>

/* Anonymous mmap (not backed by file) */
void *ptr = mmap(NULL, 4096,
                 PROT_READ | PROT_WRITE,
                 MAP_PRIVATE | MAP_ANONYMOUS,
                 -1, 0);
if (ptr == MAP_FAILED) {
    perror("mmap");
    return -1;
}

/* File-backed mmap */
int fd = open("data.bin", O_RDWR);
void *ptr = mmap(NULL, file_size,
                 PROT_READ | PROT_WRITE,
                 MAP_SHARED,  /* MAP_SHARED: changes visible to other processes */
                 fd, 0);      /* MAP_PRIVATE: copy-on-write */

/* Modify and sync */
*(int *)ptr = 42;
msync(ptr, file_size, MS_SYNC);  /* flush to disk */

/* Unmap */
munmap(ptr, 4096);
```

### MAP_PRIVATE vs MAP_SHARED

| Flag | Writes | Visibility | Persistence | Use case |
|---|---|---|---|---|
| `MAP_SHARED` | Propagated to file | Visible to all mappers | Written to file on msync/exit | Shared memory, IPC |
| `MAP_PRIVATE` | Copy-on-write | Private to this process | NOT written to file | Read-only file access, loading |

### mprotect — change page permissions

```c
/* Mark pages read-only */
mprotect(ptr, 4096, PROT_READ);

/* Mark pages no-access (guard page) */
mprotect(ptr, 4096, PROT_NONE);

/* Mark pages executable (for JIT, use with caution) */
mprotect(ptr, 4096, PROT_READ | PROT_EXEC);
```

### mlock — pin pages in physical memory

```c
/* Lock pages (prevent swapping) */
mlock(ptr, 4096);        /* per-process limit: RLIMIT_MEMLOCK */
mlockall(MCL_CURRENT | MCL_FUTURE);  /* lock all current + future pages */

/* Unlock */
munlock(ptr, 4096);
munlockall();

/* Check limits */
ulimit -l  # RLIMIT_MEMLOCK in KB
```

### posix_memalign — aligned allocation

```c
void *buf;
int ret = posix_memalign(&buf, 64, 4096);  /* 64-byte aligned, 4096 bytes */
if (ret != 0) {
    errno = ret;
    perror("posix_memalign");
}
free(buf);
```

### NUMA-aware allocation

```c
#include <numa.h>
#include <numaif.h>

/* Set memory policy for future allocations */
unsigned long nodemask = 1 << 0;  /* node 0 */
set_mempolicy(MPOL_BIND, &nodemask, sizeof(nodemask) * 8);

/* Or: interleave across nodes (good for shared data) */
set_mempolicy(MPOL_INTERLEAVE, &nodemask, sizeof(nodemask) * 8);

/* Allocate on specific node */
void *ptr = numa_alloc_onnode(4096, 1);  /* 4096 bytes on node 1 */
numa_free(ptr, 4096);

/* Check NUMA topology */
numactl --hardware
numactl --membind=0 ./my_app  /* force all memory to node 0 */
```

### Rust userspace memory

```rust
use std::alloc::{alloc, dealloc, Layout};
use memmap2::MmapMut;
use std::fs::OpenOptions;

// Aligned allocation
let layout = Layout::from_size_align(4096, 64).unwrap();
let ptr = unsafe { alloc(layout) };
// ... use ptr ...
unsafe { dealloc(ptr, layout) };

// Memory-mapped file
let file = OpenOptions::new().read(true).write(true).open("data.bin")?;
let mut mmap = unsafe { MmapMut::map_mut(&file)? };
mmap[0] = 42;
mmap.flush()?;
```

See `DMA.md` for DMA mapping and `BARRIERS.md` for memory barriers.

## Step 4: Memory diagnostics

```bash
# System memory usage
free -h
cat /proc/meminfo

# Per-process memory
cat /proc/<pid>/status | grep -i vm
cat /proc/<pid>/smaps_rollup

# Kernel slab allocator
cat /proc/slabinfo
slabtop

# Page allocator fragmentation
cat /proc/buddyinfo
cat /proc/pagetypeinfo

# Kernel memory leak detection (CONFIG_DEBUG_KMEMLEAK)
echo scan > /sys/kernel/debug/kmemleak
cat /sys/kernel/debug/kmemleak

# Page owner tracking (CONFIG_PAGE_OWNER)
echo 1 > /sys/kernel/page_owner
cat /sys/kernel/debug/page_owner | head -50
```

## Step 5: Verify

- [ ] All `kmalloc`/`vmalloc` have corresponding `kfree`/`vfree`
- [ ] All `mmap` have corresponding `munmap`
- [ ] No use-after-free (set pointer to NULL after free)
- [ ] GFP flags match allocation context (GFP_KERNEL in process, GFP_ATOMIC in IRQ)
- [ ] `devm_*` used where possible (auto-cleanup on probe failure)
- [ ] No memory leaks under fault injection
- [ ] `/proc/slabinfo` stable under load (no slab growth)
