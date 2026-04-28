---
name: migrate-unsafe-casts
description: Find and replace unsafe type casts with safer alternatives in C, Rust, and Zig. Generates Coccinelle semantic patches for C kernel code, applies clippy fixes for Rust, and suggests Zig-safe patterns. Use when user wants to improve type safety, migrate from raw casts, or enforce sparse annotations.
---

# Migrate Unsafe Casts

**Core principle**: Every cast is a place where the compiler stopped checking your code. Replace raw casts with type-safe alternatives — `container_of` for struct containment, `__user`/`__iomem` for address space annotations, `__bitwise` for flagged types. The goal is zero unannotated casts in kernel code.

Scan codebases for unsafe type casts and replace with safer alternatives. Supports C (kernel + userspace), Rust, and Zig.

## Step 1: Detect language

Check file extensions:
- `*.c`, `*.h` → C workflow
- `*.rs` → Rust workflow
- `*.zig` → Zig workflow
- Mixed → run each workflow for its files

## Step 2: C workflow

### Scan for unsafe patterns

Search for these patterns in `.c` and `.h` files:

| Pattern | Example | Risk |
|---|---|---|
| Raw pointer cast | `(struct foo *)ptr` | Bypasses type checking |
| `void *` parameter | `void *arg` | No type safety |
| Integer ↔ pointer cast | `(void *)(long)val` | Size mismatch on 64-bit |
| Enum ↔ int cast | `(int)enum_val` | Loses type information |
| Function pointer cast | `(int (*)(void *))fn` | ABI mismatch risk |

### Suggest replacements

**Raw struct pointer → `container_of`**:

Before:
```c
struct my_dev *dev = (struct my_dev *)work;
```

After:
```c
struct my_dev *dev = container_of(work, struct my_dev, work);
```

**`void *` → typed + sparse annotations**:

Before:
```c
int process(void *buf, int len)
```

After:
```c
int process(void __user *buf, int len)
```

**Enum ↔ int → `__bitwise`**:

Before:
```c
int flags = (int)MY_FLAG_A;
```

After:
```c
/* Define with __bitwise in header: */
typedef enum __bitwise my_flags_t;
#define MY_FLAG_A ((__force my_flags_t)0x01)
my_flags_t flags = MY_FLAG_A;
```

**Unsafe integer cast → explicit check**:

Before:
```c
int fd = (int)long_val;
```

After:
```c
if (long_val > INT_MAX || long_val < INT_MIN)
	return -EINVAL;
int fd = (int)long_val;
```

### Generate Coccinelle patches

For batch migration, generate Coccinelle semantic patches. See [COCCINELLE.md](COCCINELLE.md) for examples.

### Verify

- [ ] `make C=1` (sparse) passes — new annotations checked
- [ ] `make W=1` — no new warnings
- [ ] `scripts/checkpatch.pl` — no style violations

## Step 3: Rust workflow

### Run clippy with strict lints

```bash
cargo clippy -- -W clippy::as_conversions -W clippy::cast_possible_truncation -W clippy::cast_sign_loss -W clippy::cast_possible_wrap
```

### Migration patterns

**`as` → `From`/`Into`** (infallible conversions):

Before:
```rust
let x: u64 = val as u64;
```

After:
```rust
let x: u64 = u64::from(val);
// or
let x: u64 = val.into();
```

**`as` → `TryFrom`/`TryInto`** (fallible conversions):

Before:
```rust
let x: u8 = val as u8; // truncation!
```

After:
```rust
let x: u8 = u8::try_from(val)?; // returns Err on overflow
```

**`unsafe { transmute }` → safe wrapper**:

Before:
```rust
let x: u32 = unsafe { std::mem::transmute(bytes) };
```

After:
```rust
let x: u32 = u32::from_ne_bytes(bytes);
```

**Pointer cast → safe abstraction**:

Before:
```rust
let ptr = val as *const u8;
```

After:
```rust
let ptr = std::ptr::from_ref(val) as *const u8;
// Or better: use references instead of raw pointers
```

### Verify

- [ ] `cargo clippy -- -W clippy::as_conversions` passes
- [ ] `cargo test` passes
- [ ] `cargo build` compiles

## Step 4: Zig workflow

### Scan for unsafe patterns

Search for: `@ptrCast`, `@intCast`, `@floatCast`, `@alignCast`

### Migration patterns

**`@intCast` → explicit check**:

Before:
```zig
const x: u8 = @intCast(val);
```

After:
```zig
const x: u8 = std.math.cast(u8, val) orelse return error.Overflow;
// or in safe builds: @intCast panics on overflow (already safe in debug)
```

**`@ptrCast` → `@alignCast` + `@ptrCast`**:

Before:
```zig
const ptr: *u32 = @ptrCast(raw_ptr);
```

After:
```zig
const aligned: *align(1) const u32 = @alignCast(raw_ptr);
const ptr: *const u32 = @ptrCast(aligned);
```

### Verify

- [ ] `zig build` compiles
- [ ] `zig build test` passes
- [ ] Run with `-Drelease-safe=true` to verify safety checks

## Anti-patterns

- **Replacing without verifying** — Don't run Coccinelle patches blindly without reviewing each change. `spatch` can produce incorrect results for complex pointer arithmetic patterns.
- **Annotating without sparse** — Adding `__user`/`__iomem` annotations is useless without running sparse (`make C=1`). The annotations are only checked by sparse.
- **Partial migration** — Don't annotate half the call chain. If you add `__user` to a parameter, cascade the annotation through all callers and callees. An annotated pointer passed to an unannotated function defeats the purpose.
- **`as` to `into()` without checking** — Rust: `val as u8` silently truncates. `u8::try_from(val)` returns a `Result`. Don't blindly replace `as` with `into()` — use `try_from()` for fallible conversions.
- **Coccinelle on out-of-tree modules** — Coccinelle needs the kernel include path. Run with `spatch --include-path include/` or from the kernel tree root. Running without includes produces parse errors.

## Step 5: Summary

After migration, report:
- Number of casts found vs migrated
- Remaining casts that need manual review
- Verification results (sparse/clippy/zig build)
