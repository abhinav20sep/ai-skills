# Coccinelle Semantic Patch Reference

Coccinelle (spatch) is the standard tool for tree-wide refactoring in the Linux kernel. This file provides examples for cast migration patterns.

## Running Coccinelle

```bash
# Single file:
spatch --sp-file migrate.cocci drivers/gpu/drm/mydriver.c

# Entire tree:
spatch --sp-file migrate.cocci --dir drivers/gpu/drm/

# Dry run (no changes):
spatch --sp-file migrate.cocci --dir drivers/gpu/drm/ --dry-run

# With kernel includes:
spatch --sp-file migrate.cocci --dir . --include-path include/
```

## Pattern 1: void * → typed pointer

Migrate `void *` parameters to typed pointers where the type is known.

```cocci
@@
identifier fn;
type T;
expression ptr;
@@

- void *ptr
+ T *ptr
  {
  ... when != ptr
- T *typed = (T *)ptr;
  ... when != typed
  }
```

## Pattern 2: Raw cast → container_of

Replace `(struct foo *)work` with `container_of(work, struct foo, work)` when casting from a struct member back to the containing struct.

```cocci
@@
type T;
identifier member;
expression work;
@@

- (T *)work
+ container_of(work, T, member)
```

Note: Coccinelle cannot fully automate this — you must specify `T` and `member`. Use it as a starting point, then manually verify.

## Pattern 3: Add __user annotation

Add `__user` to userspace pointer parameters.

```cocci
@@
identifier fn;
type T;
expression buf;
@@

  int fn(
- T *buf
+ T __user *buf
  , ...)
  {
  ...
  }
```

## Pattern 4: Add __iomem annotation

Add `__iomem` to MMIO pointer parameters.

```cocci
@@
identifier fn;
expression regs;
@@

- void *regs
+ void __iomem *regs
  {
  ...
  }
```

## Pattern 5: Enum cast → __bitwise

This is a multi-step migration. Coccinelle can help find the casts, but the typedef change is manual.

Step 1: Find enum → int casts:
```cocci
@@
enum my_enum val;
@@

- (int)val
+ (__force int)val
```

Step 2: Manually define the `__bitwise` type:
```c
/* In header */
typedef enum __bitwise my_flags_t;
#define MY_FLAG_A ((__force my_flags_t)0x01)
#define MY_FLAG_B ((__force my_flags_t)0x02)
```

Step 3: Replace `int` with `my_flags_t` in function signatures and struct fields.

## Pattern 6: Integer overflow check

Add bounds checking before narrowing integer casts.

```cocci
@@
expression val;
type T;
@@

- (T)val
+ ({ \
+   typeof(val) __v = (val); \
+   if (__v > (typeof(__v))T##_MAX) \
+   	return -EINVAL; \
+   (T)__v; \
+ })
```

Note: This is illustrative. Real migration requires manual judgment on what to do on overflow.

## Pattern 7: Unsafe function pointer cast

Find and flag function pointer casts for manual review.

```cocci
@@
type T1, T2;
expression fn;
@@

- (T1 (*)(T2))fn
+ /* MANUAL REVIEW: function pointer cast from typeof(fn) to T1(*)(T2) */
+ (T1 (*)(T2))fn
```

These cannot be safely automated — flag for manual review.

## Writing custom patches

### Basic structure

```cocci
@@
<metavariable declarations>
@@

<matching code with - for removal>
+<replacement code>
```

### Metavariable types

- `expression e` — any expression
- `identifier f` — any identifier
- `type T` — any type
- `statement S` — any statement
- `position p` — source position (for context)

### Context matching

Use `...` to match any amount of code between patterns:

```cocci
@@
expression e;
@@

  foo(e);
  ...
  bar(e);
```

### When clauses

Restrict matches with `when`:

```cocci
@@
expression e;
@@

  foo(e);
  ... when != e
  bar(e);
```

### Nested matches

Use `exists` to match within a function:

```cocci
@@
identifier fn;
expression e;
@@

  fn(...) {
  ...
  cast(e)
  ...
  }
```

## Common kernel patterns to search for

### Find all void * parameters

```bash
grep -rn 'void \*' --include='*.c' --include='*.h' drivers/
```

### Find all raw casts

```bash
grep -rn '(\(struct [a-z_]*\))\s*[a-z_]' --include='*.c' drivers/
```

### Find all as_conversions in Rust

```bash
cargo clippy -- -W clippy::as_conversions 2>&1 | grep 'as_conversions'
```

## Verification after migration

1. `make C=1` — sparse checks all new annotations
2. `make W=1` — extra compiler warnings
3. `scripts/checkpatch.pl` — style compliance
4. `make M=<dir>` — module still compiles
5. `insmod` / `rmmod` — module loads and unloads
6. Run relevant KUnit/kselftest tests
