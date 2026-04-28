---
name: linux-fileio
description: Linux file I/O patterns in C (direct I/O, async I/O, file locking, sendfile, mmap file I/O), Rust, and Zig. Covers O_DIRECT, io_uring for files, flock, fcntl locks, zero-copy transfer, and memory-mapped file access. Use when building high-performance file processing, databases, or log processing systems.
---

# Linux File I/O Patterns

**Core principle**: The page cache is your friend — until it isn't. Use buffered I/O for general workloads. Use O_DIRECT for databases that manage their own cache. Use sendfile/splice for zero-copy transfer. Never read+write when sendfile suffices.

Advanced file I/O patterns for C, Rust, and Zig on Linux.

## Step 1: Detect language

Check file extensions and build files:
- `*.c`, `*.h`, `Makefile` → C workflow
- `*.rs`, `Cargo.toml` → Rust workflow
- `*.zig`, `build.zig` → Zig workflow

## Step 2: C workflow

### Direct I/O (O_DIRECT)

Bypasses the page cache. Data goes directly between userspace buffer and disk. Required for databases and high-performance I/O.

```c
#include <fcntl.h>
#include <unistd.h>
#include <stdlib.h>

/* Open with O_DIRECT */
int fd = open("data.bin", O_RDWR | O_DIRECT);
if (fd < 0) { perror("open"); return 1; }

/* Buffer MUST be aligned (typically 512 or 4096 bytes) */
void *buf;
posix_memalign(&buf, 4096, 4096);  /* 4096-byte aligned, 4096 bytes */
memset(buf, 0, 4096);

/* Read must use aligned offset and length */
ssize_t n = pread(fd, buf, 4096, 0);  /* offset=0, len=4096 */
if (n < 0) perror("pread");

/* Write must also be aligned */
pwrite(fd, buf, 4096, 0);

free(buf);
close(fd);
```

**O_DIRECT rules**:
- Buffer address must be aligned (512 for most disks, 4096 for NVMe)
- Transfer size must be aligned
- File offset must be aligned
- Use `posix_memalign()` or `aligned_alloc()` for buffer allocation

### File locking

```c
#include <sys/file.h>
#include <fcntl.h>

/* flock — whole-file advisory lock */
int fd = open("data.db", O_RDWR);

flock(fd, LOCK_SH);   /* shared lock (multiple readers) */
/* ... read ... */
flock(fd, LOCK_UN);   /* unlock */

flock(fd, LOCK_EX);   /* exclusive lock (one writer) */
/* ... write ... */
flock(fd, LOCK_UN);

/* Non-blocking */
if (flock(fd, LOCK_EX | LOCK_NB) < 0) {
    if (errno == EWOULDBLOCK) {
        /* Lock held by another process */
    }
}
```

```c
#include <fcntl.h>

/* fcntl — byte-range lock (POSIX) */
struct flock fl = {
    .l_type = F_WRLCK,      /* F_RDLCK (read), F_WRLCK (write), F_UNLCK (unlock) */
    .l_whence = SEEK_SET,
    .l_start = 0,            /* lock start offset */
    .l_len = 1024,           /* lock length (0 = to EOF) */
};

fcntl(fd, F_SETLKW, &fl);   /* blocking */
/* or: fcntl(fd, F_SETLK, &fl);  non-blocking */

/* Query lock owner */
fcntl(fd, F_GETLK, &fl);
if (fl.l_type != F_UNLCK) {
    printf("Lock held by PID %d\n", fl.l_pid);
}
```

**flock vs fcntl**:

| Feature | flock | fcntl |
|---|---|---|
| Scope | Whole file | Byte range |
| Advisory | Yes | Yes |
| Inherited across fork | Yes (but not exec) | No |
| NFS support | Yes | Yes |
| Per-fd vs per-file | Per open file description | Per process |

### sendfile — zero-copy file-to-socket

```c
#include <sys/sendfile.h>
#include <fcntl.h>
#include <sys/stat.h>

/* Zero-copy: file → socket, no userspace copy */
int file_fd = open("data.bin", O_RDONLY);
struct stat st;
fstat(file_fd, &st);

off_t offset = 0;
sendfile(sock_fd, file_fd, &offset, st.st_size);

close(file_fd);
```

**sendfile vs read+write**:
- `sendfile`: kernel copies file data directly to socket buffer (zero-copy)
- `read+write`: file → userspace buffer → socket (two copies)
- `sendfile` is ~2x faster for large files

### splice — kernel pipe-based zero-copy

```c
#include <fcntl.h>

/* splice: file → pipe → socket (no userspace copy) */
int pipefd[2];
pipe(pipefd);

/* File → pipe */
splice(file_fd, NULL, pipefd[1], NULL, 4096, SPLICE_F_MOVE);

/* Pipe → socket */
splice(pipefd[0], NULL, sock_fd, NULL, 4096, SPLICE_F_MOVE | SPLICE_F_MORE);
```

### Memory-mapped file I/O

```c
#include <sys/mman.h>
#include <sys/stat.h>

int fd = open("data.bin", O_RDWR);
struct stat st;
fstat(fd, &st);

/* Map entire file */
void *ptr = mmap(NULL, st.st_size, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
if (ptr == MAP_FAILED) { perror("mmap"); return 1; }

/* Access file data directly through pointer */
uint32_t val = *(uint32_t *)ptr;  /* read */
*(uint32_t *)ptr = 42;            /* write */

/* Flush changes to disk */
msync(ptr, st.st_size, MS_SYNC);  /* synchronous flush */
/* or: msync(ptr, st.st_size, MS_ASYNC);  asynchronous flush */

/* Prefetch (advise kernel about access pattern) */
madvise(ptr, st.st_size, MADV_SEQUENTIAL);  /* sequential read */
madvise(ptr, st.st_size, MADV_RANDOM);      /* random access */
madvise(ptr, st.st_size, MADV_WILLNEED);    /* prefetch pages */
madvise(ptr, st.st_size, MADV_DONTNEED);    /* release pages */

munmap(ptr, st.st_size);
close(fd);
```

### Async I/O with io_uring (file reads)

```c
#include <liburing.h>

struct io_uring ring;
io_uring_queue_init(64, &ring, 0);

/* Submit async file read */
struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
io_uring_prep_read(sqe, file_fd, buf, 4096, 0);  /* offset 0 */
io_uring_sqe_set_data(sqe, my_context);
io_uring_submit(&ring);

/* Wait for completion */
struct io_uring_cqe *cqe;
io_uring_wait_cqe(&ring, &cqe);
int bytes_read = cqe->res;  /* bytes read or negative error */
io_uring_cqe_seen(&ring, cqe);
```

### Legacy AIO

```c
#include <linux/aio_abi.h>
#include <sys/syscall.h>

/* Create AIO context */
aio_context_t ctx = 0;
syscall(__NR_io_setup, 64, &ctx);

/* Submit read */
struct iocb cb = {
    .aio_fildes = fd,
    .aio_lio_opcode = IOCB_CMD_PREAD,
    .aio_buf = (uint64_t)buf,
    .aio_nbytes = 4096,
    .aio_offset = 0,
};
struct iocb *cbs[1] = { &cb };
syscall(__NR_io_submit, ctx, 1, cbs);

/* Wait for completion */
struct io_event events[1];
syscall(__NR_io_getevents, ctx, 1, 1, events, NULL);

syscall(__NR_io_destroy, ctx);
```

## Step 3: Rust workflow

### Memory-mapped file

```toml
[dependencies]
memmap2 = "0.9"
```

```rust
use memmap2::Mmap;
use std::fs::File;

let file = File::open("data.bin")?;
let mmap = unsafe { Mmap::map(&file)? };

// Access through slice
let val = u32::from_ne_bytes([mmap[0], mmap[1], mmap[2], mmap[3]]);

// Mutable mapping
use memmap2::MmapMut;
let file = File::options().read(true).write(true).open("data.bin")?;
let mut mmap = unsafe { MmapMut::map_mut(&file)? };
mmap[0] = 42;
mmap.flush()?;
```

### File locking

```rust
use fs2::FileExt;
use std::fs::File;

let file = File::open("data.db")?;
file.lock_shared()?;    // flock LOCK_SH
// ... read ...
file.unlock()?;

file.lock_exclusive()?; // flock LOCK_EX
// ... write ...
file.unlock()?;
```

### sendfile

```rust
use std::io::Write;
use std::net::TcpStream;
use std::fs::File;
use std::os::unix::io::AsRawFd;

let file = File::open("data.bin")?;
let mut stream = TcpStream::connect("server:8080")?;
let meta = file.metadata()?;
let sent = nix::fcntl::sendfile(
    stream.as_raw_fd(),
    file.as_raw_fd(),
    None,
    meta.len() as usize,
)?;
```

## Step 4: Zig workflow

### Direct I/O

```zig
const std = @import("std");
const fd = try std.posix.open("data.bin", .{ .ACCMODE = .RDONLY, .DIRECT = true }, 0);
defer std.posix.close(fd);

var buf: [4096]u8 align(4096) = undefined;
const n = try std.posix.pread(fd, &buf, 0);
```

### File locking

```zig
const fd = try std.posix.open("data.db", .{ .ACCMODE = .RDWR }, 0);
defer std.posix.close(fd);

// flock
try std.posix.flock(fd, .EXCLUSIVE);
// ... write ...
try std.posix.flock(fd, .UN);
```

## Anti-patterns

- **Unaligned O_DIRECT buffers** — O_DIRECT requires aligned buffer address, offset, and length. Use `posix_memalign`, not `malloc`.
- **Missing msync before munmap** — Dirty mmap'd pages are lost without `msync`. Always `msync(ptr, len, MS_SYNC)` before `munmap`.
- **File lock not released** — Always release `fcntl` locks on all error paths. Or use `flock` which auto-releases on fd close.
- **sendfile without checking return** — `sendfile` may send fewer bytes than requested. Loop until all bytes are sent.
- **mmap without bounds checking** — Accessing past the mapped region causes SIGBUS. Always check file size against access offset.

## Performance comparison

| Method | Copies | CPU usage | Use case |
|---|---|---|---|
| `read` + `write` | 2 (kernel→user, user→kernel) | High | General purpose |
| `sendfile` | 0 (kernel internal) | Low | File → socket |
| `splice` | 0 (kernel pipe) | Low | File → pipe → socket |
| `mmap` + write | 0 (page cache) | Low | Random access, in-place modify |
| `O_DIRECT` | 0 (bypass page cache) | Medium | Database, raw disk I/O |
| `io_uring` read | 0-1 | Low | Async file I/O at scale |

## Step 5: Verify

- [ ] O_DIRECT buffers are properly aligned (`posix_memalign`)
- [ ] File locks are released on all error paths (or use `flock` which auto-releases on fd close)
- [ ] `mmap` has matching `munmap`
- [ ] `sendfile` offset is updated correctly for partial sends
- [ ] No file descriptor leaks on error paths
- [ ] `msync` called before `munmap` for dirty mmap'd pages
