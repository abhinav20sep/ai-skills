# Rust Kernel Module Template Reference

Scaffold for a Rust out-of-tree Linux kernel module.

## Prerequisites

- Linux kernel source tree with Rust support enabled (`CONFIG_RUST=y`)
- Rust toolchain matching the kernel's requirements (check `scripts/min-tool-version.sh`)
- `bindgen` and `pahole` installed

## File structure

```
<name>/
├── <name>.rs       # Module implementation
├── Kconfig          # Kernel config entry
├── Makefile         # Kbuild
└── rust-out-of-tree.mk  # Include from kernel (if out-of-tree)
```

## `<name>.rs`

```rust
// SPDX-License-Identifier: GPL-2.0

//! <name> kernel module
//!
//! Brief description of what this module does.

use kernel::prelude::*;

module! {
    type: <Name>,
    name: "<name>",
    author: "Your Name",
    description: "<name> module description",
    license: "GPL",
}

struct <Name> {
    // Module state goes here
}

impl kernel::Module for <Name> {
    fn init(_name: &CStr, _module: &'static ThisModule) -> Result<Self> {
        pr_info!("<name>: loaded\n");
        Ok(<Name> {})
    }
}

impl Drop for <Name> {
    fn drop(&mut self) {
        pr_info!("<name>: unloaded\n");
    }
}
```

## `Kconfig`

```
config <NAME>
	tristate "<name> module"
	depends on RUST
	help
	  Brief description of what this module does.

	  To compile as a module, choose M here: the module will be called
	  <name>.
```

## `Makefile`

```makefile
# Out-of-tree build
obj-m += <name>.o

# For in-tree, add to parent Makefile:
# obj-$(CONFIG_<NAME>) += <name>.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	$(MAKE) -C $(KDIR) M=$(PWD) LLVM=1 modules

clean:
	$(MAKE) -C $(KDIR) M=$(PWD) LLVM=1 clean
```

## Common patterns

### Registering a character device

```rust
use kernel::file::{self, File, Operations};
use kernel::io_buffer::{IoBufferReader, IoBufferWriter};

struct <Name>Device;

#[vtable]
impl Operations for <Name>Device {
    fn open(_file: &File) -> Result<Self::Data> {
        Ok(())
    }

    fn read(_data: &mut (), file: &File, writer: &mut impl IoBufferWriter, offset: u64) -> Result<usize> {
        // Handle read
        Ok(0)
    }

    fn write(_data: &mut (), file: &File, reader: &mut impl IoBufferReader, offset: u64) -> Result<usize> {
        // Handle write
        Ok(0)
    }
}
```

### Platform driver

```rust
use kernel::platform;

struct <Name>Driver;

#[vtable]
impl platform::Driver for <Name>Driver {
    type Data = Box<Name>;

    fn probe(pdev: &mut platform::Device, _info: &Self::Info) -> Result<Box<Name>> {
        // Probe logic
        Ok(Box::try_new(Name {})?)
    }

    fn remove(_data: &Box<Name>) {
        // Cleanup
    }
}

kernel::module_platform_driver! {
    type: <Name>Driver,
    name: "<name>",
    authors: ["Your Name"],
    description: "<name> platform driver",
    license: "GPL",
    of_table: <name>_of_match,
}
```

### Error handling

Rust kernel modules use `Result<T>` (which is `core::result::Result<T, kernel::error::Error>`). The `?` operator propagates errors. Common errors:

- `ENOMEM` — allocation failed (`try_*` functions)
- `EINVAL` — invalid argument
- `ENODEV` — device not found
- `EIO` — I/O error

### Allocations

- `KBox::try_new(value)?` — heap allocation (like `Box::new` but fallible)
- `KVec::try_new()?` — dynamic array (like `Vec::new` but fallible)
- `CStr::try_from(b"string\0")?` — C string from bytes

### Printing

- `pr_info!("format\n", ...)` — kernel INFO level
- `pr_warn!("format\n", ...)` — kernel WARN level
- `pr_err!("format\n", ...)` — kernel ERR level
- `dev_info!(dev, "format\n", ...)` — device-specific INFO

## Building

```bash
# In-tree (from kernel root):
make LLVM=1 M=drivers/misc/<name>

# Out-of-tree:
cd <name>/
make KDIR=/path/to/kernel LLVM=1

# Load:
sudo insmod <name>.ko
sudo rmmod <name>

# Check:
dmesg | tail
```

## Verification

- [ ] `make LLVM=1` compiles without errors
- [ ] `insmod` loads the module
- [ ] `dmesg` shows init message
- [ ] `rmmod` unloads cleanly
- [ ] `dmesg` shows exit message
- [ ] No lockdep warnings, no sparse warnings
