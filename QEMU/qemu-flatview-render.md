### QEMU内存管理之生成FlatView内存拓扑模型过程分析
本文基于六六哥的博客分析，修正文中的一些错误，[原文链接](http://blog.csdn.net/leoufung/article/details/48781209)

关于MemoryRegion 可以看QEMU 源码树中的文档memory.txt
MemoryRegion 和 Flatview是QEMU用来组织内存布局的两种形式，他们的关系可以参考
[《MemoryRegion模型原理，以及同FlatView模型的关系》](http://blog.csdn.net/leoufung/article/details/48781205)
每当mr发生变化，QEMU都要通知KVM更新GPA的mapping，首先在QEMU侧需要把树形的内存布局转换成平坦的地址空间，这样KVM侧的处理可以简单些，flatview展开的入口函数为：
```c
/*
* 将MR所管理的内存展开成FlatView后返回
*/
static FlatView *generate_memory_topology(MemoryRegion *mr)
{
    FlatView *view;

     /*新分配一个FlatView并初始化*/
    view = g_new(FlatView, 1);
    flatview_init(view);

    if (mr) {
          /*
          * 将mr所管理的内存进行平坦展开为FlatView,通过view返回
          * addrrange_make(int128_zero(), int128_2_64()) --> 指定一个GUEST内存空间，起始GPA等于0，大小等于2^64，可以想象成GUEST的整个物理地址空间         
          */
        render_memory_region(view, mr, int128_zero(),
                             addrrange_make(int128_zero(), int128_2_64()), false);
    }

     /*简化FlatView，将View中的FlatRange能合并的都合并*/
    flatview_simplify(view);

    return view;
}
```
核心函数为render_memory_region，这个函数之所以很费解是因为各种变量没有注释而让人捉摸不透，网上有几篇关于QEMU内存管理的博客，可惜或多或少存在一些错误。六六哥的博客我觉得是写的最好的，可惜有一些图没了，我会尽量补上，这样会更容易理解一些。

首先对本文中会提及到的变量说一下我的理解，有的变量的含义我不是很确定就没写
```c
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
    RAMBlock *ram_block;      //如果该mr申请了内存就指向它的RAMBlock，否则为NULL                                                                                                                                   
    Object *owner;                                                                                                                                                   
                                                                                                                                                                       
    const MemoryRegionOps *ops;        //对mr操作的callback函数，比如读写                                                                                                                             
    void *opaque;                                                                                                                                                      
    MemoryRegion *container;                                                                                                                                           
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
    int32_t priority;                                                                                                                                                  
    QTAILQ_HEAD(subregions, MemoryRegion) subregions;                                                                                                                  
    QTAILQ_ENTRY(MemoryRegion) subregions_link;                                                                                                                        
    QTAILQ_HEAD(coalesced_ranges, CoalescedMemoryRange) coalesced;                                                                                                     
    const char *name;                                                                                                                                                  
    unsigned ioeventfd_nb;                                                                                                                                             
    MemoryRegionIoeventfd *ioeventfds;                                                                                                                                 
};                                                                                                                                                                     
```



```c
/* Render a memory region into the global view.  Ranges in @view obscure
* ranges in @mr.
*/
/*
* render: 致使；提出；实施；着色；以…回报
*/
/*
* 将MR中所管理的内存，在clip指定的地址空间，逐个形成FlatRange后，将所有的FlatRange加入FlatView中
*
* @view:           待形成的View
* @mr:              待展平的Mr
* @clip:            待展开的内存将被展开在clip所在的区域内，第一次render时clip代表了整个物理地址空间
* @base:          可以理解为父mr的起始GPA
* @readonly:     读写属性
*/
static void render_memory_region(FlatView *view,
                                 MemoryRegion *mr,
                                 Int128 base,
                                 AddrRange clip,
                                 bool readonly)
{
    MemoryRegion *subregion;
    unsigned i;
    hwaddr offset_in_region;     /*在region对应的真实物理内存的偏移量*/
    Int128 remain;      /*待展开内存的长度*/
    Int128 now;      /*本次展开的长度或跳过的长度*/
    FlatRange fr;
    AddrRange tmp;

    if (!mr->enabled) {
        return;
    }

     /*base改为MR的起始地址
     addr实际上是subregion在父mr的偏移
     */
    int128_addto(&base, int128_make64(mr->addr));
    readonly |= mr->readonly;

     /*取得mr所表示的物理地址范围tmp*/
    tmp = addrrange_make(base, mr->size);

     /*更新clip为MR所代表的地址空间*/
    if (!addrrange_intersects(tmp, clip)) {
        return;
    }

    clip = addrrange_intersection(tmp, clip);

     /*如果是alias类型的MR，首先对原始MR进行FlatView展开*/
  //根据这几行代码，我们可以得知alias的GPA = origin mr的addr + alias mr的alias_offset
    if (mr->alias) {
          /*将base指向alias源MR的起始地址位置*/
        int128_subfrom(&base, int128_make64(mr->alias->addr));
        int128_subfrom(&base, int128_make64(mr->alias_offset));
        render_memory_region(view, mr->alias, base, clip, readonly);
        return;//不再展开自己了，见注1. 
    }

    /* Render subregions in priority order. */
     /* 对所有子MR递归进行FlatView展开 */
    QTAILQ_FOREACH(subregion, &mr->subregions, subregions_link) {
        render_memory_region(view, subregion, base, clip, readonly);
    }

    if (!mr->terminates) {
        return;
    }

     /*
     * 运行都这里说明MemoryRegion的子MR都已经展开了
     */
     /*
     * 更新offset_in_region，offset_in_region是mr的gpa与clip的偏移量，由于我们从clip.start开始render，因此将作为后面fr的offset_in_region，以后用来计算本FR对应MR的物理内存的HVA
     */
    offset_in_region = int128_get64(int128_sub(clip.start, base));
     /*
     * 准备展开MR为FlatRange,所有的FlatRange组成FlatView
     * clip为待展开MR
     * 更新base为clip的起始，remain为待展开的长度
     */
    base = clip.start;
    remain = clip.size;

    fr.mr = mr;
    fr.dirty_log_mask = mr->dirty_log_mask;
    fr.romd_mode = mr->romd_mode;
    fr.readonly = readonly;

    /* Render the region itself into any gaps left by the current view. */
     /* 开始展开 */
    for (i = 0; i < view->nr && int128_nz(remain); ++i) {
          /*跳过FlatView中在clip前面的FR*/
        if (int128_ge(base, addrrange_end(view->ranges[i].addr))) {
            continue;
        }

          /*
          * 处理clip起始小于当前range起始的情况
          * 展开
          */
        if (int128_lt(base, view->ranges[i].addr.start)) {
               /*计算填空部分大小*/
            now = int128_min(remain,
                             int128_sub(view->ranges[i].addr.start, base));
               /*填充新的Fr信息*/
            fr.offset_in_region = offset_in_region;
            fr.addr = addrrange_make(base, now);
               /*将新的Fr信息填充到插入到FlatView的当前位置，以前该位置往后的FlatRange都向后顺移了一位*/
            flatview_insert(view, i, &fr);
               /*i++执行原来插入位置FlatRange*/
            ++i;
              
            int128_addto(&base, now);
            offset_in_region += int128_get64(now);
            int128_subfrom(&remain, now);
        }

          /*跳过重叠的部分*/
          /*计算重叠部分的长度，现在now是已经被其他fr占据的区间*/
        now = int128_sub(int128_min(int128_add(base, remain),
                                    addrrange_end(view->ranges[i].addr)),
                         base);
          /*跳过重叠部分*/
        int128_addto(&base, now);
        offset_in_region += int128_get64(now);
        int128_subfrom(&remain, now);
    }

     /*遍历完所有现有的FlatRange后，最后发现还有未展开的内存，这里处理其展开*/    
    if (int128_nz(remain)) {
          /*填入FR的信息*/
        fr.offset_in_region = offset_in_region;
        fr.addr = addrrange_make(base, remain);
          /*插入该FR*/
        flatview_insert(view, i, &fr);
    }
}
```

注1. 原文说如果是alias还要展开自己，是不对的。事实上，alias mr和它指向的mr的“gpa”(base+addr)并不一定一致，如果都展开，映射就重复了。下面是info mtree 打印出来的虚拟机address-space及 memory-region(被alias mr指向的mr)

```
address-space: memory
  0000000000000000-ffffffffffffffff (prio 0, RW): system
    0000000000000000-00000000bfffffff (prio 0, RW): alias ram-below-4g @pc.ram 0000000000000000-00000000bfffffff
    0000000000000000-ffffffffffffffff (prio -1, RW): pci
      00000000000a0000-00000000000bffff (prio 1, RW): cirrus-lowmem-container
        00000000000a0000-00000000000a7fff (prio 1, RW): alias vga.bank0 @vga.vram 0000000000000000-0000000000007fff
        00000000000a0000-00000000000bffff (prio 0, RW): cirrus-low-memory
        00000000000a8000-00000000000affff (prio 1, RW): alias vga.bank1 @vga.vram 0000000000008000-000000000000ffff
      00000000000c0000-00000000000dffff (prio 1, RW): pc.rom
      00000000000e0000-00000000000fffff (prio 1, R-): alias isa-bios @pc.bios 0000000000020000-000000000003ffff
      00000000fc000000-00000000fdffffff (prio 1, RW): cirrus-pci-bar0
        00000000fc000000-00000000fc7fffff (prio 1, RW): vga.vram
        00000000fc000000-00000000fc7fffff (prio 0, RW): cirrus-linear-io
        00000000fd000000-00000000fd3fffff (prio 0, RW): cirrus-bitblt-mmio
      00000000febf0000-00000000febf0fff (prio 1, RW): cirrus-mmio
      00000000febf1000-00000000febf1fff (prio 1, RW): virtio-scsi-pci-msix
        00000000febf1000-00000000febf103f (prio 0, RW): msix-table
        00000000febf1800-00000000febf1807 (prio 0, RW): msix-pba
      00000000febf2000-00000000febf2fff (prio 1, RW): virtio-serial-pci-msix
        00000000febf2000-00000000febf201f (prio 0, RW): msix-table
        00000000febf2800-00000000febf2807 (prio 0, RW): msix-pba
      00000000fffc0000-00000000ffffffff (prio 0, R-): pc.bios
    00000000000a0000-00000000000bffff (prio 1, RW): alias smram-region @pci 00000000000a0000-00000000000bffff
    00000000000c0000-00000000000c3fff (prio 1, RW): alias pam-ram @pc.ram 00000000000c0000-00000000000c3fff [disabled]
    00000000000c0000-00000000000c3fff (prio 1, RW): alias pam-pci @pc.ram 00000000000c0000-00000000000c3fff [disabled]
    00000000000c0000-00000000000c3fff (prio 1, R-): alias pam-rom @pc.ram 00000000000c0000-00000000000c3fff
    00000000000c0000-00000000000c3fff (prio 1, RW): alias pam-pci @pci 00000000000c0000-00000000000c3fff [disabled]
    00000000000c4000-00000000000c7fff (prio 1, RW): alias pam-ram @pc.ram 00000000000c4000-00000000000c7fff [disabled]
    00000000000c4000-00000000000c7fff (prio 1, RW): alias pam-pci @pc.ram 00000000000c4000-00000000000c7fff [disabled]
    00000000000c4000-00000000000c7fff (prio 1, R-): alias pam-rom @pc.ram 00000000000c4000-00000000000c7fff
    00000000000c4000-00000000000c7fff (prio 1, RW): alias pam-pci @pci 00000000000c4000-00000000000c7fff [disabled]
    00000000000c8000-00000000000cbfff (prio 1, RW): alias pam-ram @pc.ram 00000000000c8000-00000000000cbfff [disabled]
    00000000000c8000-00000000000cbfff (prio 1, RW): alias pam-pci @pc.ram 00000000000c8000-00000000000cbfff [disabled]
    00000000000c8000-00000000000cbfff (prio 1, R-): alias pam-rom @pc.ram 00000000000c8000-00000000000cbfff
    00000000000c8000-00000000000cbfff (prio 1, RW): alias pam-pci @pci 00000000000c8000-00000000000cbfff [disabled]
    00000000000c9000-00000000000cbfff (prio 1000, RW): alias kvmvapic-rom @pc.ram 00000000000c9000-00000000000cbfff
    00000000000cc000-00000000000cffff (prio 1, RW): alias pam-ram @pc.ram 00000000000cc000-00000000000cffff [disabled]
    00000000000cc000-00000000000cffff (prio 1, RW): alias pam-pci @pc.ram 00000000000cc000-00000000000cffff [disabled]
    00000000000cc000-00000000000cffff (prio 1, R-): alias pam-rom @pc.ram 00000000000cc000-00000000000cffff
    00000000000cc000-00000000000cffff (prio 1, RW): alias pam-pci @pci 00000000000cc000-00000000000cffff [disabled]
    00000000000d0000-00000000000d3fff (prio 1, RW): alias pam-ram @pc.ram 00000000000d0000-00000000000d3fff [disabled]
    00000000000d0000-00000000000d3fff (prio 1, RW): alias pam-pci @pc.ram 00000000000d0000-00000000000d3fff [disabled]
    00000000000d0000-00000000000d3fff (prio 1, R-): alias pam-rom @pc.ram 00000000000d0000-00000000000d3fff
    00000000000d0000-00000000000d3fff (prio 1, RW): alias pam-pci @pci 00000000000d0000-00000000000d3fff [disabled]
    00000000000d4000-00000000000d7fff (prio 1, RW): alias pam-ram @pc.ram 00000000000d4000-00000000000d7fff [disabled]
    00000000000d4000-00000000000d7fff (prio 1, RW): alias pam-pci @pc.ram 00000000000d4000-00000000000d7fff [disabled]
    00000000000d4000-00000000000d7fff (prio 1, R-): alias pam-rom @pc.ram 00000000000d4000-00000000000d7fff
    00000000000d4000-00000000000d7fff (prio 1, RW): alias pam-pci @pci 00000000000d4000-00000000000d7fff [disabled]
    00000000000d8000-00000000000dbfff (prio 1, RW): alias pam-ram @pc.ram 00000000000d8000-00000000000dbfff [disabled]
    00000000000d8000-00000000000dbfff (prio 1, RW): alias pam-pci @pc.ram 00000000000d8000-00000000000dbfff [disabled]
    00000000000d8000-00000000000dbfff (prio 1, R-): alias pam-rom @pc.ram 00000000000d8000-00000000000dbfff
    00000000000d8000-00000000000dbfff (prio 1, RW): alias pam-pci @pci 00000000000d8000-00000000000dbfff [disabled]
    00000000000dc000-00000000000dffff (prio 1, RW): alias pam-ram @pc.ram 00000000000dc000-00000000000dffff [disabled]
    00000000000dc000-00000000000dffff (prio 1, RW): alias pam-pci @pc.ram 00000000000dc000-00000000000dffff [disabled]
    00000000000dc000-00000000000dffff (prio 1, R-): alias pam-rom @pc.ram 00000000000dc000-00000000000dffff
    00000000000dc000-00000000000dffff (prio 1, RW): alias pam-pci @pci 00000000000dc000-00000000000dffff [disabled]
    00000000000e0000-00000000000e3fff (prio 1, RW): alias pam-ram @pc.ram 00000000000e0000-00000000000e3fff [disabled]
    00000000000e0000-00000000000e3fff (prio 1, RW): alias pam-pci @pc.ram 00000000000e0000-00000000000e3fff [disabled]
    00000000000e0000-00000000000e3fff (prio 1, R-): alias pam-rom @pc.ram 00000000000e0000-00000000000e3fff
    00000000000e0000-00000000000e3fff (prio 1, RW): alias pam-pci @pci 00000000000e0000-00000000000e3fff [disabled]
    00000000000e4000-00000000000e7fff (prio 1, RW): alias pam-ram @pc.ram 00000000000e4000-00000000000e7fff [disabled]
    00000000000e4000-00000000000e7fff (prio 1, RW): alias pam-pci @pc.ram 00000000000e4000-00000000000e7fff [disabled]
    00000000000e4000-00000000000e7fff (prio 1, R-): alias pam-rom @pc.ram 00000000000e4000-00000000000e7fff
    00000000000e4000-00000000000e7fff (prio 1, RW): alias pam-pci @pci 00000000000e4000-00000000000e7fff [disabled]
    00000000000e8000-00000000000ebfff (prio 1, RW): alias pam-ram @pc.ram 00000000000e8000-00000000000ebfff [disabled]
    00000000000e8000-00000000000ebfff (prio 1, RW): alias pam-pci @pc.ram 00000000000e8000-00000000000ebfff [disabled]
    00000000000e8000-00000000000ebfff (prio 1, R-): alias pam-rom @pc.ram 00000000000e8000-00000000000ebfff
    00000000000e8000-00000000000ebfff (prio 1, RW): alias pam-pci @pci 00000000000e8000-00000000000ebfff [disabled]
    00000000000ec000-00000000000effff (prio 1, RW): alias pam-ram @pc.ram 00000000000ec000-00000000000effff
    00000000000ec000-00000000000effff (prio 1, RW): alias pam-pci @pc.ram 00000000000ec000-00000000000effff [disabled]
    00000000000ec000-00000000000effff (prio 1, R-): alias pam-rom @pc.ram 00000000000ec000-00000000000effff [disabled]
    00000000000ec000-00000000000effff (prio 1, RW): alias pam-pci @pci 00000000000ec000-00000000000effff [disabled]
    00000000000f0000-00000000000fffff (prio 1, RW): alias pam-ram @pc.ram 00000000000f0000-00000000000fffff [disabled]
    00000000000f0000-00000000000fffff (prio 1, RW): alias pam-pci @pc.ram 00000000000f0000-00000000000fffff [disabled]
    00000000000f0000-00000000000fffff (prio 1, R-): alias pam-rom @pc.ram 00000000000f0000-00000000000fffff
    00000000000f0000-00000000000fffff (prio 1, RW): alias pam-pci @pci 00000000000f0000-00000000000fffff [disabled]
    00000000fec00000-00000000fec00fff (prio 0, RW): kvm-ioapic
    00000000fee00000-00000000feefffff (prio 4096, RW): kvm-apic-msi
    0000000100000000-000000013fffffff (prio 0, RW): alias ram-above-4g @pc.ram 00000000c0000000-00000000ffffffff

address-space: I/O
  0000000000000000-000000000000ffff (prio 0, RW): io
    0000000000000000-0000000000000007 (prio 0, RW): dma-chan
    0000000000000008-000000000000000f (prio 0, RW): dma-cont
    0000000000000020-0000000000000021 (prio 0, RW): kvm-pic
    0000000000000040-0000000000000043 (prio 0, RW): kvm-pit
    0000000000000060-0000000000000060 (prio 0, RW): i8042-data
    0000000000000061-0000000000000061 (prio 0, RW): pcspk
    0000000000000064-0000000000000064 (prio 0, RW): i8042-cmd
    0000000000000070-0000000000000071 (prio 0, RW): rtc
    000000000000007e-000000000000007f (prio 0, RW): kvmvapic
    0000000000000080-0000000000000080 (prio 0, RW): ioport80
    0000000000000081-0000000000000083 (prio 0, RW): dma-page
    0000000000000087-0000000000000087 (prio 0, RW): dma-page
    0000000000000089-000000000000008b (prio 0, RW): dma-page
    000000000000008f-000000000000008f (prio 0, RW): dma-page
    0000000000000092-0000000000000092 (prio 0, RW): port92
    00000000000000a0-00000000000000a1 (prio 0, RW): kvm-pic
    00000000000000b2-00000000000000b3 (prio 0, RW): apm-io
    00000000000000c0-00000000000000cf (prio 0, RW): dma-chan
    00000000000000d0-00000000000000df (prio 0, RW): dma-cont
    00000000000000f0-00000000000000f0 (prio 0, RW): ioportF0
    0000000000000170-0000000000000177 (prio 0, RW): ide
    00000000000001f0-00000000000001f7 (prio 0, RW): ide
    0000000000000376-0000000000000376 (prio 0, RW): ide
    00000000000003b0-00000000000003df (prio 0, RW): cirrus-io
    00000000000003f1-00000000000003f5 (prio 0, RW): fdc
    00000000000003f6-00000000000003f6 (prio 0, RW): ide
    00000000000003f7-00000000000003f7 (prio 0, RW): fdc
    00000000000003f8-00000000000003ff (prio 0, RW): serial
    00000000000004d0-00000000000004d0 (prio 0, RW): kvm-elcr
    00000000000004d1-00000000000004d1 (prio 0, RW): kvm-elcr
    0000000000000510-0000000000000511 (prio 0, RW): fwcfg
    0000000000000514-000000000000051b (prio 0, RW): fwcfg.dma
    0000000000000600-000000000000063f (prio 0, RW): piix4-pm
      0000000000000600-0000000000000603 (prio 0, RW): acpi-evt
      0000000000000604-0000000000000605 (prio 0, RW): acpi-cnt
      0000000000000608-000000000000060b (prio 0, RW): acpi-tmr
    0000000000000700-000000000000073f (prio 0, RW): pm-smbus
    0000000000000cf8-0000000000000cfb (prio 0, RW): pci-conf-idx
    0000000000000cf9-0000000000000cf9 (prio 1, RW): piix3-reset-control
    0000000000000cfc-0000000000000cff (prio 0, RW): pci-conf-data
    0000000000005658-0000000000005658 (prio 0, RW): vmport
    000000000000ae00-000000000000ae13 (prio 0, RW): acpi-pci-hotplug
    000000000000af00-000000000000af1f (prio 0, RW): acpi-cpu-hotplug
    000000000000afe0-000000000000afe3 (prio 0, RW): acpi-gpe0
    000000000000c000-000000000000c0ff (prio 1, RW): pv_channel
    000000000000c100-000000000000c13f (prio 1, RW): virtio-pci
    000000000000c140-000000000000c15f (prio 1, RW): uhci
    000000000000c160-000000000000c17f (prio 1, RW): virtio-pci
    000000000000c180-000000000000c19f (prio 1, RW): virtio-pci
    000000000000c1a0-000000000000c1af (prio 1, RW): piix-bmdma-container
      000000000000c1a0-000000000000c1a3 (prio 0, RW): piix-bmdma
      000000000000c1a4-000000000000c1a7 (prio 0, RW): bmdma
      000000000000c1a8-000000000000c1ab (prio 0, RW): piix-bmdma
      000000000000c1ac-000000000000c1af (prio 0, RW): bmdma

address-space: i440FX
  0000000000000000-ffffffffffffffff (prio 0, RW): alias bus master @system 0000000000000000-ffffffffffffffff [disabled]

address-space: PIIX3
  0000000000000000-ffffffffffffffff (prio 0, RW): alias bus master @system 0000000000000000-ffffffffffffffff [disabled]

address-space: piix3-ide
  0000000000000000-ffffffffffffffff (prio 0, RW): alias bus master @system 0000000000000000-ffffffffffffffff

address-space: PIIX4_PM
  0000000000000000-ffffffffffffffff (prio 0, RW): alias bus master @system 0000000000000000-ffffffffffffffff [disabled]

address-space: piix3-usb-uhci
  0000000000000000-ffffffffffffffff (prio 0, RW): alias bus master @system 0000000000000000-ffffffffffffffff

address-space: virtio-scsi-pci
  0000000000000000-ffffffffffffffff (prio 0, RW): alias bus master @system 0000000000000000-ffffffffffffffff

address-space: virtio-pci-cfg-as
  0000000000000000-00000000007fffff (prio 0, RW): alias virtio-pci-cfg @virtio-pci 0000000000000000-00000000007fffff

address-space: virtio-serial-pci
  0000000000000000-ffffffffffffffff (prio 0, RW): alias bus master @system 0000000000000000-ffffffffffffffff

address-space: virtio-pci-cfg-as
  0000000000000000-00000000007fffff (prio 0, RW): alias virtio-pci-cfg @virtio-pci 0000000000000000-00000000007fffff

address-space: cirrus-vga
  0000000000000000-ffffffffffffffff (prio 0, RW): alias bus master @system 0000000000000000-ffffffffffffffff [disabled]

address-space: virtio-balloon-pci
  0000000000000000-ffffffffffffffff (prio 0, RW): alias bus master @system 0000000000000000-ffffffffffffffff

address-space: virtio-pci-cfg-as
  0000000000000000-00000000007fffff (prio 0, RW): alias virtio-pci-cfg @virtio-pci 0000000000000000-00000000007fffff

address-space: pv_channel
  0000000000000000-ffffffffffffffff (prio 0, RW): alias bus master @system 0000000000000000-ffffffffffffffff [disabled]

address-space: KVM-SMRAM
  0000000000000000-ffffffffffffffff (prio 0, RW): mem-container-smram
    0000000000000000-00000000ffffffff (prio 10, RW): smram
      00000000000a0000-00000000000bffff (prio 0, RW): alias smram-low @pc.ram 00000000000a0000-00000000000bffff
    0000000000000000-ffffffffffffffff (prio 0, RW): alias mem-smram @system 0000000000000000-ffffffffffffffff

memory-region: pc.ram
  0000000000000000-00000000ffffffff (prio 0, RW): pc.ram

memory-region: vga.vram
  0000000000000000-00000000007fffff (prio 1, RW): vga.vram

memory-region: pc.bios
  00000000fffc0000-00000000ffffffff (prio 0, R-): pc.bios

memory-region: pci
  0000000000000000-ffffffffffffffff (prio -1, RW): pci
    00000000000a0000-00000000000bffff (prio 1, RW): cirrus-lowmem-container
      00000000000a0000-00000000000a7fff (prio 1, RW): alias vga.bank0 @vga.vram 0000000000000000-0000000000007fff
      00000000000a0000-00000000000bffff (prio 0, RW): cirrus-low-memory
      00000000000a8000-00000000000affff (prio 1, RW): alias vga.bank1 @vga.vram 0000000000008000-000000000000ffff
    00000000000c0000-00000000000dffff (prio 1, RW): pc.rom
    00000000000e0000-00000000000fffff (prio 1, R-): alias isa-bios @pc.bios 0000000000020000-000000000003ffff
    00000000fc000000-00000000fdffffff (prio 1, RW): cirrus-pci-bar0
      00000000fc000000-00000000fc7fffff (prio 1, RW): vga.vram
      00000000fc000000-00000000fc7fffff (prio 0, RW): cirrus-linear-io
      00000000fd000000-00000000fd3fffff (prio 0, RW): cirrus-bitblt-mmio
    00000000febf0000-00000000febf0fff (prio 1, RW): cirrus-mmio
    00000000febf1000-00000000febf1fff (prio 1, RW): virtio-scsi-pci-msix
      00000000febf1000-00000000febf103f (prio 0, RW): msix-table
      00000000febf1800-00000000febf1807 (prio 0, RW): msix-pba
    00000000febf2000-00000000febf2fff (prio 1, RW): virtio-serial-pci-msix
      00000000febf2000-00000000febf201f (prio 0, RW): msix-table
      00000000febf2800-00000000febf2807 (prio 0, RW): msix-pba
    00000000fffc0000-00000000ffffffff (prio 0, R-): pc.bios

memory-region: system
  0000000000000000-ffffffffffffffff (prio 0, RW): system
    0000000000000000-00000000bfffffff (prio 0, RW): alias ram-below-4g @pc.ram 0000000000000000-00000000bfffffff
    0000000000000000-ffffffffffffffff (prio -1, RW): pci
      00000000000a0000-00000000000bffff (prio 1, RW): cirrus-lowmem-container
        00000000000a0000-00000000000a7fff (prio 1, RW): alias vga.bank0 @vga.vram 0000000000000000-0000000000007fff
        00000000000a0000-00000000000bffff (prio 0, RW): cirrus-low-memory
        00000000000a8000-00000000000affff (prio 1, RW): alias vga.bank1 @vga.vram 0000000000008000-000000000000ffff
      00000000000c0000-00000000000dffff (prio 1, RW): pc.rom
      00000000000e0000-00000000000fffff (prio 1, R-): alias isa-bios @pc.bios 0000000000020000-000000000003ffff
      00000000fc000000-00000000fdffffff (prio 1, RW): cirrus-pci-bar0
        00000000fc000000-00000000fc7fffff (prio 1, RW): vga.vram
        00000000fc000000-00000000fc7fffff (prio 0, RW): cirrus-linear-io
        00000000fd000000-00000000fd3fffff (prio 0, RW): cirrus-bitblt-mmio
      00000000febf0000-00000000febf0fff (prio 1, RW): cirrus-mmio
      00000000febf1000-00000000febf1fff (prio 1, RW): virtio-scsi-pci-msix
        00000000febf1000-00000000febf103f (prio 0, RW): msix-table
        00000000febf1800-00000000febf1807 (prio 0, RW): msix-pba
      00000000febf2000-00000000febf2fff (prio 1, RW): virtio-serial-pci-msix
        00000000febf2000-00000000febf201f (prio 0, RW): msix-table
        00000000febf2800-00000000febf2807 (prio 0, RW): msix-pba
      00000000fffc0000-00000000ffffffff (prio 0, R-): pc.bios
    00000000000a0000-00000000000bffff (prio 1, RW): alias smram-region @pci 00000000000a0000-00000000000bffff
    00000000000c0000-00000000000c3fff (prio 1, RW): alias pam-ram @pc.ram 00000000000c0000-00000000000c3fff [disabled]
    00000000000c0000-00000000000c3fff (prio 1, RW): alias pam-pci @pc.ram 00000000000c0000-00000000000c3fff [disabled]
    00000000000c0000-00000000000c3fff (prio 1, R-): alias pam-rom @pc.ram 00000000000c0000-00000000000c3fff
    00000000000c0000-00000000000c3fff (prio 1, RW): alias pam-pci @pci 00000000000c0000-00000000000c3fff [disabled]
    00000000000c4000-00000000000c7fff (prio 1, RW): alias pam-ram @pc.ram 00000000000c4000-00000000000c7fff [disabled]
    00000000000c4000-00000000000c7fff (prio 1, RW): alias pam-pci @pc.ram 00000000000c4000-00000000000c7fff [disabled]
    00000000000c4000-00000000000c7fff (prio 1, R-): alias pam-rom @pc.ram 00000000000c4000-00000000000c7fff
    00000000000c4000-00000000000c7fff (prio 1, RW): alias pam-pci @pci 00000000000c4000-00000000000c7fff [disabled]
    00000000000c8000-00000000000cbfff (prio 1, RW): alias pam-ram @pc.ram 00000000000c8000-00000000000cbfff [disabled]
    00000000000c8000-00000000000cbfff (prio 1, RW): alias pam-pci @pc.ram 00000000000c8000-00000000000cbfff [disabled]
    00000000000c8000-00000000000cbfff (prio 1, R-): alias pam-rom @pc.ram 00000000000c8000-00000000000cbfff
    00000000000c8000-00000000000cbfff (prio 1, RW): alias pam-pci @pci 00000000000c8000-00000000000cbfff [disabled]
    00000000000c9000-00000000000cbfff (prio 1000, RW): alias kvmvapic-rom @pc.ram 00000000000c9000-00000000000cbfff
    00000000000cc000-00000000000cffff (prio 1, RW): alias pam-ram @pc.ram 00000000000cc000-00000000000cffff [disabled]
    00000000000cc000-00000000000cffff (prio 1, RW): alias pam-pci @pc.ram 00000000000cc000-00000000000cffff [disabled]
    00000000000cc000-00000000000cffff (prio 1, R-): alias pam-rom @pc.ram 00000000000cc000-00000000000cffff
    00000000000cc000-00000000000cffff (prio 1, RW): alias pam-pci @pci 00000000000cc000-00000000000cffff [disabled]
    00000000000d0000-00000000000d3fff (prio 1, RW): alias pam-ram @pc.ram 00000000000d0000-00000000000d3fff [disabled]
    00000000000d0000-00000000000d3fff (prio 1, RW): alias pam-pci @pc.ram 00000000000d0000-00000000000d3fff [disabled]
    00000000000d0000-00000000000d3fff (prio 1, R-): alias pam-rom @pc.ram 00000000000d0000-00000000000d3fff
    00000000000d0000-00000000000d3fff (prio 1, RW): alias pam-pci @pci 00000000000d0000-00000000000d3fff [disabled]
    00000000000d4000-00000000000d7fff (prio 1, RW): alias pam-ram @pc.ram 00000000000d4000-00000000000d7fff [disabled]
    00000000000d4000-00000000000d7fff (prio 1, RW): alias pam-pci @pc.ram 00000000000d4000-00000000000d7fff [disabled]
    00000000000d4000-00000000000d7fff (prio 1, R-): alias pam-rom @pc.ram 00000000000d4000-00000000000d7fff
    00000000000d4000-00000000000d7fff (prio 1, RW): alias pam-pci @pci 00000000000d4000-00000000000d7fff [disabled]
    00000000000d8000-00000000000dbfff (prio 1, RW): alias pam-ram @pc.ram 00000000000d8000-00000000000dbfff [disabled]
    00000000000d8000-00000000000dbfff (prio 1, RW): alias pam-pci @pc.ram 00000000000d8000-00000000000dbfff [disabled]
    00000000000d8000-00000000000dbfff (prio 1, R-): alias pam-rom @pc.ram 00000000000d8000-00000000000dbfff
    00000000000d8000-00000000000dbfff (prio 1, RW): alias pam-pci @pci 00000000000d8000-00000000000dbfff [disabled]
    00000000000dc000-00000000000dffff (prio 1, RW): alias pam-ram @pc.ram 00000000000dc000-00000000000dffff [disabled]
    00000000000dc000-00000000000dffff (prio 1, RW): alias pam-pci @pc.ram 00000000000dc000-00000000000dffff [disabled]
    00000000000dc000-00000000000dffff (prio 1, R-): alias pam-rom @pc.ram 00000000000dc000-00000000000dffff
    00000000000dc000-00000000000dffff (prio 1, RW): alias pam-pci @pci 00000000000dc000-00000000000dffff [disabled]
    00000000000e0000-00000000000e3fff (prio 1, RW): alias pam-ram @pc.ram 00000000000e0000-00000000000e3fff [disabled]
    00000000000e0000-00000000000e3fff (prio 1, RW): alias pam-pci @pc.ram 00000000000e0000-00000000000e3fff [disabled]
    00000000000e0000-00000000000e3fff (prio 1, R-): alias pam-rom @pc.ram 00000000000e0000-00000000000e3fff
    00000000000e0000-00000000000e3fff (prio 1, RW): alias pam-pci @pci 00000000000e0000-00000000000e3fff [disabled]
    00000000000e4000-00000000000e7fff (prio 1, RW): alias pam-ram @pc.ram 00000000000e4000-00000000000e7fff [disabled]
    00000000000e4000-00000000000e7fff (prio 1, RW): alias pam-pci @pc.ram 00000000000e4000-00000000000e7fff [disabled]
    00000000000e4000-00000000000e7fff (prio 1, R-): alias pam-rom @pc.ram 00000000000e4000-00000000000e7fff
    00000000000e4000-00000000000e7fff (prio 1, RW): alias pam-pci @pci 00000000000e4000-00000000000e7fff [disabled]
    00000000000e8000-00000000000ebfff (prio 1, RW): alias pam-ram @pc.ram 00000000000e8000-00000000000ebfff [disabled]
    00000000000e8000-00000000000ebfff (prio 1, RW): alias pam-pci @pc.ram 00000000000e8000-00000000000ebfff [disabled]
    00000000000e8000-00000000000ebfff (prio 1, R-): alias pam-rom @pc.ram 00000000000e8000-00000000000ebfff
    00000000000e8000-00000000000ebfff (prio 1, RW): alias pam-pci @pci 00000000000e8000-00000000000ebfff [disabled]
    00000000000ec000-00000000000effff (prio 1, RW): alias pam-ram @pc.ram 00000000000ec000-00000000000effff
    00000000000ec000-00000000000effff (prio 1, RW): alias pam-pci @pc.ram 00000000000ec000-00000000000effff [disabled]
    00000000000ec000-00000000000effff (prio 1, R-): alias pam-rom @pc.ram 00000000000ec000-00000000000effff [disabled]
    00000000000ec000-00000000000effff (prio 1, RW): alias pam-pci @pci 00000000000ec000-00000000000effff [disabled]
    00000000000f0000-00000000000fffff (prio 1, RW): alias pam-ram @pc.ram 00000000000f0000-00000000000fffff [disabled]
    00000000000f0000-00000000000fffff (prio 1, RW): alias pam-pci @pc.ram 00000000000f0000-00000000000fffff [disabled]
    00000000000f0000-00000000000fffff (prio 1, R-): alias pam-rom @pc.ram 00000000000f0000-00000000000fffff
    00000000000f0000-00000000000fffff (prio 1, RW): alias pam-pci @pci 00000000000f0000-00000000000fffff [disabled]
    00000000fec00000-00000000fec00fff (prio 0, RW): kvm-ioapic
    00000000fee00000-00000000feefffff (prio 4096, RW): kvm-apic-msi
    0000000100000000-000000013fffffff (prio 0, RW): alias ram-above-4g @pc.ram 00000000c0000000-00000000ffffffff

memory-region: virtio-pci
  0000000000000000-00000000007fffff (prio 0, RW): virtio-pci

memory-region: virtio-pci
  0000000000000000-00000000007fffff (prio 0, RW): virtio-pci

memory-region: virtio-pci
  0000000000000000-00000000007fffff (prio 0, RW): virtio-pci

```


