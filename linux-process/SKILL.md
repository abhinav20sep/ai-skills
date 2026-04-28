---
name: linux-process
description: Process management and IPC patterns for Linux in C (fork/exec, pipes, shared memory, semaphores, message queues, Unix sockets, signals), Rust (std::process, nix crate), and Zig (std.process). Use when user wants to spawn processes, set up IPC, handle signals, implement daemon processes, or build multiprocess architectures.
---

# Linux Process Management & IPC

**Core principle**: Processes are isolated by default. IPC mechanisms break that isolation deliberately — choose the simplest mechanism that meets your needs. Pipes for parent-child, shared memory for throughput, sockets for unrelated processes. Don't use shared memory when a pipe suffices.

Process lifecycle, signals, and inter-process communication for C, Rust, and Zig on Linux.

## Step 1: Detect language

Check file extensions and build files:
- `*.c`, `*.h`, `Makefile` → C workflow
- `*.rs`, `Cargo.toml` → Rust workflow
- `*.zig`, `build.zig` → Zig workflow

## Step 2: C workflow

### fork/exec — process creation

```c
#include <unistd.h>
#include <sys/wait.h>
#include <stdlib.h>

pid_t pid = fork();
if (pid == 0) {
    /* Child */
    execlp("ls", "ls", "-la", NULL);
    perror("execlp");  /* only reached on error */
    _exit(127);
} else if (pid > 0) {
    /* Parent */
    int status;
    waitpid(pid, &status, 0);
    if (WIFEXITED(status))
        printf("child exited %d\n", WEXITSTATUS(status));
} else {
    perror("fork");
}
```

**Common exec variants**:
- `execlp(file, arg0, ..., NULL)` — searches PATH
- `execv(path, argv)` — explicit path + argv array
- `execve(path, argv, envp)` — explicit environment
- `posix_spawn()` — lighter alternative to fork+exec

### Daemon process

```c
#include <unistd.h>
#include <sys/stat.h>
#include <fcntl.h>

/* Double-fork to daemonize */
pid_t pid = fork();
if (pid > 0) _exit(0);    /* parent exits */
setsid();                   /* new session, lose controlling terminal */
pid = fork();
if (pid > 0) _exit(0);    /* first child exits */

/* Redirect stdio to /dev/null */
int fd = open("/dev/null", O_RDWR);
dup2(fd, 0); dup2(fd, 1); dup2(fd, 2);
close(fd);

umask(022);
chdir("/");

/* Now running as daemon */
```

### Signal handling (sigaction)

```c
#include <signal.h>
#include <stdio.h>

static volatile sig_atomic_t got_sigint = 0;

static void handler(int sig)
{
    if (sig == SIGINT)
        got_sigint = 1;  /* async-signal-safe only */
}

/* Setup */
struct sigaction sa = {
    .sa_handler = handler,
    .sa_flags = SA_RESTART,  /* restart interrupted syscalls */
};
sigemptyset(&sa.sa_mask);
sigaction(SIGINT, &sa, NULL);
sigaction(SIGTERM, &sa, NULL);

/* Main loop */
while (!got_sigint) {
    /* ... work ... */
}
```

**signalfd** (event-driven signal handling):
```c
#include <sys/signalfd.h>

sigset_t mask;
sigemptyset(&mask);
sigaddset(&mask, SIGINT);
sigaddset(&mask, SIGTERM);
sigprocmask(SIG_BLOCK, &mask, NULL);  /* block traditional delivery */

int sfd = signalfd(-1, &mask, 0);
/* Now read sfd with read()/epoll for signal events */
struct signalfd_siginfo fdsi;
read(sfd, &fdsi, sizeof(fdsi));
printf("Got signal %d\n", fdsi.ssi_signo);
```

### Pipes

```c
/* Anonymous pipe (parent-child) */
int pipefd[2];
pipe(pipefd);
pid_t pid = fork();
if (pid == 0) {
    close(pipefd[0]);          /* close read end */
    write(pipefd[1], "hello", 5);
    close(pipefd[1]);
    _exit(0);
} else {
    close(pipefd[1]);          /* close write end */
    char buf[64];
    ssize_t n = read(pipefd[0], buf, sizeof(buf));
    close(pipefd[0]);
    waitpid(pid, NULL, 0);
}
```

### Named pipes (FIFO)

```c
#include <sys/stat.h>
mkfifo("/tmp/myfifo", 0666);

/* Writer */
int fd = open("/tmp/myfifo", O_WRONLY);
write(fd, "hello", 5);
close(fd);

/* Reader */
int fd = open("/tmp/myfifo", O_RDONLY);
char buf[64];
read(fd, buf, sizeof(buf));
close(fd);
```

### Shared memory (POSIX)

```c
#include <sys/mman.h>
#include <fcntl.h>

/* Create */
int fd = shm_open("/myshm", O_CREAT | O_RDWR, 0666);
ftruncate(fd, 4096);
void *ptr = mmap(NULL, 4096, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
close(fd);  /* fd no longer needed after mmap */

/* Use ptr for shared data */
*(int *)ptr = 42;

/* Cleanup */
munmap(ptr, 4096);
shm_unlink("/myshm");
```

### Semaphores (POSIX)

```c
#include <semaphore.h>
#include <fcntl.h>

sem_t *sem = sem_open("/mysem", O_CREAT, 0666, 1);  /* binary semaphore */
sem_wait(sem);    /* decrement (block if 0) */
/* critical section */
sem_post(sem);    /* increment */
sem_close(sem);
sem_unlink("/mysem");
```

### Message queues (POSIX)

```c
#include <mqueue.h>

/* Create */
struct mq_attr attr = { .mq_maxmsg = 10, .mq_msgsize = 256 };
mqd_t mq = mq_open("/myqueue", O_CREAT | O_WRONLY, 0666, &attr);

/* Send */
mq_send(mq, "hello", 5, 0);

/* Receive */
char buf[256];
unsigned int prio;
mq_receive(mq, buf, sizeof(buf), &prio);

/* Cleanup */
mq_close(mq);
mq_unlink("/myqueue");
```

### Unix domain sockets

```c
#include <sys/socket.h>
#include <sys/un.h>

/* Server */
int sfd = socket(AF_UNIX, SOCK_STREAM, 0);
struct sockaddr_un addr = { .sun_family = AF_UNIX };
strcpy(addr.sun_path, "/tmp/mysock");
bind(sfd, (struct sockaddr *)&addr, sizeof(addr));
listen(sfd, 5);

int cfd = accept(sfd, NULL, NULL);
char buf[64];
read(cfd, buf, sizeof(buf));
close(cfd);
close(sfd);
unlink("/tmp/mysock");

/* Client */
int sfd = socket(AF_UNIX, SOCK_STREAM, 0);
struct sockaddr_un addr = { .sun_family = AF_UNIX };
strcpy(addr.sun_path, "/tmp/mysock");
connect(sfd, (struct sockaddr *)&addr, sizeof(addr));
write(sfd, "hello", 5);
close(sfd);
```

### Compile with IPC support

```bash
gcc -o app app.c -lrt -lpthread    # -lrt for shm, mq, sem
# For semaphores: -lpthread
```

## Step 3: Rust workflow

### Process spawning

```rust
use std::process::Command;

// Simple
let output = Command::new("ls")
    .arg("-la")
    .output()
    .expect("failed to execute");
println!("{}", String::from_utf8_lossy(&output.stdout));

// With pipes
let child = Command::new("grep")
    .arg("pattern")
    .stdin(Stdio::piped())
    .stdout(Stdio::piped())
    .spawn()?;

child.stdin.unwrap().write_all(b"hello\nworld\n")?;
let output = child.wait_with_output()?;
```

### Signals (nix crate)

```toml
[dependencies]
nix = { version = "0.29", features = ["signal", "process"] }
```

```rust
use nix::sys::signal::{self, Signal, SigHandler, SigAction, SaFlags, SigSet};
use nix::unistd::Pid;

unsafe {
    let handler = SigHandler::Handler(handle_sigint);
    let sa = SigAction::new(handler, SaFlags::SA_RESTART, SigSet::empty());
    signal::sigaction(Signal::SIGINT, &sa).unwrap();
}

extern "C" fn handle_sigint(sig: i32) {
    // async-signal-safe only
}
```

### Shared memory (nix)

```rust
use nix::sys::mman::{mmap, shm_open, shm_unlink, MapFlags, ProtFlags};
use nix::fcntl::OFlag;
use std::num::NonZeroUsize;

let fd = shm_open("/myshm", OFlag::O_CREAT | OFlag::O_RDWR, Mode::S_IRWXU)?;
ftruncate(&fd, 4096)?;
let ptr = unsafe {
    mmap(
        None,
        NonZeroUsize::new(4096).unwrap(),
        ProtFlags::PROT_READ | ProtFlags::PROT_WRITE,
        MapFlags::MAP_SHARED,
        &fd,
        0,
    )?
};
shm_unlink("/myshm")?;
```

### Unix sockets (std)

```rust
use std::os::unix::net::{UnixListener, UnixStream};
use std::io::{Read, Write};

// Server
let listener = UnixListener::bind("/tmp/mysock")?;
for stream in listener.incoming() {
    let mut stream = stream?;
    let mut buf = [0u8; 64];
    let n = stream.read(&mut buf)?;
    println!("received: {}", String::from_utf8_lossy(&buf[..n]));
}

// Client
let mut stream = UnixStream::connect("/tmp/mysock")?;
stream.write_all(b"hello")?;
```

## Step 4: Zig workflow

### Process spawning

```zig
const std = @import("std");

var child = try std.process.Child.run(.{
    .argv = &[_][]const u8{ "ls", "-la" },
    .allocator = std.heap.page_allocator,
});
defer std.process.Child.runFree(child);

std.debug.print("stdout: {s}\n", .{child.stdout});
```

### Signals

```zig
const os = std.os;

const act = os.Sigaction{
    .handler = .{ .handler = handleSignal },
    .mask = os.empty_sigset,
    .flags = 0,
};
try os.sigaction(os.SIG.INT, &act, null);

fn handleSignal(sig: i32) callconv(.C) void {
    // handle signal
}
```

### Pipes

```zig
const pipe = try std.posix.pipe();
const pid = try std.posix.fork();

if (pid == 0) {
    std.posix.close(pipe[0]); // close read end
    _ = try std.posix.write(pipe[1], "hello");
    std.posix.close(pipe[1]);
    std.process.exit(0);
} else {
    std.posix.close(pipe[1]); // close write end
    var buf: [64]u8 = undefined;
    const n = try std.posix.read(pipe[0], &buf);
    std.posix.close(pipe[0]);
}
```

## Anti-patterns

- **Zombie processes** — Always `waitpid()` on children. Or use `signal(SIGCHLD, SIG_IGN)` to auto-reap.
- **Forgetting to close pipe ends** — Close the read end in the writer, close the write end in the reader. Leaked ends block the other process.
- **Unsafe functions in signal handlers** — Only async-signal-safe functions allowed. No `malloc`, `printf`, `locks`, or `errno`. Use `volatile sig_atomic_t` flags.
- **Missing error check on `fork()`** — Always check `fork()` return for `< 0`. The `== 0` (child) and `> 0` (parent) branches must both be handled.
- **Named pipe / shared memory cleanup** — Always `shm_unlink`, `mq_unlink`, `sem_unlink` in the creator process. Leaked names persist across reboots.

## IPC mechanism selection guide

| Mechanism | Use case | Performance | Complexity |
|---|---|---|---|
| **Pipe** | Parent-child, one-directional | Fast (kernel buffer) | Low |
| **Named pipe (FIFO)** | Unrelated processes, one-directional | Fast | Low |
| **Unix socket** | Bidirectional, unrelated processes | Fast | Medium |
| **Shared memory** | High-throughput shared data | Fastest (zero-copy) | High (need sync) |
| **Message queue** | Structured messages with priority | Medium | Medium |
| **Semaphore** | Synchronization only | Fast | Low |
| **Signal** | Async notification (not data) | N/A | Medium |

## Step 5: Verify

- [ ] C: compiles with `-lrt -lpthread`, no valgrind errors
- [ ] Rust: `cargo clippy` passes, `cargo test` passes
- [ ] Zig: `zig build` passes
- [ ] No zombie processes (all children waited on)
- [ ] No leaked file descriptors (check `/proc/<pid>/fd`)
- [ ] No leaked shared memory / semaphores (`ipcs -a`, `ls /dev/shm`)
- [ ] Signal handlers are async-signal-safe (no malloc, no printf, no locks)
