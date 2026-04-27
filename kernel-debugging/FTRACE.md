# ftrace Reference

The kernel's built-in function tracer. No external tools needed — everything is in `/sys/kernel/debug/tracing/`.

## Setup

```bash
# Mount debugfs if not already mounted
mount -t debugfs none /sys/kernel/debug

# Check available tracers
cat /sys/kernel/debug/tracing/available_tracers
# Expected: nop function function_graph ...

# Check current tracer
cat /sys/kernel/debug/tracing/current_tracer
# Default: nop (no tracing)
```

## Function tracer

Traces every function call in the kernel (or filtered subset).

```bash
# Enable function tracing
echo function > /sys/kernel/debug/tracing/current_tracer

# Start tracing
echo 1 > /sys/kernel/debug/tracing/tracing_on

# Do something (run your workload)

# Stop tracing
echo 0 > /sys/kernel/debug/tracing/tracing_on

# Read the trace
cat /sys/kernel/debug/tracing/trace
```

**Output format**:
```
#           TASK-PID     CPU#  TIMESTAMP     FUNCTION
            bash-1234  [003]  1234.567890: vfs_read <-ksys_read
            bash-1234  [003]  1234.567891: my_driver_read <-vfs_read
            bash-1234  [003]  1234.567892: ioread32 <-my_driver_read
```

## Function graph tracer

Shows call graph with timing (indentation = call depth).

```bash
echo function_graph > /sys/kernel/debug/tracing/current_tracer
echo 1 > /sys/kernel/debug/tracing/tracing_on
# ... run workload ...
echo 0 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace
```

**Output format**:
```
#           TASK-PID   CPU#  DURATION         FUNCTION CALLS
            bash-1234  [003]               |  vfs_read() {
            bash-1234  [003]   1.234 us    |    my_driver_read() {
            bash-1234  [003]   0.456 us    |      ioread32();
            bash-1234  [003]   0.789 us    |    }
            bash-1234  [003]   2.345 us    |  }
```

`DURATION` shows time spent in each function. `{` = entry, `}` = exit. Functions without `}` are leaf calls.

## Event tracing

Trace specific kernel events (scheduling, syscalls, IRQs, custom tracepoints).

```bash
# List available events
ls /sys/kernel/debug/tracing/events/

# List events in a subsystem
ls /sys/kernel/debug/tracing/events/sched/
ls /sys/kernel/debug/tracing/events/irq/
ls /sys/kernel/debug/tracing/events/kmem/

# Enable specific events
echo 1 > /sys/kernel/debug/tracing/events/sched/sched_switch/enable
echo 1 > /sys/kernel/debug/tracing/events/irq/irq_handler_entry/enable

# Read event trace
cat /sys/kernel/debug/tracing/trace

# Disable
echo 0 > /sys/kernel/debug/tracing/events/sched/sched_switch/enable
```

## Filter functions

Only trace specific functions or processes.

```bash
# Filter by function name
echo my_driver_read > /sys/kernel/debug/tracing/set_ftrace_filter
echo function > /sys/kernel/debug/tracing/current_tracer

# Filter by function with wildcard
echo 'my_driver_*' > /sys/kernel/debug/tracing/set_ftrace_filter

# Filter by PID
echo 1234 > /sys/kernel/debug/tracing/set_ftrace_pid

# Filter by process (current shell)
echo $$ > /sys/kernel/debug/tracing/set_ftrace_pid

# Clear filters
echo > /sys/kernel/debug/tracing/set_ftrace_filter
echo > /sys/kernel/debug/tracing/set_ftrace_notrace
```

## trace-cmd (convenience wrapper)

```bash
# Install
apt install trace-cmd  # or: dnf install trace-cmd

# Record function graph of a command
trace-cmd record -p function_graph -g my_driver_read -F -- ls /dev/

# Record all events for 5 seconds
trace-cmd record -e all sleep 5

# Record specific events
trace-cmd record -e sched_switch -e sched_wakeup sleep 5

# Read the trace
trace-cmd report

# Live tracing (stream to terminal)
trace-cmd stream -p function_graph -g my_driver_read
```

## Custom tracepoints in modules

Add tracepoints to your kernel module:

```c
#include <linux/tracepoint.h>

/* Define tracepoint */
TRACE_EVENT(my_event,
    TP_PROTO(struct my_dev *dev, int status),
    TP_ARGS(dev, status),
    TP_STRUCT__entry(
        __field(int, status)
        __string(name, dev->name)
    ),
    TP_fast_assign(
        __entry->status = status;
        __assign_str(name, dev->name);
    ),
    TP_printk("dev=%s status=%d", __get_str(name), __entry->status)
);

/* In your code */
trace_my_event(dev, 0);

/* Enable from userspace */
echo 1 > /sys/kernel/debug/tracing/events/my_module/my_event/enable
```

## Useful trace configurations

### Trace module probe function with timing

```bash
echo function_graph > /sys/kernel/debug/tracing/current_tracer
echo 'my_driver_probe' > /sys/kernel/debug/tracing/set_graph_function
echo 1 > /sys/kernel/debug/tracing/tracing_on
# ... trigger probe ...
echo 0 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace
```

### Trace all IRQ handlers

```bash
echo 1 > /sys/kernel/debug/tracing/events/irq/irq_handler_entry/enable
echo 1 > /sys/kernel/debug/tracing/events/irq/irq_handler_exit/enable
echo 1 > /sys/kernel/debug/tracing/tracing_on
# ... run workload ...
cat /sys/kernel/debug/tracing/trace | head -100
```

### Trace context switches

```bash
echo 1 > /sys/kernel/debug/tracing/events/sched/sched_switch/enable
echo 1 > /sys/kernel/debug/tracing/tracing_on
# ... run workload ...
cat /sys/kernel/debug/tracing/trace | grep sched_switch | head -50
```

## Reset everything

```bash
echo nop > /sys/kernel/debug/tracing/current_tracer
echo 0 > /sys/kernel/debug/tracing/tracing_on
echo > /sys/kernel/debug/tracing/set_ftrace_filter
echo > /sys/kernel/debug/tracing/set_graph_function
echo 0 > /sys/kernel/debug/tracing/events/enable
echo > /sys/kernel/debug/tracing/trace
```
