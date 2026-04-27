# Kernel Sanitizers Reference

KASAN, KCSAN, and KMSAN — runtime bug detectors for the Linux kernel.

## KASAN (Kernel Address Sanitizer)

Detects use-after-free, buffer overflows, out-of-bounds access.

### Setup

```bash
# Kernel config
CONFIG_KASAN=y
CONFIG_KASAN_INLINE=y    # faster than CONFIG_KASAN_OUTLINE
CONFIG_KASAN_GENERIC=y   # or CONFIG_KASAN_SW_TAGS for ARM64

# Boot and check
dmesg | grep kasan
```

### Reading KASAN reports

```
==================================================================
BUG: KASAN: use-after-free in my_driver_read+0x42/0x1a0 [mydriver]
Read of size 4 at addr ffff888003a40010 by task myapp/1234

CPU: 3 PID: 1234 Comm: myapp Tainted: G        W  O      5.15.0
Call Trace:
 dump_stack+0x64/0x80
 print_address_description.constprop.0+0x1f/0x200
 kasan_report.cold+0x7b/0x94
 my_driver_read+0x42/0x1a0 [mydriver]
 vfs_read+0x9e/0x1a0

Allocated by task 1230:
 kmem_cache_alloc_node+0xd3/0x260
 my_driver_probe+0x89/0x200 [mydriver]

Freed by task 1235:
 kfree+0x8c/0x200
 my_driver_remove+0x34/0x80 [mydriver]
```

**Key fields**:
- `BUG: KASAN: <type> in <function>` — what happened and where
- `Read/Write of size N` — the access that triggered the report
- `addr ffff888003a40010` — the memory address accessed
- `Allocated by` — who allocated this memory (with stack trace)
- `Freed by` — who freed this memory (for use-after-free)

**Common KASAN types**:

| Type | Meaning |
|---|---|
| `use-after-free` | Accessing memory after `kfree()` |
| `out-of-bounds` | Accessing past the end of an allocation |
| `slab-out-of-bounds` | Accessing past slab object boundary |
| `stack-out-of-bounds` | Accessing past stack variable boundary |
| `global-out-of-bounds` | Accessing past global variable boundary |
| `use-after-scope` | Accessing stack variable after its scope ends |

### Enabling at runtime

```bash
# Enable KASAN for a specific module
echo 1 > /sys/kernel/debug/kasan/quarantine_size

# Check quarantine (delayed free pool)
cat /sys/kernel/debug/kasan/quarantine_size
```

### Memory overhead

KASAN adds ~1/8 memory overhead (shadow memory). For 4GB RAM, expect ~500MB extra usage.

## KCSAN (Kernel Concurrency Sanitizer)

Detects data races — concurrent accesses to the same memory without proper synchronization.

### Setup

```bash
# Kernel config
CONFIG_KCSAN=y
CONFIG_KCSAN_REPORT_VALUE_CHANGE_ONLY=y   # reduce noise
CONFIG_KCSAN_WEAK_MEMORY=y                # detect weak memory ordering bugs

# Boot and check
dmesg | grep kcsan
```

### Reading KCSAN reports

```
==================================================================
BUG: KCSAN: data-race in my_driver_read / my_driver_write

write to 0xffff888003a40020 of 4 bytes by task 1234 on cpu 3:
 my_driver_write+0x56/0x120 [mydriver]
 vfs_write+0x1a0/0x2c0

read to 0xffff888003a40020 of 4 bytes by task 1235 on cpu 1:
 my_driver_read+0x34/0x1a0 [mydriver]
 vfs_read+0x9e/0x1a0

Reported by Kernel Concurrency Sanitizer on:
CPU: 1 PID: 1235 Comm: reader Tainted: G        W  O
```

**Key fields**:
- `write to <addr>` / `read to <addr>` — the conflicting accesses
- `by task <pid> on cpu <n>` — which threads on which CPUs
- `my_driver_write` / `my_driver_read` — the functions performing the access

### Fixing data races

```c
/* BEFORE: data race */
static int shared_counter;

/* Thread 1 */
shared_counter++;

/* Thread 2 */
int val = shared_counter;  /* race! */

/* AFTER: option 1 — mutex */
static DEFINE_MUTEX(counter_lock);
mutex_lock(&counter_lock);
shared_counter++;
mutex_unlock(&counter_lock);

/* AFTER: option 2 — atomic */
static atomic_t shared_counter = ATOMIC_INIT(0);
atomic_inc(&shared_counter);
int val = atomic_read(&shared_counter);

/* AFTER: option 3 — READ_ONCE/WRITE_ONCE for flags */
static int done;
WRITE_ONCE(done, 1);
while (!READ_ONCE(done)) cpu_relax();
```

## KMSAN (Kernel Memory Sanitizer)

Detects use of uninitialized memory.

### Setup

```bash
# Kernel config
CONFIG_KMSAN=y
# Note: KMSAN requires Clang (not GCC)
# Build with: make CC=clang

# Boot and check
dmesg | grep kmsan
```

### Reading KMSAN reports

```
==================================================================
BUG: KMSAN: uninit-value in my_driver_read+0x42/0x1a0 [mydriver]
CPU: 3 PID: 1234 Comm: myapp Tainted: G        W  O      5.15.0

Call Trace:
 dump_stack+0x64/0x80
 kmsan_report+0xfb/0x1e0
 my_driver_read+0x42/0x1a0 [mydriver]
 vfs_read+0x9e/0x1a0

Uninit was created at:
 kmem_cache_alloc+0x2c0/0x400
 my_driver_probe+0x89/0x200 [mydriver]
```

**Key fields**:
- `uninit-value in <function>` — where uninitialized memory was used
- `Uninit was created at` — where the uninitialized allocation happened

### Common causes

```c
/* BEFORE: uninitialized field */
struct my_dev *dev = kzalloc(sizeof(*dev), GFP_KERNEL);
dev->status = ACTIVE;
/* dev->count is uninitialized! */

/* AFTER: use kzalloc to zero-initialize */
struct my_dev *dev = kzalloc(sizeof(*dev), GFP_KERNEL);
/* All fields are zero now */
dev->status = ACTIVE;
```

## kmemleak — kernel memory leak detector

```bash
# Setup (CONFIG_DEBUG_KMEMLEAK=y)
# Enable scanning
echo scan > /sys/kernel/debug/kmemleak

# Trigger a scan
echo scan > /sys/kernel/debug/kmemleak

# Read results
cat /sys/kernel/debug/kmemleak

# Clear existing leaks (after fixing)
echo clear > /sys/kernel/debug/kmemleak
```

**Report format**:
```
unreferenced object 0xffff888003a40000 (size 256):
  comm "insmod", pid 1234, jiffies 4294967295
  hex dump (first 32 bytes):
    00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
  backtrace:
    [<0000000012345678>] kmem_cache_alloc+0xd3/0x260
    [<00000000abcdef01>] my_driver_probe+0x89/0x200 [mydriver]
```

## Choosing a sanitizer

| Bug type | Sanitizer | Config | Overhead |
|---|---|---|---|
| Use-after-free, buffer overflow | **KASAN** | `CONFIG_KASAN=y` | ~2x memory, ~3x slower |
| Data races | **KCSAN** | `CONFIG_KCSAN=y` | ~5x slower |
| Uninitialized memory | **KMSAN** | `CONFIG_KMSAN=y` | ~3x memory, ~5x slower |
| Memory leaks | **kmemleak** | `CONFIG_DEBUG_KMEMLEAK=y` | low |
| Slab corruption | **SLUB debug** | `slub_debug=FZPU` | low-medium |

## Running multiple sanitizers

You can enable KASAN + KCSAN + kmemleak simultaneously. KMSAN is incompatible with KASAN (both use shadow memory, different layouts).

```bash
# Recommended for development builds
CONFIG_KASAN=y
CONFIG_KCSAN=y
CONFIG_DEBUG_KMEMLEAK=y
CONFIG_KASAN_INLINE=y
CONFIG_KCSAN_REPORT_VALUE_CHANGE_ONLY=y
```

For KMSAN, disable KASAN:
```bash
# KMSAN build (requires Clang)
# CONFIG_KASAN is not set
CONFIG_KMSAN=y
```
