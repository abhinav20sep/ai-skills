---
name: kernel-debugging
description: Systematic Linux kernel debugging workflows using ftrace, perf, crash/kdump, KGDB, dynamic debug, and KASAN/KCSAN/KMSAN. Teaches symptom-to-tool-to-interpretation patterns for kernel panics, hangs, performance issues, memory corruption, and data races. Use when debugging kernel modules, device drivers, or diagnosing production kernel issues.
---

# Kernel Debugging

**Core principle**: Start with the symptom, not the tool. A kernel panic needs dmesg and crash, not perf. A performance regression needs perf and flame graphs, not KASAN. Classify the problem first, then choose the tool. See [FTRACE.md](FTRACE.md) for ftrace workflows and [PERF.md](PERF.md) for perf profiling.

Systematic debugging workflows for the Linux kernel. Teaches how to go from symptom → tool → interpretation.

## Step 1: Identify the symptom

Classify the problem before choosing a tool:

| Symptom | Primary tools | Secondary tools |
|---|---|---|
| Kernel panic / oops | dmesg, crash, addr2line | KASAN, ftrace |
| System hang / deadlock | NMI watchdog, crash, lockdep | ftrace, KGDB |
| Performance regression | perf, ftrace | flame graphs, PMU counters |
| Memory corruption | KASAN, KMSAN | crash, kmemleak, slabinfo |
| Data race | KCSAN, lockdep | perf lock, ftrace |
| Memory leak | kmemleak, perf, /proc/meminfo | slabinfo, page_owner |
| Slow I/O | blktrace, ftrace, perf | iostat, iotop |
| Interrupt storm | /proc/interrupts, perf, ftrace | irqbalance, /proc/irq/ |

## Step 2: Collect diagnostic data

### dmesg — kernel messages

```bash
# Follow in real-time
dmesg -w

# Filter by severity
dmesg --level=err,crit,alert,emerg

# Show timestamps (seconds since boot)
dmesg -T

# Last 100 lines with subsystem filter
dmesg | grep -iE 'oops|panic|bug|warning|error|fault' | tail -100

# Full kernel journal (systemd)
journalctl -k --since "1 hour ago" -p err
```

**Reading an oops message**:
```
BUG: unable to handle kernel NULL pointer dereference at 0000000000000010
PGD 0 P4D 0
Oops: 0000 [#1] SMP NOPTI      ← fault type: 0000 = read, non-exec
CPU: 3 PID: 1234 Comm: myapp Tainted: G        W         5.15.0
RIP: 0010:my_driver_read+0x42/0x1a0 [mydriver]  ← faulting function + offset
Call Trace:                    ← stack trace
 my_driver_read+0x42/0x1a0 [mydriver]
 vfs_read+0x9e/0x1a0
 ksys_read+0x67/0xe0
```

**Key fields**:
- `RIP` — instruction pointer at fault (`my_driver_read+0x42` = function + 0x42 bytes)
- `[#1]` — oops count (multiple = cascading failures)
- `Tainted` — `G`=proprietary module, `W`=warned before, `D`=died recently
- `Call Trace` — reverse call stack (most recent first)

**Resolve address to line**:
```bash
# For in-tree modules:
addr2line -e vmlinux ffffffff81234567

# For out-of-tree modules:
addr2line -e mydriver.ko 0x42

# Or use faddr-to-line from kernel scripts:
scripts/faddr-to-line mydriver.ko my_driver_read+0x42
```

### crash — post-mortem dump analysis

Setup (requires `kdump`):
```bash
# Install
apt install kdump-tools crash linux-image-$(uname -r)-dbg

# Enable kdump (add to kernel cmdline):
crashkernel=256M

# Verify kdump is active
systemctl status kdump-tools
```

Analyzing a vmcore:
```bash
crash /usr/lib/debug/boot/vmlinux-$(uname -r) /var/crash/*/vmcore
```

**Essential crash commands**:
| Command | Purpose |
|---|---|
| `bt` | Backtrace of current task |
| `bt -a` | Backtrace of ALL CPUs |
| `ps` | Process list |
| `log` | Kernel log buffer (dmesg) |
| `mod` | Loaded modules |
| `struct task_struct <addr>` | Inspect task struct |
| `kmem -s` | Slab allocator stats |
| `kmem -i` | Memory usage summary |
| `vm <pid>` | Virtual memory map |
| `files <pid>` | Open file descriptors |
| `dis <function>` | Disassemble function |
| `rd <addr> <count>` | Read memory |
| `search -k <value>` | Search kernel memory |
| `irq -d` | IRQ stats |

**Live system analysis** (no crash dump needed):
```bash
crash /usr/lib/debug/boot/vmlinux-$(uname -r) /proc/kcore
```

### KGDB / KDB — interactive kernel debugging

**Setup** (requires kernel compiled with `CONFIG_KGDB=y`, `CONFIG_KDB=y`):
```bash
# Add to kernel cmdline:
kgdboc=ttyS0,115200 kgdbwait

# Or trigger from running system:
echo g > /proc/sysrq-trigger   # drops into KGDB
```

**Connect GDB**:
```bash
gdb vmlinux
(gdb) target remote /dev/ttyS0      # or
(gdb) target remote localhost:1234   # if using QEMU -s
```

**Common KGDB commands**:
```
(gdb) break my_driver_probe         # set breakpoint
(gdb) continue                       # resume execution
(gdb) bt                            # backtrace
(gdb) info threads                  # all CPUs
(gdb) print *dev                    # inspect struct
(gdb) p/x *(struct pci_dev *)0xffff888003a40000
(gdb) monitor dmesg                 # read kernel log from GDB
```

**KDB** (simpler, no GDB):
```
Entering kdb (current=0xffff888003a18000, pid 1234) on processor 3
[3]kdb> bt                          # backtrace
[3]kdb> ps                          # process list
[3]kdb> dmesg 50                    # last 50 dmesg lines
[3]kdb> go                          # resume
```

### /proc and /sys — runtime kernel state

```bash
# Memory usage
cat /proc/meminfo
cat /proc/slabinfo
cat /proc/buddyinfo          # page allocator fragmentation

# Process memory maps
cat /proc/<pid>/maps
cat /proc/<pid>/smaps_rollup

# Open files
ls -la /proc/<pid>/fd
cat /proc/<pid>/fdinfo/0

# Kernel modules
cat /proc/modules
cat /sys/module/<name>/sections/.text   # module text section addr

# Network stats
cat /proc/net/dev
cat /proc/net/tcp

# Block I/O stats
cat /proc/diskstats
cat /sys/block/sda/stat

# CPU info
cat /proc/cpuinfo
cat /proc/stat

# Interrupts
cat /proc/interrupts
cat /proc/softirqs
```

See [FTRACE.md](FTRACE.md) for ftrace workflows and [PERF.md](PERF.md) for perf profiling.

## Step 3: Follow systematic debugging workflows

### Workflow: Kernel panic / oops

```
1. Capture the oops message (dmesg, serial console, pstore)
2. Identify the faulting function from RIP
3. Identify the fault type:
   - 0000 = read access
   - 0002 = write access
   - 0010 = supervisor read (kernel)
   - 0012 = supervisor write (kernel)
   - 0014 = user access (likely copy_from_user)
4. Check if the address is NULL (0x0) or near-NULL (small offset)
5. Resolve RIP to source line: addr2line or faddr-to-line
6. Read the code at the faulting line
7. Check what variables could be NULL or invalid
8. Trace the code path: who calls this function? Are there error paths?
9. Check recent changes: git log on the affected file
10. If module-related: check if the module was loaded, check module's probe/remove
```

### Workflow: System hang / deadlock

```
1. Check if NMI watchdog fired:
   dmesg | grep -i 'watchdog.*stuck\|hard LOCKUP\|soft LOCKUP'
2. If yes, get the stuck CPU's backtrace from the watchdog message
3. Check lockdep:
   dmesg | grep -i 'lockdep\|deadlock\|inconsistent lock'
4. If no watchdog, trigger NMI:
   echo 1 > /proc/sys/kernel/hardlockup_panic
   # or: echo c > /proc/sysrq-trigger (panic + crashdump)
5. Analyze with crash:
   crash> bt -a          # backtrace all CPUs
   crash> ps -m          # show mutex waiters
   crash> struct mutex <addr>   # check lock owner
6. Look for patterns:
   - Two threads holding each other's locks (classic ABBA deadlock)
   - Thread holding spinlock while sleeping
   - IRQ handler acquiring lock already held by process context
```

### Workflow: Performance regression

```
1. Get baseline: perf stat -a -- sleep 10
2. Record profile: perf record -ag -- sleep 10
3. Generate flame graph:
   perf script | stackcollapse-perf.pl | flamegraph.pl > out.svg
4. Look for:
   - Unexpectedly hot functions
   - Lock contention (spinlock/mutex in top functions)
   - Cache misses (perf stat -e cache-misses,cache-references)
   - Branch mispredictions (perf stat -e branch-misses)
5. Check if regression is in kernel or userspace:
   perf record -ag -- sleep 10
   perf report --sort comm,dso
6. Bisect if needed:
   git bisect start
   git bisect bad <current>
   git bisect good <known-good>
```

### Workflow: Memory leak

```
1. Check if it's a slab leak:
   cat /proc/slabinfo | sort -k2 -nr | head -20
2. Enable kmemleak (requires CONFIG_DEBUG_KMEMLEAK):
   echo scan > /sys/kernel/debug/kmemleak
   cat /sys/kernel/debug/kmemleak
3. Check page allocator:
   cat /proc/buddyinfo
   cat /proc/pagetypeinfo
4. Check for leaked file descriptors:
   ls /proc/<pid>/fd | wc -l
   ls /proc/<pid>/fdinfo/
5. Check for leaked sockets:
   ss -s
   cat /proc/net/sockstat
6. Use perf to profile allocation paths:
   perf record -ag -e 'kmem:kmalloc' -- sleep 10
   perf script | head -50
```

## Anti-patterns

- **Starting with perf for a panic** — Perf profiles CPU usage. A panic needs dmesg, crash, and addr2line. Match tool to symptom.
- **No addr2line on oops** — The oops gives you `function+0x42`. Always resolve to source line with `addr2line` or `scripts/faddr-to-line` before reading code.
- **KASAN in production** — KASAN adds ~2x memory overhead and ~3x slowdown. Use it in development/test builds only.
- **Missing crash dump setup** — If kdump isn't configured, a panic means lost diagnostics. Set up `crashkernel=256M` on test machines.
- **Grepping dmesg without -T** — Without timestamps, you can't correlate events. Always use `dmesg -T` or `journalctl -k`.

## Step 4: Common bug patterns and fixes

| Bug pattern | Symptom | Detection | Fix |
|---|---|---|---|
| NULL pointer deref | oops at address 0x0-0xfff | KASAN, dmesg | Check return values, add NULL checks |
| Use-after-free | oops at freed address | KASAN, kmemleak | Check refcounting, use-after-free detection |
| Double-free | slab corruption | KASAN | Set pointer to NULL after free |
| Buffer overflow | oops at adjacent object | KASAN | Bounds check before write |
| Deadlock | hang + watchdog | lockdep, KCSAN | Consistent lock ordering |
| Data race | intermittent corruption | KCSAN, lockdep | Add proper locking or use atomics |
| Memory leak | growing memory usage | kmemleak, /proc/slabinfo | Check all error paths |
| Integer overflow | wrap-around value | sparse, compiler warnings | Use check_add_overflow() |
| Missing error path | resource leak on probe failure | manual review, fault injection | Unwind in reverse order |
| Stack overflow | random corruption | CONFIG_VMAP_STACK | Reduce stack usage, use kmalloc |

## Step 5: Verify

- [ ] `dmesg` shows no errors/warnings
- [ ] Module loads and unloads cleanly (`insmod`/`rmmod`)
- [ ] No lockdep warnings (`CONFIG_LOCKDEP=y`)
- [ ] No KASAN reports (`CONFIG_KASAN=y`)
- [ ] No KCSAN reports (`CONFIG_KCSAN=y`)
- [ ] `perf stat` shows expected CPU/memory counters
- [ ] No leaked file descriptors, memory, or sockets
