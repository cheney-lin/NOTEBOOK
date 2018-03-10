## Memory backend
### 1. anonymous mmapped memory
使用mmap申请的匿名页
### 2. file-backed mmapped memory
ivshmem设备的共享内存
hugupage fs

## RAM blocks
每个RAM类型的mr都会有一个RAMBlock *ram_block，



``` c
struct RAMBlock {
    struct rcu_head rcu;
    struct MemoryRegion *mr;
    uint8_t *host;
    ram_addr_t offset;
    ram_addr_t used_length;
    ram_addr_t max_length;
    void (*resized)(const char*, uint64_t length, void *host);
    uint32_t flags;
    /* Protected by iothread lock.  */
    char idstr[256];
    /* RCU-enabled, writes protected by the ramlist lock */
    QLIST_ENTRY(RAMBlock) next;
    QLIST_HEAD(, RAMBlockNotifier) ramblock_notifiers;
    int fd;
    size_t page_size;
    /* dirty bitmap used during migration */
    unsigned long *bmap;
    /* bitmap of pages that haven't been sent even once
     * only maintained and used in postcopy at the moment
     * where it's used to send the dirtymap at the start
     * of the postcopy phase
     */
    unsigned long *unsentmap;
    /* bitmap of already received pages in postcopy */
    unsigned long *receivedmap;
};

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