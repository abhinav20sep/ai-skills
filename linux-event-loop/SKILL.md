---
name: linux-event-loop
description: Event-driven I/O patterns for Linux in C (epoll, select, poll, io_uring), Rust (mio, tokio), and Zig (std.net). Covers non-blocking I/O, event loops, io_uring, and high-performance network server design. Use when user wants to build event-driven servers, implement non-blocking I/O, use io_uring, or design scalable network architectures.
---

# Linux Event Loop Patterns

Event-driven I/O and non-blocking patterns for C, Rust, and Zig on Linux.

## Step 1: Detect language

Check file extensions and build files:
- `*.c`, `*.h`, `Makefile` → C workflow
- `*.rs`, `Cargo.toml` → Rust workflow
- `*.zig`, `build.zig` → Zig workflow

## Step 2: C workflow

### Non-blocking socket setup

```c
#include <fcntl.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <netinet/tcp.h>

int make_nonblocking(int fd)
{
    int flags = fcntl(fd, F_GETFL, 0);
    return fcntl(fd, F_SETFL, flags | O_NONBLOCK);
}

int create_listen_socket(uint16_t port)
{
    int fd = socket(AF_INET, SOCK_STREAM | SOCK_NONBLOCK, 0);
    int opt = 1;
    setsockopt(fd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));
    setsockopt(fd, SOL_SOCKET, SO_REUSEPORT, &opt, sizeof(opt));

    struct sockaddr_in addr = {
        .sin_family = AF_INET,
        .sin_port = htons(port),
        .sin_addr.s_addr = INADDR_ANY,
    };
    bind(fd, (struct sockaddr *)&addr, sizeof(addr));
    listen(fd, SOMAXCONN);
    return fd;
}
```

### epoll (recommended for Linux)

```c
#include <sys/epoll.h>
#include <unistd.h>

#define MAX_EVENTS 64

int epfd = epoll_create1(0);

/* Add listen socket */
struct epoll_event ev = {
    .events = EPOLLIN,
    .data.fd = listen_fd,
};
epoll_ctl(epfd, EPOLL_CTL_ADD, listen_fd, &ev);

/* Event loop */
struct epoll_event events[MAX_EVENTS];
for (;;) {
    int nfds = epoll_wait(epfd, events, MAX_EVENTS, -1);
    for (int i = 0; i < nfds; i++) {
        int fd = events[i].data.fd;
        uint32_t revents = events[i].events;

        if (fd == listen_fd) {
            /* Accept new connection */
            int client = accept4(listen_fd, NULL, NULL, SOCK_NONBLOCK);
            struct epoll_event cev = {
                .events = EPOLLIN | EPOLLET,  /* edge-triggered */
                .data.fd = client,
            };
            epoll_ctl(epfd, EPOLL_CTL_ADD, client, &cev);
        } else if (revents & EPOLLIN) {
            /* Read data */
            char buf[4096];
            ssize_t n = read(fd, buf, sizeof(buf));
            if (n <= 0) {
                close(fd);
                epoll_ctl(epfd, EPOLL_CTL_DEL, fd, NULL);
            } else {
                /* Process and respond */
                write(fd, buf, n);  /* echo */
            }
        }
    }
}
```

**Level-triggered vs edge-triggered**:

| Mode | Flag | Behavior | When to use |
|---|---|---|---|
| Level-triggered | (default) | Reports ready as long as data available | Simple servers, must read all data |
| Edge-triggered | `EPOLLET` | Reports once per state change | High-performance, must read until EAGAIN |

**Edge-triggered read loop** (mandatory with `EPOLLET`):
```c
for (;;) {
    ssize_t n = read(fd, buf, sizeof(buf));
    if (n == -1) {
        if (errno == EAGAIN || errno == EWOULDBLOCK)
            break;  /* all data read */
        perror("read");
        break;
    }
    if (n == 0) {
        /* EOF */
        close(fd);
        break;
    }
    /* process buf[0..n] */
}
```

### epoll with data pointer (for connection state)

```c
struct connection {
    int fd;
    char buf[4096];
    size_t buf_len;
    /* ... connection state ... */
};

struct connection *conn = malloc(sizeof(*conn));
conn->fd = client_fd;

struct epoll_event ev = {
    .events = EPOLLIN | EPOLLET,
    .data.ptr = conn,  /* store pointer, not fd */
};
epoll_ctl(epfd, EPOLL_CTL_ADD, client_fd, &ev);

/* In event loop */
struct connection *conn = events[i].data.ptr;
int fd = conn->fd;
```

### poll (portable fallback)

```c
#include <poll.h>

struct pollfd fds[MAX_FDS];
fds[0].fd = listen_fd;
fds[0].events = POLLIN;
nfds_t nfds = 1;

for (;;) {
    int ret = poll(fds, nfds, -1);
    for (nfds_t i = 0; i < nfds; i++) {
        if (fds[i].revents & POLLIN) {
            if (fds[i].fd == listen_fd) {
                int client = accept(listen_fd, NULL, NULL);
                fds[nfds].fd = client;
                fds[nfds].events = POLLIN;
                nfds++;
            } else {
                /* read from fds[i].fd */
            }
        }
    }
}
```

### io_uring (Linux 5.1+, highest performance)

```c
#include <liburing.h>

struct io_uring ring;
io_uring_queue_init(256, &ring, 0);

/* Submit a read */
struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
io_uring_prep_recv(sqe, client_fd, buf, sizeof(buf), 0);
io_uring_sqe_set_data(sqe, conn);
io_uring_submit(&ring);

/* Wait for completions */
struct io_uring_cqe *cqe;
io_uring_wait_cqe(&ring, &cqe);
struct connection *conn = io_uring_cqe_get_data(cqe);
int result = cqe->res;  /* bytes read or error */
io_uring_cqe_seen(&ring, cqe);
```

**io_uring advantages over epoll**:
- Zero syscall overhead for submission/completion (ring buffer shared with kernel)
- Batches multiple I/O operations in single submission
- Supports async file I/O (epoll doesn't work with regular files)
- Kernel-side polling mode (`IORING_SETUP_SQPOLL`) eliminates syscalls entirely

### Compile with io_uring

```bash
gcc -o app app.c -luring
```

## Step 3: Rust workflow

### mio (low-level event loop)

```toml
[dependencies]
mio = { version = "1", features = ["net", "os-poll"] }
```

```rust
use mio::net::{TcpListener, TcpStream};
use mio::{Events, Interest, Poll, Token};
use std::collections::HashMap;
use std::io::{Read, Write};

const LISTENER: Token = Token(0);

let mut poll = Poll::new()?;
let mut listener = TcpListener::bind("0.0.0.0:8080".parse()?)?;
poll.registry().register(&mut listener, LISTENER, Interest::READABLE)?;

let mut events = Events::with_capacity(128);
let mut connections: HashMap<Token, TcpStream> = HashMap::new();
let mut next_token = Token(1);

loop {
    poll.poll(&mut events, None)?;

    for event in events.iter() {
        match event.token() {
            LISTENER => {
                let (mut stream, _) = listener.accept()?;
                let token = next_token;
                next_token = Token(next_token.0 + 1);
                poll.registry().register(&mut stream, token, Interest::READABLE)?;
                connections.insert(token, stream);
            }
            token => {
                if let Some(stream) = connections.get_mut(&token) {
                    let mut buf = [0u8; 4096];
                    match stream.read(&mut buf) {
                        Ok(0) => { connections.remove(&token); }
                        Ok(n) => { stream.write_all(&buf[..n])?; }
                        Err(_) => { connections.remove(&token); }
                    }
                }
            }
        }
    }
}
```

### tokio (async runtime)

```toml
[dependencies]
tokio = { version = "1", features = ["full"] }
```

```rust
use tokio::net::{TcpListener, TcpStream};
use tokio::io::{AsyncReadExt, AsyncWriteExt};

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let listener = TcpListener::bind("0.0.0.0:8080").await?;

    loop {
        let (mut stream, _) = listener.accept().await?;
        tokio::spawn(async move {
            let mut buf = [0u8; 4096];
            loop {
                match stream.read(&mut buf).await {
                    Ok(0) => break,
                    Ok(n) => { stream.write_all(&buf[..n]).await.unwrap(); }
                    Err(_) => break,
                }
            }
        });
    }
}
```

**What the compiler sees**: `#[tokio::main]` expands to a `fn main()` that creates a tokio `Runtime` and blocks on the async body. Each `tokio::spawn` creates a lightweight task (green thread) multiplexed onto OS threads.

### tokio with channels and timers

```rust
use tokio::sync::{mpsc, oneshot};
use tokio::time::{sleep, Duration};

let (tx, mut rx) = mpsc::channel(32);

tokio::spawn(async move {
    tx.send("hello").await.unwrap();
});

// Timeout
match tokio::time::timeout(Duration::from_secs(5), rx.recv()).await {
    Ok(Some(msg)) => println!("got: {}", msg),
    Ok(None) => println!("channel closed"),
    Err(_) => println!("timeout"),
}
```

## Step 4: Zig workflow

### Non-blocking TCP server

```zig
const std = @import("std");
const net = std.net;
const posix = std.posix;

const server = try net.Address.resolveIp("0.0.0.0", 8080);
const listener = try posix.socket(posix.AF.INET, posix.SOCK.STREAM | posix.SOCK.NONBLOCK, 0);
try posix.bind(listener, &server.any, server.getOsSockLen());
try posix.listen(listener, 128);

var pollfds = [_]posix.pollfd{
    .{ .fd = listener, .events = posix.POLL.IN, .revents = 0 },
};

while (true) {
    const n = try posix.poll(&pollfds, -1);
    for (pollfds[0..n]) |pfd| {
        if (pfd.revents & posix.POLL.IN != 0) {
            const client = try posix.accept(listener, null, null, posix.SOCK.NONBLOCK);
            // handle client
        }
    }
}
```

## Performance comparison

| Mechanism | Syscalls per I/O | Max FDs | Scalability | Linux-only |
|---|---|---|---|---|
| `select` | 1 (select) | 1024 (FD_SETSIZE) | O(n) scan | No |
| `poll` | 1 (poll) | unlimited | O(n) scan | No |
| `epoll` | 1 (epoll_wait) | unlimited | O(1) event delivery | Yes |
| `io_uring` | 0-1 (ring buffer) | unlimited | O(1), zero-copy | Yes (5.1+) |

**Recommendation**:
- **Linux, high-performance**: `io_uring` (5.1+)
- **Linux, general**: `epoll` with edge-triggered mode
- **Portable**: `poll`
- **Rust async**: `tokio` (uses epoll/io_uring internally)
- **Simple**: `select` (only for <100 connections)

## Step 5: Verify

- [ ] C: compiles with `-luring` if using io_uring
- [ ] No file descriptor leaks (`/proc/<pid>/fd` count stable under load)
- [ ] All sockets set to `O_NONBLOCK` / `SOCK_NONBLOCK`
- [ ] Edge-triggered epoll: read loop runs until `EAGAIN`
- [ ] `SO_REUSEADDR` / `SO_REUSEPORT` set on listen sockets
- [ ] Connection cleanup on EOF and error (close + EPOLL_CTL_DEL)
- [ ] Rust: `cargo clippy` passes, `cargo test` passes
- [ ] Zig: `zig build` passes
- [ ] Load test: `wrk` or `hey` shows stable performance under concurrency
