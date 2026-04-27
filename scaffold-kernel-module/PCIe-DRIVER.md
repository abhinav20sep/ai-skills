# PCIe Driver Template Reference

Full scaffold for a PCIe device driver with BAR mapping, MSI-X interrupts, and DMA support.

## File structure

```
<name>/
├── <name>.c        # Driver implementation
├── <name>.h        # Device struct, register defs
└── Makefile         # Kbuild
```

## `<name>.h`

```c
#ifndef <NAME>_H
#define <NAME>_H

#include <linux/pci.h>
#include <linux/io.h>

#define <NAME>_DRV_NAME	"<name>"
#define <NAME>_BAR_REGS	0	/* BAR for register MMIO */
#define <NAME>_BAR_MEM		2	/* BAR for device memory (if needed) */

/* Hardware register offsets (example) */
#define <NAME>_REG_CTRL		0x00
#define <NAME>_REG_STATUS	0x04
#define <NAME>_REG_IRQ_MASK	0x08
#define <NAME>_REG_IRQ_STATUS	0x0c

/* Device private data */
struct <name>_dev {
	struct pci_dev		*pdev;
	void __iomem		*regs;		/* ioremap'd BAR */
	int			irq;
	/* Add DMA buffers, locks, workqueues, cdev, etc. */
};

#endif /* <NAME>_H */
```

## `<name>.c`

```c
#include <linux/module.h>
#include <linux/pci.h>
#include <linux/interrupt.h>
#include <linux/dma-mapping.h>
#include "<name>.h"

static const struct pci_device_id <name>_id_table[] = {
	{ PCI_DEVICE(VENDOR_ID, DEVICE_ID) },
	{ 0, },
};
MODULE_DEVICE_TABLE(pci, <name>_id_table);

static irqreturn_t <name>_irq_handler(int irq, void *data)
{
	struct <name>_dev *priv = data;
	u32 status;

	status = ioread32(priv->regs + <NAME>_REG_IRQ_STATUS);
	if (!status)
		return IRQ_NONE;

	/* Handle interrupt */
	iowrite32(status, priv->regs + <NAME>_REG_IRQ_STATUS);
	return IRQ_HANDLED;
}

static int <name>_probe(struct pci_dev *pdev, const struct pci_device_id *id)
{
	struct <name>_dev *priv;
	int err;

	/* Enable PCI device */
	err = pci_enable_device(pdev);
	if (err)
		return err;

	/* Request BAR regions */
	err = pci_request_regions(pdev, <NAME>_DRV_NAME);
	if (err)
		goto err_disable;

	/* Map register BAR */
	priv->regs = pci_iomap(pdev, <NAME>_BAR_REGS, 0);
	if (!priv->regs) {
		err = -EIO;
		goto err_regions;
	}

	/* Set up DMA mask (64-bit preferred, fall back to 32-bit) */
	err = dma_set_mask_and_coherent(&pdev->dev, DMA_BIT_MASK(64));
	if (err) {
		err = dma_set_mask_and_coherent(&pdev->dev, DMA_BIT_MASK(32));
		if (err)
			goto err_iounmap;
	}

	/* Allocate MSI-X vectors (1 vector = 1 interrupt) */
	err = pci_alloc_irq_vectors(pdev, 1, 1, PCI_IRQ_MSI | PCI_IRQ_LEGACY);
	if (err < 0)
		goto err_iounmap;

	priv = devm_kzalloc(&pdev->dev, sizeof(*priv), GFP_KERNEL);
	if (!priv) {
		err = -ENOMEM;
		goto err_irq_vectors;
	}

	priv->pdev = pdev;
	priv->irq = pci_irq_vector(pdev, 0);

	/* Register interrupt handler */
	err = request_irq(priv->irq, <name>_irq_handler, 0, <NAME>_DRV_NAME, priv);
	if (err)
		goto err_irq_vectors;

	pci_set_drvdata(pdev, priv);

	/* TODO: register char device, sysfs, crypto API, etc. */

	dev_info(&pdev->dev, "%s: probe succeeded\n", <NAME>_DRV_NAME);
	return 0;

err_irq_vectors:
	pci_free_irq_vectors(pdev);
err_iounmap:
	pci_iounmap(pdev, priv->regs);
err_regions:
	pci_release_regions(pdev);
err_disable:
	pci_disable_device(pdev);
	return err;
}

static void <name>_remove(struct pci_dev *pdev)
{
	struct <name>_dev *priv = pci_get_drvdata(pdev);

	/* TODO: unregister char device, sysfs, etc. */

	free_irq(priv->irq, priv);
	pci_free_irq_vectors(pdev);
	pci_iounmap(pdev, priv->regs);
	pci_release_regions(pdev);
	pci_disable_device(pdev);

	dev_info(&pdev->dev, "%s: removed\n", <NAME>_DRV_NAME);
}

static struct pci_driver <name>_driver = {
	.name		= <NAME>_DRV_NAME,
	.id_table	= <name>_id_table,
	.probe		= <name>_probe,
	.remove		= <name>_remove,
};
module_pci_driver(<name>_driver);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Your Name");
MODULE_DESCRIPTION("<name> PCIe driver");
```

## `Makefile`

```makefile
obj-m += <name>.o
<name>-objs := <name>_main.o <name>_hw.o  # if splitting files

# For single file:
# obj-m += <name>.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	$(MAKE) -C $(KDIR) M=$(PWD) modules

clean:
	$(MAKE) -C $(KDIR) M=$(PWD) clean
```

## Probe ordering and error handling

The probe function must clean up in reverse order on failure. This is the standard pattern:

```
pci_enable_device
  pci_request_regions
    pci_iomap
      dma_set_mask_and_coherent
        pci_alloc_irq_vectors
          devm_kzalloc
            request_irq
              pci_set_drvdata
              [register devices]
```

On failure at any level, unwind everything below it. Use `devm_*` variants where possible to simplify cleanup.

## Common additions

- **DMA buffers**: `dma_alloc_coherent()` / `dma_free_coherent()` for consistent DMA mappings
- **Character device**: `cdev_init`, `cdev_add`, `class_create`, `device_create` for /dev entry
- **sysfs attributes**: `DEVICE_ATTR_RW` / `DEVICE_ATTR_RO` for exposing registers/status
- **Workqueues**: `INIT_WORK` / `queue_work` for deferred interrupt processing
- **Locking**: `spinlock_t` for IRQ-context, `mutex_t` for process-context
- **Power management**: `pm_message_t` in pci_driver for suspend/resume

## Vendor/device ID lookup

Replace `VENDOR_ID` and `DEVICE_ID` with actual PCI IDs. Use `lspci -nn` to find them:

```bash
lspci -nn | grep <device>
# Output: 01:00.0 Encryption controller [1080]: Device 1234:5678 (rev 01)
# Vendor ID = 1234, Device ID = 5678
```
