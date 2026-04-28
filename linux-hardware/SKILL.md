---
name: linux-hardware
description: Hardware access patterns for Linux device drivers in C. Covers MMIO register access, port I/O, device tree parsing, ACPI, regmap API, and interrupt handling (threaded IRQs, IRQ domains). Use when writing platform drivers, SoC drivers, or working with hardware registers and device tree.
---

# Linux Hardware Access Patterns

**Core principle**: Never access hardware registers directly. Use `readl`/`writel` for MMIO, `inb`/`outb` for port I/O, and `regmap` for I2C/SPI. Direct pointer dereference of MMIO regions bypasses barriers and breaks on architectures with posted writes. See [DEVICETREE.md](DEVICETREE.md) for device tree reference.

Device driver hardware access for Linux kernel in C.

## Step 1: Identify hardware interface

| Interface | Detection | API |
|---|---|---|
| MMIO (memory-mapped) | BAR resource, `ioremap` | `readl`/`writel` |
| Port I/O | `request_region` | `inb`/`outb` |
| Device tree | `of_match_table` in driver | `of_property_read_*` |
| ACPI | `acpi_device_id` in driver | `acpi_*` functions |
| I2C/SPI | `i2c_driver` / `spi_driver` | `regmap` API |

## Step 2: MMIO register access

### Basic MMIO

```c
#include <linux/io.h>

/* Map registers (in probe) */
void __iomem *regs = devm_ioremap_resource(dev, res);
if (IS_ERR(regs))
	return PTR_ERR(regs);

/* 32-bit register access */
u32 val = readl(regs + REG_STATUS);
writel(0x01, regs + REG_CONTROL);

/* 16-bit */
u16 val16 = readw(regs + REG_VERSION);

/* 8-bit */
u8 val8 = readb(regs + REG_FLAGS);

/* Write with read-back verification */
writel(0x01, regs + REG_CONTROL);
wmb();  /* ensure write is visible */
u32 verify = readl(regs + REG_CONTROL);
if (verify != 0x01)
	dev_warn(dev, "register write failed\n");
```

### Register access with barriers

```c
/* Ordered register writes */
writel(control_val, regs + REG_CONTROL);
wmb();                              /* ensure control is written before data */
writel(data_val, regs + REG_DATA);

/* Ordered register reads */
u32 status = readl(regs + REG_STATUS);
rmb();                              /* ensure status is read before data */
u32 data = readl(regs + REG_DATA);

/* MMIO write barrier (some architectures need this) */
writel(val, regs + REG_X);
mmiowb();                           /* flush posted MMIO writes */
```

### Set/clear bits

```c
/* Set bits */
u32 val = readl(regs + REG_CONTROL);
val |= CTRL_ENABLE | CTRL_IRQ_EN;
writel(val, regs + REG_CONTROL);

/* Clear bits */
val = readl(regs + REG_CONTROL);
val &= ~CTRL_RESET;
writel(val, regs + REG_CONTROL);

/* Modify with read-modify-write (non-atomic, use with care in IRQ context) */
static inline void reg_set_bits(void __iomem *reg, u32 mask)
{
	u32 val = readl(reg);
	val |= mask;
	writel(val, reg);
}

static inline void reg_clear_bits(void __iomem *reg, u32 mask)
{
	u32 val = readl(reg);
	val &= ~mask;
	writel(val, reg);
}
```

### Port I/O (legacy x86)

```c
#include <linux/io.h>

/* Request port region */
if (!request_region(0x3f8, 8, "serial")) {
	pr_err("port region busy\n");
	return -EBUSY;
}

/* 8-bit */
outb(0x01, 0x3f8);          /* write */
u8 val = inb(0x3f8);        /* read */

/* 16-bit */
outw(0x1234, 0x3f8);
u16 val16 = inw(0x3f8);

/* 32-bit */
outl(0xdeadbeef, 0x3f8);
u32 val32 = inl(0x3f8);

/* Release */
release_region(0x3f8, 8);
```

## Step 3: Device tree

### Matching

```c
static const struct of_device_id my_driver_of_match[] = {
	{ .compatible = "vendor,my-device" },
	{ .compatible = "vendor,my-device-v2" },
	{ },
};
MODULE_DEVICE_TABLE(of, my_driver_of_match);

static struct platform_driver my_driver = {
	.probe = my_probe,
	.remove = my_remove,
	.driver = {
		.name = "my-driver",
		.of_match_table = my_driver_of_match,
	},
};
```

### Reading properties

```c
struct device_node *np = dev->of_node;
u32 val;
const char *str;
struct resource res;

/* Read u32 property */
if (of_property_read_u32(np, "clock-frequency", &val)) {
	dev_err(dev, "missing clock-frequency\n");
	return -EINVAL;
}

/* Read u32 array */
u32 regs[2];
of_property_read_u32_array(np, "reg", regs, 2);

/* Read string property */
of_property_read_string(np, "label", &str);

/* Read boolean property */
if (of_property_read_bool(np, "dma-coherent"))
	dev_info(dev, "device is DMA coherent\n");

/* Read GPIO */
struct gpio_desc *gpio = devm_gpiod_get(dev, "reset", GPIOD_OUT_HIGH);

/* Read interrupt */
int irq = irq_of_parse_and_map(np, 0);
if (irq <= 0) {
	/* Or: platform_get_irq(dev, 0) */
	dev_err(dev, "no interrupt\n");
	return -EINVAL;
}
```

### Common DT properties

| Property | Type | Purpose |
|---|---|---|
| `reg` | `<addr> <size>` | MMIO base address and size |
| `reg-names` | `string-list` | Names for `reg` entries |
| `interrupts` | `<type> <number> <flags>` | Interrupt configuration |
| `interrupt-names` | `string-list` | Names for interrupt entries |
| `clocks` | `phandle` | Clock references |
| `clock-names` | `string-list` | Names for clock references |
| `status` | `"okay"` or `"disabled"` | Enable/disable device |
| `compatible` | `"vendor,device"` | Driver matching string |
| `#address-cells` | `<n>` | Address cell size |
| `#size-cells` | `<n>` | Size cell size |

## Step 4: ACPI

```c
static const struct acpi_device_id my_driver_acpi_match[] = {
	{ "MYDV0001", 0 },
	{ },
};
MODULE_DEVICE_TABLE(acpi, my_driver_acpi_match);

static struct platform_driver my_driver = {
	.driver = {
		.name = "my-driver",
		.acpi_match_table = my_driver_acpi_match,
	},
};
```

## Step 5: Regmap API

Unified register access for I2C, SPI, MMIO. Handles caching, locking, endianness.

```c
#include <linux/regmap.h>

/* MMIO regmap */
struct regmap_config my_regmap_cfg = {
	.reg_bits = 32,
	.val_bits = 32,
	.reg_stride = 4,
	.max_register = 0x100,
};

struct regmap *map = devm_regmap_init_mmio(dev, regs, &my_regmap_cfg);
if (IS_ERR(map))
	return PTR_ERR(map);

/* Read/write (handles locking, caching, endianness) */
u32 val;
regmap_read(map, REG_STATUS, &val);
regmap_write(map, REG_CONTROL, 0x01);

/* Set/clear bits (atomic if supported by bus) */
regmap_update_bits(map, REG_CONTROL, CTRL_ENABLE, CTRL_ENABLE);

/* Bulk read */
u32 buf[4];
regmap_bulk_read(map, REG_DATA_BASE, buf, 4);

/* I2C regmap */
struct regmap_config i2c_cfg = {
	.reg_bits = 8,
	.val_bits = 8,
};
struct regmap *map = devm_regmap_init_i2c(i2c_client, &i2c_cfg);

/* SPI regmap */
struct regmap_config spi_cfg = {
	.reg_bits = 16,
	.val_bits = 8,
	.read_flag_mask = 0x80,
};
struct regmap *map = devm_regmap_init_spi(spi_dev, &spi_cfg);
```

## Step 6: Interrupt handling

### Standard IRQ

```c
int irq = platform_get_irq(dev, 0);
if (irq < 0)
	return irq;

ret = devm_request_irq(dev, irq, my_irq_handler, IRQF_SHARED, "mydev", priv);
if (ret)
	return ret;

static irqreturn_t my_irq_handler(int irq, void *data)
{
	struct my_dev *priv = data;
	u32 status = readl(priv->regs + REG_IRQ_STATUS);

	if (!status)
		return IRQ_NONE;

	/* Acknowledge interrupt */
	writel(status, priv->regs + REG_IRQ_STATUS);

	/* Process */
	...

	return IRQ_HANDLED;
}
```

### Threaded IRQ

```c
ret = devm_request_threaded_irq(dev, irq,
                                 my_irq_handler_top,    /* hardirq (fast) */
                                 my_irq_handler_thread, /* threaded (can sleep) */
                                 IRQF_ONESHOT,
                                 "mydev", priv);

/* Top half: runs in IRQ context, quick check */
static irqreturn_t my_irq_handler_top(int irq, void *data)
{
	struct my_dev *priv = data;
	u32 status = readl(priv->regs + REG_IRQ_STATUS);
	if (!status)
		return IRQ_NONE;
	priv->irq_status = status;
	writel(status, priv->regs + REG_IRQ_STATUS);
	return IRQ_WAKE_THREAD;
}

/* Bottom half: runs in thread context, can sleep, do heavy work */
static irqreturn_t my_irq_handler_thread(int irq, void *data)
{
	struct my_dev *priv = data;
	u32 status = priv->irq_status;

	if (status & IRQ_RX_READY)
		process_rx(priv);
	if (status & IRQ_TX_DONE)
		complete(&priv->tx_done);

	return IRQ_HANDLED;
}
```

### Workqueue (deferred work from IRQ)

```c
#include <linux/workqueue.h>

struct my_dev {
	struct work_struct work;
	...
};

/* In probe */
INIT_WORK(&priv->work, my_work_func);

/* In IRQ handler */
static irqreturn_t my_irq_handler(int irq, void *data)
{
	struct my_dev *priv = data;
	schedule_work(&priv->work);
	return IRQ_HANDLED;
}

/* Deferred work function (process context, can sleep) */
static void my_work_func(struct work_struct *work)
{
	struct my_dev *priv = container_of(work, struct my_dev, work);
	/* Do heavy processing here */
}
```

## Anti-patterns

- **Direct pointer dereference on MMIO** — Never `*(volatile u32 *)regs`. Use `readl`/`writel`. Direct access bypasses barriers and breaks on ARM/POWER.
- **Missing wmb() between dependent writes** — If register B depends on register A, `writel(A)` then `wmb()` then `writel(B)`. Without the barrier, the CPU may reorder them.
- **IRQ handler that sleeps** — Standard IRQ handlers run in hard IRQ context. Use threaded IRQs (`request_threaded_irq`) if you need to do I2C/SPI access or allocate memory.
- **DT compatible string mismatch** — The `compatible` in the driver's `of_device_id` table must exactly match the DT node's `compatible`. One typo means the driver never binds.
- **Missing MODULE_DEVICE_TABLE** — Without it, the module won't auto-load when the device is present. Always add `MODULE_DEVICE_TABLE(of, ...)`.

## Step 7: Verify

- [ ] Register definitions match hardware manual
- [ ] `readl`/`writel` used consistently (not mixed with raw pointer dereference)
- [ ] `wmb()` between dependent register writes
- [ ] `rmb()` between dependent register reads
- [ ] IRQ handler returns `IRQ_NONE` when no interrupt pending
- [ ] Threaded IRQ used when handler needs to sleep (I2C/SPI access)
- [ ] All `request_region` have matching `release_region` (or use `devm_*`)
- [ ] Device tree compatible string matches `of_device_id` table
- [ ] Regmap `max_register` set correctly

See [DEVICETREE.md](DEVICETREE.md) for device tree reference.
