---
name: scaffold-kernel-module
description: Scaffold Linux kernel modules and drivers (C, Rust, Zig) with correct boilerplate. Supports PCIe drivers, character devices, misc devices, platform drivers, Rust kernel modules, and Zig projects. Use when user wants to create a new kernel module, driver skeleton, or systems project.
---

# Scaffold Kernel Module

Generate boilerplate for kernel modules, drivers, and systems projects in C, Rust, or Zig.

## Step 1: Ask user what to scaffold

Present these options:

| Type | When to use |
|---|---|
| **C kernel module** | Minimal out-of-tree module (init/exit) |
| **C character device** | Char driver with file_operations (open/read/write/ioctl) |
| **C misc device** | Simple /dev entry with minor number auto-assign |
| **C platform driver** | SoC/platform device (DT or ACPI enumerated) |
| **C PCIe driver** | PCIe device driver (BAR mapping, MSI-X, DMA) |
| **Rust kernel module** | Rust out-of-tree module using the kernel crate |
| **Rust userspace** | Rust project (Cargo-based) |
| **Zig project** | Zig library or executable |

Ask: "What type of module/driver do you want to scaffold?" and "What name should it have?"

## Step 2: Detect kernel source tree

Check if the current directory is a Linux kernel tree:

- Look for `Kconfig`, `Makefile` at root with `KERNELRELEASE`
- Look for `include/linux/`, `drivers/`, `arch/`
- If not a kernel tree, scaffold as out-of-tree module with its own Makefile

## Step 3: Scaffold based on type

### C kernel module (minimal)

Create `<name>.c`:

```c
#include <linux/module.h>
#include <linux/init.h>

static int __init <name>_init(void)
{
	pr_info("<name>: loaded\n");
	return 0;
}

static void __exit <name>_exit(void)
{
	pr_info("<name>: unloaded\n");
}

module_init(<name>_init);
module_exit(<name>_exit);
MODULE_LICENSE("GPL");
MODULE_AUTHOR("Your Name");
MODULE_DESCRIPTION("<name> module");
```

Create `Makefile` (out-of-tree):

```makefile
obj-m += <name>.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	$(MAKE) -C $(KDIR) M=$(PWD) modules

clean:
	$(MAKE) -C $(KDIR) M=$(PWD) clean
```

If in-tree, add to the appropriate `Kconfig` and `Makefile` in the subsystem directory.

### C character device

Create `<name>.c` with:
- `file_operations` struct (open, release, read, write, unlocked_ioctl)
- `cdev` and `dev_t` management
- `class_create` / `device_create` for auto /dev entry
- Module init: alloc_chrdev_region, cdev_init, cdev_add, class_create, device_create
- Module exit: reverse order cleanup
- Makefile

### C misc device

Create `<name>.c` with:
- `miscdevice` struct (minor = MISC_DYNAMIC_MINOR, fops, name)
- `misc_register` / `misc_deregister`
- Minimal file_operations (read, write, ioctl)
- Makefile

### C platform driver

Create `<name>.c` with:
- `platform_driver` struct (.probe, .remove, .driver.name, .driver.of_match_table)
- `of_device_id` table with compatible strings
- probe: request resources, ioremap, register device
- remove: reverse cleanup
- `module_platform_driver()` macro
- Makefile

### C PCIe driver

See `PCIe-DRIVER.md` for the full template. Key files:
- `<name>.c` — driver with pci_driver struct, probe/remove, BAR, IRQ, DMA
- `<name>.h` — device private data struct, register definitions
- `Makefile` — Kbuild

### Rust kernel module

See `RUST-MODULE.md` for the full template. Key files:
- `<name>.rs` — module with `module!` macro, `impl kernel::Module`
- `Kconfig` — config entry
- `Makefile` — Kbuild with Rust targets

### Rust userspace

Run:
```bash
cargo init --lib --name <name>
# or
cargo init --name <name>  # for binary
```

Create `Cargo.toml`, `src/lib.rs` (or `src/main.rs`), `tests/`.

### Zig project

Run:
```bash
zig init-exe --name <name>
# or
zig init-lib --name <name>
```

Creates `build.zig`, `build.zig.zon`, `src/main.zig` (or `src/root.zig`).

## Step 4: Verify

- [ ] For C: `make` compiles (or `make M=<path>` for out-of-tree)
- [ ] For Rust kernel: `make LLVM=1 M=<path>` compiles
- [ ] For Rust userspace: `cargo check` passes
- [ ] For Zig: `zig build` passes
- [ ] Module loads (if applicable): `insmod <name>.ko` / `rmmod <name>`

## Notes

- Always use `MODULE_LICENSE("GPL")` — required for EXPORT_SYMBOL_GPL and DMA APIs
- For PCIe: reference `PCIe-DRIVER.md` for the full probe/remove/BAR/IRQ/DMA template
- For Rust kernel: reference `RUST-MODULE.md` for the module! macro and kernel crate patterns
- In-tree modules go under `drivers/<subsystem>/` with proper Kconfig + Makefile entries
