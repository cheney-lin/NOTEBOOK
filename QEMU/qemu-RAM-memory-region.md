## Memory backend
### 1. anonymous mmapped memory
使用mmap申请的匿名页
### 2. file-backed mmapped memory
ivshmem设备的共享内存
hugupage fs

## RAM blocks
每个RAM类型的mr都会有一个RAM


``` c
struct MemoryRegion {
    Object parent_obj;

    /* All fields are private - violators will be prosecuted */

    /* The following fields should fit in a cache line */
    bool romd_mode;
    bool ram;
    bool subpage;
    bool readonly; /* For RAM regions */
    bool rom_device;
    bool flush_coalesced_mmio;
    bool global_locking;
    uint8_t dirty_log_mask;
    bool is_iommu;
    RAMBlock *ram_block;
    Object *owner;

```



## RAM Memory Region API
QEMU具有多种类型的mr（比如RAM,ROM,MMIO,container,alias），官方文档给出了很详尽的解释：`docs/memory.txt`
RAM类型的mr，物理机世界的内存条相似，可由guest OS直接访问而不会退到跟模式。
简单介绍下RAM mr初始化相关的API：
memory_region_init_ram:

memory_region_init_ram_ptr:
memory_region_init_from_file:

## refernce
[QEMU Internals: How guest phtsical RAM works —— Hajnoczi
](http://blog.vmsplice.net/2016/01/qemu-internals-how-guest-physical-ram.html)

[QEMU Devices and RAM Memory Regions](http://nairobi-embedded.org/050_devices_and_ram_memory_regions.html)