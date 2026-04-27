# Memory Barriers Reference

Linux kernel memory barriers for ordering memory accesses across CPUs and devices.

## Why barriers are needed

Modern CPUs and compilers reorder memory accesses for performance. Barriers prevent reordering when it would cause correctness issues.

**CPU reordering**: the CPU may execute loads/stores out of order. On x86, stores can be reordered with loads (store-buffer forwarding). On ARM/POWER, loads and stores can be freely reordered.

**Compiler reordering**: the compiler may reorder code if it sees no data dependency. Barriers tell the compiler not to move accesses across the barrier.

## Compiler barrier

```c
/* Prevent compiler from reordering across this point */
barrier();

/* Example: prevent compiler from caching a shared variable */
int flag = 0;
while (!flag) {
    barrier();  /* re-read flag from memory each iteration */
}
```

`barrier()` is a **compiler-only** barrier. It does NOT emit CPU instructions. It only prevents the compiler from reordering or optimizing away memory accesses.

## CPU memory barriers

### Full barrier

```c
/* Full barrier: no loads or stores reordered across this point */
mb();    /* CPU instruction (mfence on x86, dmb ish on ARM) */

/* SMP variant: only orders across CPUs, not device I/O */
smp_mb();
```

### Read barrier

```c
/* Read barrier: no loads reordered across this point */
rmb();   /* lfence on x86, dmb ishld on ARM */
smp_rmb();
```

### Write barrier

```c
/* Write barrier: no stores reordered across this point */
wmb();   /* sfence on x86, dmb ishst on ARM */
smp_wmb();
```

### I/O barrier

```c
/* I/O barrier: orders MMIO accesses */
mb();    /* full barrier includes I/O ordering */
wmb();   /* some architectures need separate I/O barriers */
```

## Barrier decision tree

```
What are you ordering?
├── CPU ↔ CPU (SMP)?
│   ├── Loads only → smp_rmb()
│   ├── Stores only → smp_wmb()
│   └── Both → smp_mb()
├── CPU ↔ Device (MMIO)?
│   ├── Writes to device → wmb() or mmiowb()
│   ├── Reads from device → rmb()
│   └── Both → mb()
└── Compiler only (single CPU, no device)?
    └── barrier()
```

## SMP barriers — most common in kernel code

`smp_*` variants are no-ops on UP (uniprocessor) kernels. They only emit instructions on SMP systems. Use these for CPU-to-CPU synchronization.

```c
/* Producer-consumer pattern */
/* CPU 0 (producer):                CPU 1 (consumer): */
data = 42;                          while (!flag) cpu_relax();
smp_wmb();                          smp_rmb();
flag = 1;                           val = data;  /* guaranteed to see 42 */
```

## READ_ONCE / WRITE_ONCE

Prevent the compiler from tearing, caching, or optimizing away accesses to shared variables.

```c
/* Write: prevent store tearing and compiler optimization */
WRITE_ONCE(shared_flag, 1);

/* Read: prevent load tearing and compiler caching */
int val = READ_ONCE(shared_flag);
```

**When to use**:
- Shared variables accessed without locks
- Flag variables (done, ready, shutdown)
- Any variable written by one CPU and read by another

**What they prevent**:
- Store tearing: a 64-bit write split into two 32-bit writes
- Load tearing: a 64-bit read split into two 32-bit reads
- Compiler caching: compiler keeps value in register instead of re-reading
- Compiler reordering: compiler moves the access

## smp_store_release / smp_load_acquire

Release-acquire ordering. A weaker but more efficient alternative to full barriers for many patterns.

```c
/* Release: all prior accesses are visible before this store */
smp_store_release(&flag, 1);

/* Acquire: all subsequent accesses see the released value */
int val = smp_load_acquire(&flag);
if (val) {
    /* data is guaranteed to be visible */
    int data_val = data;  /* safe to read */
}
```

**Equivalent to**:
```c
/* smp_store_release(&flag, 1) is equivalent to: */
smp_mb();          /* full barrier before store */
WRITE_ONCE(flag, 1);

/* smp_load_acquire(&flag) is equivalent to: */
int val = READ_ONCE(flag);
smp_mb();          /* full barrier after load */
```

But `smp_store_release`/`smp_load_acquire` may be more efficient on weakly-ordered architectures (ARM, POWER).

## Common barrier patterns

### Spinlock/unlock

```c
spin_lock(&lock);
/* All accesses inside spinlock are ordered.
   spin_lock implies acquire barrier.
   spin_unlock implies release barrier. */
data = new_value;
spin_unlock(&lock);
```

### Flag + data (producer-consumer)

```c
/* Producer (CPU 0) */
data = payload;
smp_store_release(&ready, 1);  /* or: smp_wmb(); WRITE_ONCE(ready, 1); */

/* Consumer (CPU 1) */
while (!smp_load_acquire(&ready))  /* or: while (!READ_ONCE(ready)) smp_rmb(); */
    cpu_relax();
/* data is now visible */
```

### Circular buffer

```c
/* Producer */
buf[write_idx] = data;
smp_wmb();
write_idx = (write_idx + 1) % BUF_SIZE;

/* Consumer */
idx = read_idx;
smp_rmb();
data = buf[idx];
read_idx = (idx + 1) % BUF_SIZE;
```

### Device register ordering

```c
/* Write to device registers in order */
writel(control, regs + REG_CONTROL);   /* control first */
wmb();                                 /* ensure control is visible */
writel(data, regs + REG_DATA);         /* then data */

/* Read status then data */
status = readl(regs + REG_STATUS);
rmb();                                 /* ensure data register is read after status */
data = readl(regs + REG_DATA);
```

## barrier() vs smp_mb() vs mb()

| Barrier | Compiler | CPU (SMP) | CPU (device) | UP kernel |
|---|---|---|---|---|
| `barrier()` | Yes | No | No | Yes |
| `smp_mb()` | Yes | Yes | No | No (nop) |
| `mb()` | Yes | Yes | Yes | Yes |

**Use `smp_*` for CPU synchronization. Use `mb()`/`wmb()`/`rmb()` for device MMIO.**

## When NOT to use barriers

- **Locks already provide ordering**: `spin_lock`/`spin_unlock`, `mutex_lock`/`mutex_unlock` all include the necessary barriers. Don't add extra barriers inside locked sections.
- **Atomic operations provide ordering**: `atomic_inc`, `atomic_dec_and_test`, etc. have implied barriers (check the specific function's documentation).
- **Single-CPU systems**: `smp_*` barriers are nops. Only compiler barriers (`barrier()`, `READ_ONCE`, `WRITE_ONCE`) matter.
- **RCU provides ordering**: `rcu_read_lock`/`rcu_dereference`/`rcu_assign_pointer` handle their own ordering.

## Reference

| API | Header | Purpose |
|---|---|---|
| `barrier()` | `linux/compiler.h` | Compiler barrier only |
| `mb()` | `asm/barrier.h` | Full CPU + device barrier |
| `rmb()` | `asm/barrier.h` | Read barrier |
| `wmb()` | `asm/barrier.h` | Write barrier |
| `smp_mb()` | `asm/barrier.h` | Full SMP barrier (nop on UP) |
| `smp_rmb()` | `asm/barrier.h` | SMP read barrier |
| `smp_wmb()` | `asm/barrier.h` | SMP write barrier |
| `mmiowb()` | `asm/io.h` | MMIO write barrier |
| `READ_ONCE(x)` | `linux/compiler.h` | Volatile read, prevent tearing |
| `WRITE_ONCE(x, v)` | `linux/compiler.h` | Volatile write, prevent tearing |
| `smp_store_release(p, v)` | `asm/barrier.h` | Store with release semantics |
| `smp_load_acquire(p)` | `asm/barrier.h` | Load with acquire semantics |
