---
name: linux-threading
description: Multithreading patterns for Linux in C (pthreads), Rust (std::thread, Arc, Mutex, channels), and Zig (std.Thread). Covers thread creation, synchronization primitives, thread pools, lock-free patterns, and common concurrency bugs. Use when user wants to add concurrency, fix race conditions, implement thread pools, or reason about shared state.
---

# Linux Threading Patterns

**Core principle**: Shared mutable state is the root of all concurrency bugs. Every access to shared data must be protected by a lock, an atomic, or a channel. If you can't identify the synchronization mechanism for a shared variable, you have a bug.

Multithreading patterns for C (pthreads), Rust, and Zig on Linux.

## Step 1: Detect language

Check file extensions and build files:
- `*.c`, `*.h`, `Makefile` → C (pthreads) workflow
- `*.rs`, `Cargo.toml` → Rust workflow
- `*.zig`, `build.zig` → Zig workflow

## Step 2: C (pthreads) workflow

### Thread creation and joining

```c
#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>

struct thread_arg {
    int id;
    void *data;
};

static void *thread_func(void *arg)
{
    struct thread_arg *ta = arg;
    printf("Thread %d running\n", ta->id);
    /* ... work ... */
    return NULL;  /* or return result pointer */
}

/* Create and join */
pthread_t tid;
struct thread_arg ta = { .id = 1, .data = NULL };
pthread_create(&tid, NULL, thread_func, &ta);
pthread_join(tid, NULL);  /* or &retval for return value */
```

### Mutex (pthread_mutex_t)

```c
pthread_mutex_t lock = PTHREAD_MUTEX_INITIALIZER;

pthread_mutex_lock(&lock);
/* critical section — access shared data */
pthread_mutex_unlock(&lock);

/* Destroy when done */
pthread_mutex_destroy(&lock);
```

**Error-checking mutex** (catches double-lock, unlock-from-wrong-thread):
```c
pthread_mutexattr_t attr;
pthread_mutexattr_init(&attr);
pthread_mutexattr_settype(&attr, PTHREAD_MUTEX_ERRORCHECK);
pthread_mutex_init(&lock, &attr);
```

### Read-write lock (pthread_rwlock_t)

```c
pthread_rwlock_t rwlock = PTHREAD_RWLOCK_INITIALIZER;

/* Readers (concurrent) */
pthread_rwlock_rdlock(&rwlock);
/* read shared data */
pthread_rwlock_unlock(&rwlock);

/* Writer (exclusive) */
pthread_rwlock_wrlock(&rwlock);
/* modify shared data */
pthread_rwlock_unlock(&rwlock);
```

### Condition variables (pthread_cond_t)

```c
pthread_mutex_t lock = PTHREAD_MUTEX_INITIALIZER;
pthread_cond_t cond = PTHREAD_COND_INITIALIZER;
int ready = 0;

/* Waiting thread */
pthread_mutex_lock(&lock);
while (!ready)              /* always use while, not if */
    pthread_cond_wait(&cond, &lock);  /* releases lock, sleeps, re-acquires */
/* process when ready */
pthread_mutex_unlock(&lock);

/* Signaling thread */
pthread_mutex_lock(&lock);
ready = 1;
pthread_cond_signal(&cond);      /* wake one waiter */
/* or pthread_cond_broadcast(&cond);  wake all waiters */
pthread_mutex_unlock(&lock);
```

### Thread pool pattern

```c
#include <pthread.h>

#define MAX_THREADS 8
#define MAX_QUEUE   256

typedef void (*task_fn)(void *arg);

struct thread_pool {
    pthread_t       threads[MAX_THREADS];
    task_fn         queue[MAX_QUEUE];
    void           *queue_arg[MAX_QUEUE];
    int             head, tail, count;
    int             shutdown;
    pthread_mutex_t lock;
    pthread_cond_t  not_empty;
    pthread_cond_t  not_full;
};

static void *worker(void *arg)
{
    struct thread_pool *pool = arg;
    for (;;) {
        pthread_mutex_lock(&pool->lock);
        while (pool->count == 0 && !pool->shutdown)
            pthread_cond_wait(&pool->not_empty, &pool->lock);
        if (pool->shutdown && pool->count == 0) {
            pthread_mutex_unlock(&pool->lock);
            break;
        }
        task_fn fn = pool->queue[pool->head];
        void *fn_arg = pool->queue_arg[pool->head];
        pool->head = (pool->head + 1) % MAX_QUEUE;
        pool->count--;
        pthread_cond_signal(&pool->not_full);
        pthread_mutex_unlock(&pool->lock);
        fn(fn_arg);
    }
    return NULL;
}
```

### Common C threading bugs

| Bug | Detection | Fix |
|---|---|---|
| Data race | `ThreadSanitizer` (`-fsanitize=thread`) | Add mutex or use atomics |
| Deadlock | Lock ordering, lockdep (kernel), TSAN | Consistent lock ordering, `trylock` |
| Lost wakeup | `pthread_cond_signal` outside lock | Always signal inside mutex |
| Spurious wakeup | `if` instead of `while` on condvar | Always use `while` loop |
| Double-free | TSAN, valgrind | Set pointer to NULL after free |
| Use-after-join | Accessing thread-local data after `pthread_join` | Copy result before join returns |

### Compile with threading support

```bash
gcc -pthread -o app app.c       # -lpthread also works
# With sanitizer:
gcc -pthread -fsanitize=thread -o app app.c
```

## Step 3: Rust workflow

### Thread creation

```rust
use std::thread;

let handle = thread::spawn(|| {
    // work in new thread
    42
});

let result = handle.join().unwrap(); // blocks until thread finishes
```

### Shared state: Arc + Mutex

```rust
use std::sync::{Arc, Mutex};
use std::thread;

let counter = Arc::new(Mutex::new(0));
let mut handles = vec![];

for _ in 0..10 {
    let counter = Arc::clone(&counter);
    let handle = thread::spawn(move || {
        let mut num = counter.lock().unwrap();
        *num += 1;
    });
    handles.push(handle);
}

for handle in handles {
    handle.join().unwrap();
}

println!("Result: {}", *counter.lock().unwrap());
```

**What the compiler sees**: `Arc<Mutex<T>>` is a heap-allocated reference-counted mutex. `counter.lock()` returns a `MutexGuard<T>` that implements `DerefMut` — auto-unlocks on drop.

### Read-write lock: Arc + RwLock

```rust
use std::sync::{Arc, RwLock};

let data = Arc::new(RwLock::new(vec![1, 2, 3]));

// Readers (concurrent)
let data_clone = Arc::clone(&data);
let reader = thread::spawn(move || {
    let val = data_clone.read().unwrap(); // blocks if writer holds lock
    println!("{:?}", *val);
});

// Writer (exclusive)
let mut val = data.write().unwrap();
val.push(4);
```

### Channels (message passing)

```rust
use std::sync::mpsc;
use std::thread;

let (tx, rx) = mpsc::channel();

thread::spawn(move || {
    tx.send("hello from thread").unwrap();
});

let msg = rx.recv().unwrap();
println!("{}", msg);

// Multi-producer: clone tx
let tx2 = tx.clone();
```

### Atomics

```rust
use std::sync::atomic::{AtomicUsize, Ordering};
use std::sync::Arc;
use std::thread;

let counter = Arc::new(AtomicUsize::new(0));
let counter_clone = Arc::clone(&counter);

thread::spawn(move || {
    counter_clone.fetch_add(1, Ordering::SeqCst);
});

// Relaxed: no ordering guarantees
// Acquire/Release: sync point
// SeqCst: full sequential consistency (safest, slowest)
```

### Rayon (data parallelism)

```toml
[dependencies]
rayon = "1"
```

```rust
use rayon::prelude::*;

let sum: i64 = (0..1_000_000)
    .into_par_iter()
    .map(|x| x * x)
    .sum();
```

### Common Rust concurrency bugs

| Bug | Detection | Fix |
|---|---|---|
| Deadlock | Manual review, `parking_lot` deadlock detector | Consistent lock ordering |
| Lock poisoning | `.unwrap()` on poisoned mutex | Use `parking_lot::Mutex` (no poisoning) |
| Excessive contention | `perf top` showing lock wait | Reduce lock scope, use `RwLock`, sharding |
| Send/Sync violation | Compile error | Use `Arc` not `Rc` across threads |

## Step 4: Zig workflow

### Thread creation

```zig
const std = @import("std");

fn worker(_: void) void {
    std.debug.print("thread running\n", .{});
}

const thread = try std.Thread.spawn(.{}, worker, .{});
thread.join();
```

### Mutex

```zig
var mutex: std.Thread.Mutex = .{};
var condition: std.Thread.Condition = .{};

mutex.lock();
defer mutex.unlock();

// With condition variable:
while (!ready) {
    condition.wait(&mutex);
}
```

### Atomics

```zig
const Atomic = std.atomic.Value;

var counter: Atomic(u32) = Atomic(u32).init(0);

_ = counter.fetchAdd(1, .seq_cst);
const val = counter.load(.seq_cst);
```

### Channels (via std)

```zig
const Channel = std.Channel;

var chan = Channel(i32).init(.{});
try chan.send(42);
const val = try chan.recv();
```

### Compile with threads

```zig
// build.zig
const exe = b.addExecutable(.{
    .name = "app",
    .root_source_file = b.path("src/main.zig"),
    .target = target,
    .optimize = optimize,
});
exe.linkLibC(); // needed for pthreads on some targets
```

## Anti-patterns

- **Double-locking** — Never acquire the same mutex twice in the same thread (unless it's `PTHREAD_MUTEX_RECURSIVE`). Use `PTHREAD_MUTEX_ERRORCHECK` during development to catch this.
- **Signaling outside lock** — `pthread_cond_signal` must be called while holding the mutex. Signaling outside the lock causes lost wakeups.
- **`if` instead of `while` on condvar** — Spurious wakeups are real. Always `while (!ready)`, never `if (!ready)`.
- **Locking too broadly** — Don't hold a lock across I/O, sleep, or long computations. Acquire, copy data, release, then process.
- **`Rc` across threads** — Rust: `Rc` is not `Send`. Use `Arc` for shared ownership across threads.

## Step 5: Verify

- [ ] C: compiles with `-pthread`, no TSAN warnings
- [ ] Rust: `cargo clippy` passes, `cargo test` passes
- [ ] Zig: `zig build` passes, `zig build test` passes
- [ ] No data races detected (TSAN / miri / manual review)
- [ ] Lock ordering is consistent (no deadlock potential)
- [ ] All threads are joined or detached (no leaked threads)
