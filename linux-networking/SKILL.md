---
name: linux-networking
description: Linux network programming patterns in C (sockets, raw sockets, netlink), Rust (std::net, tokio), and Zig. Covers TCP/UDP client/server, raw sockets, netlink sockets, TCP tuning, and TLS integration. Use when building network servers, clients, packet capture tools, or kernel-userspace control channels.
---

# Linux Network Programming

**Core principle**: Network I/O is inherently asynchronous. Use non-blocking sockets with an event loop for servers. Use blocking sockets only for simple clients. Never block in an event loop — it freezes all connections.

Socket programming patterns for C, Rust, and Zig on Linux.

## Step 1: Detect language

Check file extensions and build files:
- `*.c`, `*.h`, `Makefile` → C workflow
- `*.rs`, `Cargo.toml` → Rust workflow
- `*.zig`, `build.zig` → Zig workflow

## Step 2: C workflow

### TCP client

```c
#include <sys/socket.h>
#include <netinet/in.h>
#include <netinet/tcp.h>
#include <netdb.h>
#include <unistd.h>

int tcp_connect(const char *host, uint16_t port)
{
    struct addrinfo hints = { .ai_family = AF_UNSPEC, .ai_socktype = SOCK_STREAM };
    struct addrinfo *res;
    char port_str[6];
    snprintf(port_str, sizeof(port_str), "%u", port);

    int err = getaddrinfo(host, port_str, &hints, &res);
    if (err)
        return -1;

    int fd = socket(res->ai_family, res->ai_socktype, res->ai_protocol);
    if (fd < 0) { freeaddrinfo(res); return -1; }

    if (connect(fd, res->ai_addr, res->ai_addrlen) < 0) {
        close(fd);
        freeaddrinfo(res);
        return -1;
    }
    freeaddrinfo(res);
    return fd;
}
```

### UDP client/server

```c
#include <sys/socket.h>
#include <netinet/in.h>

/* UDP server */
int sfd = socket(AF_INET, SOCK_DGRAM, 0);
struct sockaddr_in addr = { .sin_family = AF_INET, .sin_port = htons(5353), .sin_addr.s_addr = INADDR_ANY };
bind(sfd, (struct sockaddr *)&addr, sizeof(addr));

char buf[1500];
struct sockaddr_in client;
socklen_t client_len = sizeof(client);
ssize_t n = recvfrom(sfd, buf, sizeof(buf), 0, (struct sockaddr *)&client, &client_len);

sendto(sfd, buf, n, 0, (struct sockaddr *)&client, client_len);

/* UDP client */
int sfd = socket(AF_INET, SOCK_DGRAM, 0);
struct sockaddr_in server = { .sin_family = AF_INET, .sin_port = htons(5353) };
inet_pton(AF_INET, "192.168.1.1", &server.sin_addr);
sendto(sfd, "hello", 5, 0, (struct sockaddr *)&server, sizeof(server));
ssize_t n = recvfrom(sfd, buf, sizeof(buf), 0, NULL, NULL);
```

### Raw sockets — packet capture

```c
#include <sys/socket.h>
#include <linux/if_packet.h>
#include <net/ethernet.h>

/* Capture all Ethernet frames */
int sfd = socket(AF_PACKET, SOCK_RAW, htons(ETH_P_ALL));
if (sfd < 0) { perror("socket"); return 1; }

/* Bind to specific interface */
struct sockaddr_ll addr = { .sll_family = AF_PACKET, .sll_protocol = htons(ETH_P_ALL) };
addr.sll_ifindex = if_nametoindex("eth0");
bind(sfd, (struct sockaddr *)&addr, sizeof(addr));

/* Receive frames */
unsigned char buf[65536];
ssize_t n = recvfrom(sfd, buf, sizeof(buf), 0, NULL, NULL);
/* buf[0..5] = dst MAC, buf[6..11] = src MAC, buf[12..13] = EtherType */

close(sfd);
```

### Netlink sockets — kernel-userspace control

```c
#include <linux/netlink.h>
#include <sys/socket.h>

/* Netlink socket for route information */
int sfd = socket(AF_NETLINK, SOCK_RAW, NETLINK_ROUTE);
struct sockaddr_nl addr = { .nl_family = AF_NETLINK, .nl_pid = getpid() };
bind(sfd, (struct sockaddr *)&addr, sizeof(addr));

/* Send request */
struct {
    struct nlmsghdr nlh;
    struct rtmsg rtm;
} req = {
    .nlh.nlmsg_len = sizeof(req),
    .nlh.nlmsg_type = RTM_GETROUTE,
    .nlh.nlmsg_flags = NLM_F_REQUEST | NLM_F_DUMP,
    .nlh.nlmsg_seq = 1,
    .rtm.rtm_family = AF_INET,
};
send(sfd, &req, sizeof(req), 0);

/* Receive response */
char buf[8192];
ssize_t n = recv(sfd, buf, sizeof(buf), 0);
/* Parse nlmsghdr chain from buf */
```

### TCP tuning

```c
int fd = socket(AF_INET, SOCK_STREAM, 0);

/* Disable Nagle's algorithm (send immediately) */
int opt = 1;
setsockopt(fd, IPPROTO_TCP, TCP_NODELAY, &opt, sizeof(opt));

/* Enable TCP_CORK (batch writes, flush on uncork or timeout) */
setsockopt(fd, IPPROTO_TCP, TCP_CORK, &opt, sizeof(opt));

/* Adjust send/receive buffers */
int sndbuf = 262144;  /* 256KB */
setsockopt(fd, SOL_SOCKET, SO_SNDBUF, &sndbuf, sizeof(sndbuf));
int rcvbuf = 262144;
setsockopt(fd, SOL_SOCKET, SO_RCVBUF, &rcvbuf, sizeof(rcvbuf));

/* Quick ACK (disable delayed ACK) */
setsockopt(fd, IPPROTO_TCP, TCP_QUICKACK, &opt, sizeof(opt));

/* Defer accept (don't wake until data arrives) */
setsockopt(fd, IPPROTO_TCP, TCP_DEFER_ACCEPT, &opt, sizeof(opt));
```

**TCP_NODELAY vs TCP_CORK**:
| Option | Behavior | Use case |
|---|---|---|
| `TCP_NODELAY` | Send immediately, no buffering | Low-latency (interactive, gaming) |
| `TCP_CORK` | Batch writes, send when uncorked or 200ms timeout | High-throughput (HTTP responses, file transfer) |
| Neither (default) | Nagle's algorithm: batch small writes until ACK | General purpose |

### TLS with OpenSSL

```c
#include <openssl/ssl.h>
#include <openssl/err.h>

SSL_CTX *ctx = SSL_CTX_new(TLS_client_method());
SSL_CTX_set_min_proto_version(ctx, TLS1_2_VERSION);

SSL *ssl = SSL_new(ctx);
int fd = tcp_connect("example.com", 443);
SSL_set_fd(ssl, fd);
SSL_set_tlsext_host_name(ssl, "example.com");  /* SNI */

if (SSL_connect(ssl) <= 0) {
    ERR_print_errors_fp(stderr);
    return 1;
}

SSL_write(ssl, "GET / HTTP/1.0\r\nHost: example.com\r\n\r\n", 37);
char buf[4096];
int n = SSL_read(ssl, buf, sizeof(buf));

SSL_shutdown(ssl);
SSL_free(ssl);
close(fd);
SSL_CTX_free(ctx);
```

## Step 3: Rust workflow

### UDP

```rust
use std::net::UdpSocket;

// Server
let socket = UdpSocket::bind("0.0.0.0:5353")?;
let mut buf = [0u8; 1500];
let (n, src) = socket.recv_from(&mut buf)?;
socket.send_to(&buf[..n], src)?;

// Client
let socket = UdpSocket::bind("0.0.0.0:0")?;
socket.send_to(b"hello", "192.168.1.1:5353")?;
let n = socket.recv(&mut buf)?;
```

### tokio UDP

```rust
use tokio::net::UdpSocket;

let socket = UdpSocket::bind("0.0.0.0:5353").await?;
let mut buf = [0u8; 1500];
let (n, src) = socket.recv_from(&mut buf).await?;
socket.send_to(&buf[..n], src).await?;
```

### TLS with tokio

```toml
[dependencies]
tokio = { version = "1", features = ["full"] }
tokio-rustls = "0.26"
rustls-pemfile = "2"
```

```rust
use tokio_rustls::TlsConnector;
use rustls::ClientConfig;
use std::sync::Arc;

let mut roots = rustls::RootCertStore::empty();
roots.extend(webpki_roots::TLS_SERVER_ROOTS.iter().cloned());
let config = ClientConfig::builder()
    .with_root_certificates(roots)
    .with_no_client_auth();
let connector = TlsConnector::from(Arc::new(config));

let stream = tokio::net::TcpStream::connect("example.com:443").await?;
let domain = "example.com".try_into()?;
let mut stream = connector.connect(domain, stream).await?;

use tokio::io::{AsyncReadExt, AsyncWriteExt};
stream.write_all(b"GET / HTTP/1.0\r\nHost: example.com\r\n\r\n").await?;
let mut buf = vec![0u8; 4096];
let n = stream.read(&mut buf).await?;
```

## Step 4: Zig workflow

### UDP

```zig
const std = @import("std");
const net = std.net;

const address = try net.Address.resolveIp("0.0.0.0", 5353);
const sock = try std.posix.socket(address.any.family, std.posix.SOCK.DGRAM, 0);
try std.posix.bind(sock, &address.any, address.getOsSockLen());

var buf: [1500]u8 = undefined;
const result = try std.posix.recvfrom(sock, &buf, 0, null, null);
```

### TCP client

```zig
const stream = try std.net.tcpConnectToHost(allocator, "example.com", 80);
defer stream.close();
try stream.writeAll("GET / HTTP/1.0\r\nHost: example.com\r\n\r\n");
var buf: [4096]u8 = undefined;
const n = try stream.read(&buf);
```

## Anti-patterns

- **Missing SO_REUSEADDR** — Without it, server restart fails with "Address already in use" during TIME_WAIT.
- **Blocking DNS in event loop** — `getaddrinfo` blocks. Use async DNS or a dedicated resolver thread.
- **No error check on socket/accept** — Always check return values. `-1` means failure, handle it.
- **Disabling TLS certificate verification** — `SSL_CTX_set_verify(ctx, SSL_VERIFY_NONE, NULL)` defeats the purpose of TLS. Always verify.
- **Hardcoded buffer sizes** — Use `MSG_PEEK` or length-prefixed protocols instead of assuming message size.

## Step 5: Verify

- [ ] C: compiles without warnings, valgrind clean
- [ ] Rust: `cargo clippy` passes, `cargo test` passes
- [ ] Zig: `zig build` passes
- [ ] No file descriptor leaks on error paths
- [ ] Sockets have `SO_REUSEADDR` set for server rebind
- [ ] Non-blocking sockets used with event loops
- [ ] TLS certificates validated (no `SSL_CTX_set_verify(ctx, 0, NULL)`)
