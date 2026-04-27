---
name: kernel-qa
description: Interactive QA session for kernel and driver bugs. Explores codebase with kernel-specific awareness (subsystem layers, lockdep, DMA, interrupt handling), suggests reproduction strategies (QEMU, KUnit, fault injection), and files issues with kernel-appropriate detail. Use when testing drivers, debugging kernel panics, or doing QA on kernel/HSM code.
---

# Kernel QA Session

Run an interactive QA session for kernel and driver bugs. The user describes problems — you clarify, explore the codebase with kernel-specific awareness, and file issues with appropriate detail.

## For each issue the user raises

### 1. Listen and capture

Let the user describe the problem. Accept any of:
- Kernel panic / oops message
- `dmesg` output
- Call trace
- Unexpected behavior description
- Lockdep warning
- Sparse warning

Ask **at most 2-3 clarifying questions**:
- Kernel version and architecture (`uname -a`)
- Config options relevant to the subsystem (`zgrep CONFIG_<SUBSYS> /proc/config.gz`)
- Steps to reproduce (if not in the panic log)
- Whether it's consistent or intermittent
- Recent changes (commits, config changes, hardware)

Do NOT over-interview. If the description is clear enough, move on.

### 2. Explore codebase with kernel awareness

Kick off parallel exploration to understand the affected code:

**Trace the code path**:
- Userspace entry: `ioctl`, `sysfs`, `read`/`write`, `mmap`
- Driver layer: `file_operations`, `pci_driver`, `platform_driver`
- Subsystem: DMA API, crypto API, netdev, block layer
- Hardware: register accesses, BAR reads/writes, DMA descriptors

**Check for common driver bugs**:

| Bug pattern | What to check |
|---|---|
| Use-after-free | `kfree`/`devm_kfree` followed by access; check refcounting |
| Double-free | Error paths that free the same resource twice |
| Race conditions | Missing locks around shared state; IRQ vs process context |
| Missing error paths | `probe()` success path allocated resources not freed on failure |
| Incorrect DMA direction | `DMA_TO_DEVICE` vs `DMA_FROM_DEVICE` mismatch |
| Leaked regions | `pci_request_regions` without `pci_release_regions` in `remove()` |
| Missing `free_irq` | `request_irq` without `free_irq` in error/cleanup paths |
| Integer overflow | Size calculations without overflow checks |
| Buffer overflow | `copy_from_user`/`copy_to_user` without size validation |
| Missing `MODULE_LICENSE` | Required for GPL-only symbols |

**Check kernel messages**:
```bash
dmesg | grep -iE 'oops|panic|bug|warning|error|fault' | tail -50
journalctl -k --since "1 hour ago"
```

**Check lockdep**:
```bash
# If lockdep is enabled:
dmesg | grep -iE 'lockdep|deadlock|inconsistent lock'
```

### 3. Assess scope

Before filing, decide: **single issue** or **breakdown**?

Break down when:
- Multiple independent failure modes
- Fix spans multiple subsystems
- Separable concerns (e.g., memory leak AND race condition)

Keep as single issue when:
- One root cause, one fix
- Symptoms all trace to the same code path

### 4. Attempt reproduction

Suggest strategies based on the bug type:

**QEMU device emulation**:
```bash
# For PCIe devices:
qemu-system-x86_64 -kernel bzImage -append "console=ttyS0" \
  -device pci-assign,host=01:00.0 \
  -nographic

# For platform devices:
qemu-system-x86_64 -kernel bzImage -append "console=ttyS0" \
  -dtb qemu-arm64.dtb -nographic
```

**KUnit test skeleton**:
```c
#include <kunit/test.h>

static void test_<name>(struct kunit *test)
{
	/* Test the failing behavior */
	/* Use KUNIT_EXPECT_EQ, KUNIT_ASSERT_NOT_NULL, etc. */
}

static struct kunit_case <name>_cases[] = {
	KUNIT_CASE(test_<name>),
	{}
};

static struct kunit_suite <name>_suite = {
	.name = "<name>",
	.test_cases = <name>_cases,
};
kunit_test_suite(<name>_suite);
MODULE_LICENSE("GPL");
```

Run: `./tools/testing/kunit/kunit.py run <name>`

**Fault injection** (for error path testing):
```bash
# Enable fault injection in kernel config:
# CONFIG_FAULT_INJECTION=y
# CONFIG_FAIL_MAKE_REQUEST=y
# CONFIG_FAULT_INJECTION_DEBUG_FS=y

# Inject allocation failures:
echo 100 > /sys/kernel/debug/failslab/probability
echo 1 > /sys/kernel/debug/failslab/interval
echo 1 > /sys/kernel/debug/failslab/times
```

### 5. File the issue

File with `gh issue create`. Use kernel-appropriate detail:

```
## Summary

[One-line description of the bug]

## Kernel info

- Kernel version: (uname -r)
- Architecture: (uname -m)
- Config: relevant CONFIG_* options
- Hardware: (lspci -vvv output for PCIe devices)

## Call trace / oops

[Paste the full call trace or oops message]
[Include the function, offset, and module if visible]

## Steps to reproduce

1. [Load module: modprobe <name>]
2. [Trigger condition: e.g., run workload, hotplug, etc.]
3. [Observe: dmesg shows oops]

## Expected behavior

[What should happen]

## Actual behavior

[What actually happens — describe the failure mode]

## Additional context

[Any relevant dmesg output, lockdep warnings, sparse warnings]
[If regression: suggest git bisect range]
```

**Regression detection**:
If the user says it used to work:
- Ask for last known-good kernel version
- Suggest: `git bisect start <bad> <good>`
- Identify the subsystem for targeted bisect: `git bisect -- drivers/<subsystem>/`

### 6. Continue session

Keep going until the user says they're done. Each issue is independent — don't batch them.

## Kernel debugging tools reference

| Tool | Use | Command |
|---|---|---|
| `dmesg` | Kernel messages | `dmesg -w` (follow) |
| `ftrace` | Function tracing | `echo function > /sys/kernel/debug/tracing/current_tracer` |
| `perf` | Performance profiling | `perf top -p <pid>` |
| `lockdep` | Lock ordering validation | `CONFIG_LOCKDEP=y` |
| `sparse` | Static analysis | `make C=1` |
| `KUnit` | Unit tests | `./tools/testing/kunit/kunit.py run` |
| `kselftest` | Integration tests | `make kselftest` |
| `dynamic debug` | Runtime debug prints | `echo 'module <name> +p' > /sys/kernel/debug/dynamic_debug/control` |
| `crash` | Post-mortem dump analysis | `crash vmlinux vmcore` |
