# DMA Mapping Reference

The Linux DMA mapping API for device drivers. Covers streaming and consistent DMA mappings.

## Core concepts

| Concept | Description |
|---|---|
| **Streaming DMA** | One-shot or periodic transfers. Buffer ownership transfers between CPU and device. Must sync. |
| **Consistent (coherent) DMA** | Long-lived buffer. Both CPU and device can access simultaneously. No sync needed. |
| **DMA direction** | `DMA_TO_DEVICE` (CPU→device), `DMA_FROM_DEVICE` (device→CPU), `DMA_BIDIRECTIONAL` |
| **DMA address** | Physical address the device sees (may differ from CPU physical on IOMMU systems) |

## Setup

```c
#include <linux/dma-mapping.h>

/* In probe(): set DMA mask */
err = dma_set_mask_and_coherent(&pdev->dev, DMA_BIT_MASK(64));
if (err) {
	err = dma_set_mask_and_coherent(&pdev->dev, DMA_BIT_MASK(32));
	if (err) {
		dev_err(&pdev->dev, "DMA configuration failed\n");
		return err;
	}
}
```

## Streaming DMA

### Single buffer

```c
/* Map a buffer for DMA (CPU→device) */
dma_addr_t dma_addr = dma_map_single(&pdev->dev, buf, len, DMA_TO_DEVICE);
if (dma_mapping_error(&pdev->dev, dma_addr)) {
	dev_err(&pdev->dev, "DMA mapping failed\n");
	return -EIO;
}

/* Program device with dma_addr */
iowrite32(lower_32_bits(dma_addr), priv->regs + REG_DMA_ADDR_LOW);
iowrite32(upper_32_bits(dma_addr), priv->regs + REG_DMA_ADDR_HIGH);
iowrite32(len, priv->regs + REG_DMA_LEN);

/* Unmap when device is done */
dma_unmap_single(&pdev->dev, dma_addr, len, DMA_TO_DEVICE);
```

### Sync for CPU access

```c
/* After device writes to buffer, sync for CPU read */
dma_sync_single_for_cpu(&pdev->dev, dma_addr, len, DMA_FROM_DEVICE);

/* Read the data */
memcpy(local_buf, buf, len);

/* Before device reads, sync for device */
dma_sync_single_for_device(&pdev->dev, dma_addr, len, DMA_TO_DEVICE);
```

### Scatter-gather list (SG)

```c
struct scatterlist sg;
int nents;

/* Initialize SG list */
sg_init_table(&sg, 1);
sg_set_page(&sg, page, len, offset);

/* Map for DMA */
nents = dma_map_sg(&pdev->dev, &sg, 1, DMA_TO_DEVICE);
if (nents == 0) {
	dev_err(&pdev->dev, "SG mapping failed\n");
	return -EIO;
}

/* Iterate mapped entries */
for_each_sg(&sg, s, nents, i) {
	dma_addr_t addr = sg_dma_address(s);
	unsigned int length = sg_dma_len(s);
	/* Program device with addr and length */
}

/* Unmap */
dma_unmap_sg(&pdev->dev, &sg, 1, DMA_TO_DEVICE);
```

## Consistent (coherent) DMA

Long-lived buffer that both CPU and device can access without sync.

```c
/* Allocate consistent DMA buffer */
void *buf = dma_alloc_coherent(&pdev->dev, 4096, &dma_addr, GFP_KERNEL);
if (!buf) {
	dev_err(&pdev->dev, "DMA alloc failed\n");
	return -ENOMEM;
}

/* Use buf from CPU side */
memset(buf, 0, 4096);
*(u32 *)buf = 0xDEADBEEF;

/* Program device with dma_addr */
iowrite32(dma_addr, priv->regs + REG_DMA_ADDR);

/* Free when done */
dma_free_coherent(&pdev->dev, 4096, buf, dma_addr);
```

**When to use consistent vs streaming**:

| Criterion | Streaming | Consistent |
|---|---|---|
| Sync required | Yes (map/unmap or sync) | No |
| Performance | Better for bulk transfers | Better for control/status |
| Use case | Data buffers, network packets | Ring buffers, descriptor rings, shared state |
| Cache behavior | Can use write-combining | Uncached or write-through |

## DMA pool — small consistent allocations

```c
/* Create a pool for small DMA objects (e.g., descriptors) */
struct dma_pool *pool = dma_pool_create("my_pool", &pdev->dev,
                                         64,    /* object size */
                                         64,    /* alignment */
                                         0);    /* boundary (0 = page) */

/* Allocate from pool */
dma_addr_t dma_addr;
void *desc = dma_pool_alloc(pool, GFP_KERNEL, &dma_addr);
if (!desc)
	return -ENOMEM;

/* Use desc / dma_addr */
memset(desc, 0, 64);

/* Free to pool */
dma_pool_free(pool, desc, dma_addr);

/* Destroy pool */
dma_pool_destroy(pool);
```

## DMA pool vs kmalloc for descriptors

Use `dma_pool_create` when you need many small DMA-accessible objects (descriptors, status blocks). It's more efficient than `dma_alloc_coherent` for many small allocations because it batches them into single pages.

## IOMMU

The IOMMU (I/O Memory Management Unit) translates device DMA addresses to physical addresses. Most modern systems have an IOMMU (Intel VT-d, AMD-Vi, ARM SMMU).

```c
/* Check if IOMMU is active */
if (iommu_present(&pci_bus_type)) {
	dev_info(&pdev->dev, "IOMMU active — DMA addresses are IOVA\n");
}

/* DMA mask still works with IOMMU — the API abstracts it */
dma_set_mask_and_coherent(&pdev->dev, DMA_BIT_MASK(64));

/* IOMMU groups and isolation */
struct iommu_group *group = iommu_group_get(&pdev->dev);
if (group) {
	int id = iommu_group_id(group);
	dev_info(&pdev->dev, "IOMMU group %d\n", id);
	iommu_group_put(group);
}
```

With an IOMMU, `dma_addr` is an IOVA (I/O Virtual Address), not a physical address. The IOMMU hardware translates it. This provides:
- **DMA isolation**: devices can only access their own mapped memory
- **DMA remapping**: non-contiguous physical pages appear contiguous to device
- **Bounce buffering**: if device can't address high memory, IOMMU can remap

## Debugging DMA issues

```bash
# Check IOMMU status
dmesg | grep -i iommu
cat /sys/kernel/iommu_groups/*/info 2>/dev/null

# Check DMA debug (CONFIG_DMA_API_DEBUG)
echo 1 > /sys/kernel/debug/dma-api/error_dump
cat /sys/kernel/debug/dma-api/tracked  # all active DMA mappings

# Check DMA zone usage
cat /proc/buddyinfo
cat /proc/zoneinfo | grep -A5 DMA
```

## Common DMA bugs

| Bug | Symptom | Fix |
|---|---|---|
| Missing `dma_mapping_error` check | Silent data corruption | Always check return value |
| Wrong direction | Device reads stale data | Use `DMA_TO_DEVICE` for CPU→device |
| Missing sync | CPU reads stale data after device write | `dma_sync_single_for_cpu` |
| Double unmap | Oops, IOMMU fault | Track mapping state, unmap once |
| DMA after free | Data corruption, IOMMU fault | Unmap before freeing buffer |
| Missing DMA mask | High memory not reachable | Set mask in probe, check return |
