# Device Tree Reference

Linux device tree (DT/DTB) for describing hardware to the kernel.

## What device tree does

The device tree is a data structure that describes hardware that cannot be auto-discovered (unlike PCIe/USB). It tells the kernel:
- What devices exist
- Where their registers are (MMIO addresses)
- What interrupts they use
- What clocks, GPIOs, regulators they need
- How they're connected (bus topology)

Used primarily on ARM, ARM64, RISC-V, and some PowerPC/MIPS systems. x86 uses ACPI instead (though DT is used for some x86 embedded).

## DT source format (.dts)

```dts
/* soc.dtsi — included by board .dts files */
/ {
	compatible = "vendor,soc-name";
	#address-cells = <2>;
	#size-cells = <2>;

	soc {
		compatible = "simple-bus";
		#address-cells = <2>;
		#size-cells = <2>;
		ranges;

		my_device@f0000000 {
			compatible = "vendor,my-device";
			reg = <0x0 0xf0000000 0x0 0x1000>;   /* base=0xf0000000, size=0x1000 */
			interrupts = <GIC_SPI 42 IRQ_TYPE_LEVEL_HIGH>;
			clocks = <&clk_ctrl CLK_PERIPH>;
			clock-names = "periph";
			dma-coherent;
			status = "okay";
		};
	};
};
```

## Common properties

### reg — register addresses

```dts
/* #address-cells=2, #size-cells=2: 64-bit address + 64-bit size */
reg = <0x0 0xf0000000 0x0 0x1000>;
/* Reads as: address=0xf0000000, size=0x1000 */

/* Named registers */
reg = <0x0 0xf0000000 0x0 0x1000>,
      <0x0 0xf0010000 0x0 0x100>;
reg-names = "control", "dma";
```

Reading in driver:
```c
/* Single reg entry */
struct resource *res = platform_get_resource(dev, IORESOURCE_MEM, 0);
void __iomem *regs = devm_ioremap_resource(dev, res);

/* Named reg entry */
struct resource *res = platform_get_resource_byname(dev, IORESOURCE_MEM, "control");

/* Or use device API */
res = platform_get_resource(dev, IORESOURCE_MEM, 0);
dma_res = platform_get_resource(dev, IORESOURCE_MEM, 1);
```

### interrupts — hardware interrupts

```dts
/* SPI (Shared Peripheral Interrupt) 42, level-high */
interrupts = <GIC_SPI 42 IRQ_TYPE_LEVEL_HIGH>;

/* PPI (Private Peripheral Interrupt) 14, edge-rising */
interrupts = <GIC_PPI 14 IRQ_TYPE_EDGE_RISING>;

/* Named interrupts */
interrupts = <GIC_SPI 42 IRQ_TYPE_LEVEL_HIGH>,
             <GIC_SPI 43 IRQ_TYPE_EDGE_RISING>;
interrupt-names = "rx", "tx";
```

Reading in driver:
```c
int irq = platform_get_irq(dev, 0);           /* first interrupt */
int irq_tx = platform_get_irq_byname(dev, "tx");  /* named interrupt */
```

**Interrupt types**:
| Type | Value | Description |
|---|---|---|
| `IRQ_TYPE_NONE` | 0 | Not specified |
| `IRQ_TYPE_EDGE_RISING` | 1 | Rising edge |
| `IRQ_TYPE_EDGE_FALLING` | 2 | Falling edge |
| `IRQ_TYPE_EDGE_BOTH` | 3 | Both edges |
| `IRQ_TYPE_LEVEL_HIGH` | 4 | Active high |
| `IRQ_TYPE_LEVEL_LOW` | 8 | Active low |

### clocks

```dts
clocks = <&clk_ctrl CLK_UART>, <&clk_ctrl CLK_BUS>;
clock-names = "uart", "bus";
```

Reading:
```c
struct clk *clk_uart = devm_clk_get(dev, "uart");
struct clk *clk_bus = devm_clk_get(dev, "bus");
clk_prepare_enable(clk_uart);

/* Rate */
unsigned long rate = clk_get_rate(clk_uart);
```

### GPIOs

```dts
reset-gpios = <&gpio0 12 GPIO_ACTIVE_LOW>;
led-gpios = <&gpio1 3 GPIO_ACTIVE_HIGH>;
```

Reading:
```c
struct gpio_desc *reset = devm_gpiod_get(dev, "reset", GPIOD_OUT_HIGH);
gpiod_set_value(reset, 0);  /* assert reset */
msleep(10);
gpiod_set_value(reset, 1);  /* deassert reset */
```

### phandle references

```dts
/* Define a node */
clk_ctrl: clock-controller@f0001000 {
	compatible = "vendor,clock-controller";
	reg = <0x0 0xf0001000 0x0 0x1000>;
	#clock-cells = <1>;  /* consumers need 1 cell to specify clock */
};

/* Reference it */
my_device@f0000000 {
	clocks = <&clk_ctrl 5>;  /* 5 = clock ID within clk_ctrl */
};
```

## DTB (compiled device tree)

```bash
# Compile .dts → .dtb
dtc -I dts -O dtb -o board.dtb board.dts

# Decompile .dtb → .dts
dtc -I dtb -O dts -o board.dts board.dtb

# Validate
dtc -I dtb -O dtb -o /dev/null board.dtb

# Apply overlay
fdtoverlay -i base.dtb -o combined.dtb overlay.dtbo
```

## DT overlays

```dts
/* overlay.dts — adds a device to a running system */
/dts-v1/;
/plugin/;

&i2c0 {
	status = "okay";

	my_sensor@48 {
		compatible = "vendor,my-sensor";
		reg = <0x48>;
	};
};
```

```bash
# Compile overlay
dtc -@ -I dts -O dtb -o overlay.dtbo overlay.dts

# Apply at runtime (if CONFIG_OF_OVERLAY=y)
mkdir -p /sys/kernel/config/device-tree/overlays/my-overlay
cp overlay.dtbo /sys/kernel/config/device-tree/overlays/my-overlay/dtbo
```

## DT bindings documentation

Kernel docs for DT bindings: `Documentation/devicetree/bindings/`

```bash
# Search for a binding
grep -r "vendor,my-device" Documentation/devicetree/bindings/

# Check binding YAML (newer kernels)
ls Documentation/devicetree/bindings/*/vendor,my-device.yaml
```

## DT from running system

```bash
# Full DT from running system
dtc -I fs -O dts /sys/firmware/devicetree/base

# Specific device node
ls /sys/firmware/devicetree/base/soc/my-device@f0000000/
cat /sys/firmware/devicetree/base/soc/my-device@f0000000/compatible

# Check matched devices
cat /sys/bus/platform/devices/*/of_node/compatible
```
