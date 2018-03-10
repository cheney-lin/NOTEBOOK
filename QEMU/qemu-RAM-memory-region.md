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
    struct rcu_head rcu;//读RAMBlock需要拿rcu读锁
    struct MemoryRegion *mr;//对应的mr
    uint8_t *host;//hva
    ram_addr_t offset;//在所有RAMBlock space 中的偏移
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
## Memory region
```c
struct MemoryRegion {                                                                                                                                                  
    Object parent_obj;                                                                                                                                                 
                                                                                                                                                                       
    /* All fields are private - violators will be prosecuted */                                                                                                        
                                                                                                                                                                       
    /* The following fields should fit in a cache line */                                                                                                              
    bool romd_mode;                                                                                                                                                    
    bool ram;                                                                                                                                                          
    bool subpage;                                                                                                                                                      
    bool readonly; /* For RAM regions */ //比如pc.rom                                                                                                                              
    bool rom_device;                                                                                                                                                   
    bool flush_coalesced_mmio;                                                                                                                                         
    bool global_locking;                                                                                                                                               
    uint8_t dirty_log_mask;                                                                                                                                            
    bool is_iommu;                                                                                                                                                     
    RAMBlock *ram_block;      //如果该mr申请了内存就指向它的RAMBlock，否则为NULL                                                                                                                                   
    Object *owner;  属于哪个设备                                                                                                                                                 
                                                                                                                                                                       
    const MemoryRegionOps *ops;        //对mr操作的callback函数，比如读写                                                                                                                             
    void *opaque;                                                                                                                                                      
    MemoryRegion *container;   //父mr                                                                                                                                        
    Int128 size;         //mr的大小                                                                                                                  
    hwaddr addr;      //相对父mr的偏移，起始GPA=base+addr 可以去源码中搜索一下如何初始化的就明白了                                                                                                        
    void (*destructor)(MemoryRegion *mr);                                                                                                                              
    uint64_t align;                                                                                                                                                    
    bool terminates;                                                                                                                                                   
    bool ram_device;                                                                                                                                                   
    bool enabled;                                                                                                                                                      
    bool warning_printed; /* For reservations */                                                                                                                       
    uint8_t vga_logging_count;                                                                                                                                         
    MemoryRegion *alias;     // 如果本mr是个alias mr，这个字段指向真实的mr，否则为NULL                                                                                                            
    hwaddr alias_offset;        //如果本mr是个alias mr，这个字段表示在真实的mr中的偏移                                                                                                             
    int32_t priority;      //优先级，属于同一个mr的subregions中高优先级的mr会被优先渲染                                                                                                                                              
    QTAILQ_HEAD(subregions, MemoryRegion) subregions;                                                                                                                  
    QTAILQ_ENTRY(MemoryRegion) subregions_link;                                                                                                                        
    QTAILQ_HEAD(coalesced_ranges, CoalescedMemoryRange) coalesced;                                                                                                     
    const char *name;                                                                                                                                                  
    unsigned ioeventfd_nb;                                                                                                                                             
    MemoryRegionIoeventfd *ioeventfds;                                                                                                                                 
};                                                                                                                                                                     
```






## RAM Memory Region API
QEMU具有多种类型的mr（比如RAM,ROM,MMIO,container,alias），官方文档给出了很详尽的解释：`docs/memory.txt`
RAM类型的mr，物理机世界的内存条相似，可由guest OS直接访问而不会退到跟模式。
简单介绍下RAM mr初始化相关的API：
**memory_region_init_ram**:
最终分调用mmap分配匿名页类型的内存
**memory_region_init_ram_ptr**:
多传入了一个用户已经分配了内存的指针，不再去申请内存
**memory_region_init_from_file**:
利用fd映射虚拟内存

## refernce
[QEMU Internals: How guest phtsical RAM works —— Hajnoczi
](http://blog.vmsplice.net/2016/01/qemu-internals-how-guest-physical-ram.html)

[QEMU Devices and RAM Memory Regions](http://nairobi-embedded.org/050_devices_and_ram_memory_regions.html)