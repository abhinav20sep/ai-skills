# perf Reference

Linux profiling with `perf_events`. Covers CPU profiling, cache analysis, lock contention, and flame graphs.

## Setup

```bash
# Install
apt install linux-perf linux-tools-$(uname -r)
# or: dnf install perf

# Check permissions
cat /proc/sys/kernel/perf_event_paranoid
# 0 = full access, 1 = restricted, 2 = kernel only, 3 = no access
echo 1 > /proc/sys/kernel/perf_event_paranoid  # if needed
```

## Basic profiling

### perf stat — hardware counter summary

```bash
# System-wide for 10 seconds
perf stat -a -- sleep 10

# Specific command
perf stat ls -la

# Specific process
perf stat -p <pid> -- sleep 10

# Specific events
perf stat -e cycles,instructions,cache-misses,cache-references,branch-misses -- sleep 10

# Per-CPU
perf stat -a -A -- sleep 10
```

**Key metrics**:
| Counter | Meaning | Good ratio |
|---|---|---|
| `instructions/cycle` (IPC) | CPU efficiency | >1.0 for compute, >0.5 for memory-bound |
| `cache-misses/cache-references` | L1/L2 miss rate | <5% |
| `branch-misses/branches` | Branch prediction miss rate | <2% |
| `context-switches` | Context switch rate | depends on workload |

### perf record — sampling profile

```bash
# System-wide with call graphs (dwarf)
perf record -ag -- sleep 10

# System-wide with frame pointers (more accurate stacks)
perf record -ag --call-graph fp -- sleep 10

# Specific process
perf record -p <pid> -g -- sleep 10

# Specific command
perf record -g -- ./my_app

# CPU cycles only (default)
perf record -e cycles -ag -- sleep 10

# Cache misses
perf record -e cache-misses -ag -- sleep 10

# Context switches
perf record -e context-switches -ag -- sleep 10
```

### perf report — interactive analysis

```bash
perf report                    # interactive TUI
perf report --stdio            # text output
perf report --sort comm,dso    # sort by process + library
perf report -n --stdio | head -50   # top 50 functions
```

## Flame graphs

```bash
# Install FlameGraph tools
git clone https://github.com/brendangregg/FlameGraph.git

# Record + generate
perf record -ag -- sleep 10
perf script | ./FlameGraph/stackcollapse-perf.pl | ./FlameGraph/flamegraph.pl > flame.svg

# Open in browser
xdg-open flame.svg
```

**Reading a flame graph**:
- Width = percentage of time spent in that function
- Height = call stack depth
- Hot spots = wide plateaus at the top
- Kernel functions = red/orange, userspace = blue/green

## Hardware event analysis

### Cache profiling

```bash
# List available cache events
perf list cache

# Profile cache misses
perf stat -e L1-dcache-load-misses,L1-dcache-loads,LLC-load-misses,LLC-loads -- sleep 10

# Record cache miss profile
perf record -e LLC-load-misses -ag -- sleep 10
perf report --stdio | head -30
```

### Branch prediction

```bash
perf stat -e branch-misses,branches -- sleep 10
perf record -e branch-misses -ag -- sleep 10
perf report --stdio | head -30
```

### CPU cycles + instructions (IPC)

```bash
perf stat -e cycles,instructions -- sleep 10
# IPC = instructions/cycles
# Low IPC = memory-bound or pipeline stalls
# High IPC = compute-bound, good pipeline utilization
```

## Lock profiling

```bash
# Trace lock contention
perf lock record -- sleep 10
perf lock report

# Or with tracepoints
perf record -e lock:lock_acquire -e lock:lock_release -ag -- sleep 10
perf script | head -50
```

## Kernel-specific profiling

### Trace kernel functions

```bash
# Trace a specific kernel function
perf probe --add my_driver_read
perf record -e probe:my_driver_read -ag -- sleep 10
perf script
perf probe --del probe:my_driver_read

# Trace with arguments
perf probe --add 'my_driver_read dev->name'
perf record -e probe:my_driver_read -ag -- sleep 10
perf script
```

### Trace syscalls

```bash
# Trace all syscalls
perf trace -- sleep 10

# Trace specific syscall
perf trace -e read -- sleep 10

# Summary of syscalls
perf trace -s -- sleep 10
```

### Scheduler profiling

```bash
# Scheduling latency
perf sched record -- sleep 10
perf sched latency          # per-thread wakeup latency
perf sched map              # CPU timeline
perf sched timehist         # per-thread scheduling history
```

## PMU counters (kernel modules)

Access hardware performance counters from kernel code:

```c
#include <linux/perf_event.h>

struct perf_event *event;
struct perf_event_attr attr = {
    .type = PERF_TYPE_HARDWARE,
    .config = PERF_COUNT_HW_CACHE_MISSES,
    .disabled = 1,
    .exclude_kernel = 0,
    .exclude_hv = 1,
};

event = perf_event_create_kernel_counter(&attr, -1, NULL, NULL, NULL);
perf_event_enable(event);

/* ... workload ... */

pr_info("cache misses: %lld\n", local64_read(&event->count));

perf_event_disable(event);
perf_event_release_kernel(event);
```

## Useful one-liners

```bash
# Top functions by CPU time (system-wide, 10s)
perf top -a

# Profile a specific CPU
perf record -C 0 -g -- sleep 10

# Profile with frequency (4000 Hz = 4000 samples/sec)
perf record -F 4000 -ag -- sleep 10

# Profile with call chain (8 levels deep)
perf record -g --max-stack 8 -- sleep 10

# Stat with per-thread breakdown
perf stat -e cycles -a --per-thread -- sleep 10

# Record only when >100ms of CPU consumed
perf record -ag -c 1000000 -- sleep 10
```
