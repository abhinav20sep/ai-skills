---
name: setup-kernel-hooks
description: Set up native git hooks for C/Rust/Zig kernel and systems projects. Configures clang-format, checkpatch.pl, sparse, rustfmt, clippy, zig fmt, and test runners. Use when user wants pre-commit checks, kernel coding style enforcement, or CI-ready hooks.
---

# Setup Kernel Hooks

Set up native git hooks for kernel and systems projects. No Node.js required — uses shell scripts and native tooling.

## Step 1: Detect project languages

Check file extensions in the repo:

| Files found | Language | Build system |
|---|---|---|
| `*.c`, `*.h`, `Makefile` (with `obj-m`/`obj-y`) | C (kernel) | Kbuild |
| `*.c`, `*.h`, `Makefile` (standard) | C (userspace) | Make |
| `*.rs`, `Cargo.toml` | Rust | Cargo |
| `*.zig`, `build.zig` | Zig | Zig build |
| `Kconfig` at root | Linux kernel tree | Kbuild |

Check which tools are available:

```bash
which clang-format checkpatch.pl sparse rustfmt cargo zig 2>/dev/null
```

Report available/missing tools to user. Skip checks for missing tools with a warning.

## Step 2: Create `githooks/` directory

```bash
mkdir -p githooks/
```

## Step 3: Create `githooks/pre-commit`

Write the pre-commit hook with appropriate checks per detected language.

### For C kernel code

```bash
#!/bin/bash
set -e

echo "Running pre-commit checks..."

# C/H files: clang-format + checkpatch
C_FILES=$(git diff --cached --name-only --diff-filter=ACM | grep -E '\.(c|h)$' || true)
if [ -n "$C_FILES" ]; then
    # clang-format check
    if command -v clang-format &>/dev/null; then
        for f in $C_FILES; do
            clang-format --dry-run --Werror "$f" || {
                echo "FAIL: clang-format: $f (run: clang-format -i $f)"
                exit 1
            }
        done
    fi

    # checkpatch.pl
    if [ -f scripts/checkpatch.pl ]; then
        for f in $C_FILES; do
            ./scripts/checkpatch.pl --no-tree -f "$f" || true
        done
    fi
fi
```

### For Rust code

```bash
# Rust files: rustfmt
RS_FILES=$(git diff --cached --name-only --diff-filter=ACM | grep -E '\.rs$' || true)
if [ -n "$RS_FILES" ]; then
    if command -v rustfmt &>/dev/null; then
        for f in $RS_FILES; do
            rustfmt --check "$f" || {
                echo "FAIL: rustfmt: $f (run: rustfmt $f)"
                exit 1
            }
        done
    fi
fi
```

### For Zig code

```bash
# Zig files: zig fmt
ZIG_FILES=$(git diff --cached --name-only --diff-filter=ACM | grep -E '\.zig$' || true)
if [ -n "$ZIG_FILES" ]; then
    if command -v zig &>/dev/null; then
        for f in $ZIG_FILES; do
            zig fmt --check "$f" || {
                echo "FAIL: zig fmt: $f (run: zig fmt $f)"
                exit 1
            }
        done
    fi
fi
```

Combine all relevant sections into a single `githooks/pre-commit` file based on detected languages.

## Step 4: Create `githooks/commit-msg`

Enforce kernel-style commit messages: `subsystem: description` with 72-char limit.

```bash
#!/bin/bash
set -e

MSG=$(cat "$1")

# Skip merge commits
if echo "$MSG" | head -1 | grep -qE '^Merge '; then
    exit 0
fi

# First line: subsystem: description
FIRST=$(echo "$MSG" | head -1)

# Check format: lowercase start, colon present, no period at end, max 72 chars
if ! echo "$FIRST" | grep -qE '^[a-z].*: .+'; then
    echo "ERROR: Commit message must start with 'subsystem: description'"
    echo "  Example: 'driver: fix null pointer in probe function'"
    exit 1
fi

if [ ${#FIRST} -gt 72 ]; then
    echo "ERROR: First line too long (${#FIRST} chars, max 72)"
    exit 1
fi

if echo "$FIRST" | grep -qE '\.$'; then
    echo "ERROR: First line should not end with a period"
    exit 1
fi

# Blank line after first line
SECOND=$(echo "$MSG" | sed -n '2p')
if [ -n "$SECOND" ]; then
    echo "ERROR: Second line must be blank"
    exit 1
fi

# Body lines max 72 chars (warn, don't fail)
echo "$MSG" | tail -n +3 | while IFS= read -r line; do
    if [ ${#line} -gt 72 ]; then
        echo "WARNING: Body line too long (${#line} chars): $line"
    fi
done
```

## Step 5: Create `githooks/pre-push`

Run tests before pushing.

```bash
#!/bin/bash
set -e

echo "Running pre-push checks..."

# Detect and run tests based on project type
if [ -f build.zig ]; then
    echo "Running Zig tests..."
    zig build test
elif [ -f Cargo.toml ]; then
    echo "Running Rust tests..."
    cargo test
elif [ -f tools/testing/kunit/kunit.py ]; then
    echo "Running KUnit tests..."
    ./tools/testing/kunit/kunit.py run
elif [ -f Makefile ] && grep -q 'kselftest' Makefile; then
    echo "Running kselftest..."
    make kselftest
fi
```

## Step 6: Make hooks executable

```bash
chmod +x githooks/pre-commit githooks/commit-msg githooks/pre-push
```

## Step 7: Activate hooks

Tell the user to run:

```bash
git config core.hooksPath githooks/
```

This is a one-time per-clone setup. Add to project README.

## Step 8: Verify

- [ ] `githooks/pre-commit` exists and is executable
- [ ] `githooks/commit-msg` exists and is executable
- [ ] `githooks/pre-push` exists and is executable
- [ ] `git config core.hooksPath` points to `githooks/`
- [ ] Test: `echo "test: bad message." | git commit --allow-empty -F -` should fail
- [ ] Test: `echo "test: good message" | git commit --allow-empty -F -` should pass
- [ ] Test: stage a `.c` file and verify clang-format runs

## Tool-specific notes

- **checkpatch.pl**: Must be at `scripts/checkpatch.pl` relative to repo root. For out-of-tree modules, copy from kernel tree or use `--no-tree` flag.
- **sparse**: Run with `make C=1` for full kernel tree. For individual files: `sparse -D__KERNEL__ file.c`.
- **clang-format**: Kernel uses `.clang-format` at tree root. For out-of-tree, copy from kernel or create one.
- **rustfmt**: Uses `rustfmt.toml` config if present. Kernel has `rustfmt.toml` at tree root.
