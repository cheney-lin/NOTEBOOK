# 内存热插流程分析
## Libvirt热插内存流程
无论是利用python还是virsh命令添加设备，都会走qemuDomainAttachDeviceFlags函数：
``` c
static int qemuDomainAttachDeviceFlags(virDomainPtr dom, const char *xml,
                                       unsigned int flags)
{
    virQEMUDriverPtr driver = dom->conn->privateData;
    virDomainObjPtr vm = NULL;
    virDomainDefPtr vmdef = NULL;
    virDomainDeviceDefPtr dev = NULL, dev_copy = NULL;
    int ret = -1;
    unsigned int parse_flags = VIR_DOMAIN_DEF_PARSE_INACTIVE |
                               VIR_DOMAIN_DEF_PARSE_ABI_UPDATE;
    virQEMUCapsPtr qemuCaps = NULL;
    qemuDomainObjPrivatePtr priv;
    virQEMUDriverConfigPtr cfg = NULL;
    virCapsPtr caps = NULL;

    virCheckFlags(VIR_DOMAIN_AFFECT_LIVE |
                  VIR_DOMAIN_AFFECT_CONFIG, -1);

    virNWFilterReadLockFilterUpdates();

    cfg = virQEMUDriverGetConfig(driver);

    if (!(caps = virQEMUDriverGetCapabilities(driver, false)))
        goto cleanup;

    if (!(vm = qemuDomObjFromDomain(dom)))
        goto cleanup;

    priv = vm->privateData;

    if (virDomainAttachDeviceFlagsEnsureACL(dom->conn, vm->def, flags) < 0)
        goto cleanup;

    if (qemuDomainObjBeginJob(driver, vm, QEMU_JOB_MODIFY) < 0)
        goto cleanup;

    if (virDomainObjUpdateModificationImpact(vm, &flags) < 0)
        goto endjob;

    dev = dev_copy = virDomainDeviceDefParse(xml, vm->def,
                                             caps, driver->xmlopt,
                                             parse_flags);
    if (dev == NULL)
        goto endjob;

    if ((virDomainDeviceType) dev->type == VIR_DOMAIN_DEVICE_DISK) {
        if (dev->data.disk->iothread &&
            virDomainDefCheckDuplicateIOThreadID(vm->def, dev->data.disk)) {
            virReportError(VIR_ERR_CONFIG_UNSUPPORTED,
                           _("A disk is already using iothread_id '%u'"),
                           dev->data.disk->iothread);
            goto endjob;
        }
    }

    if (flags & VIR_DOMAIN_AFFECT_CONFIG &&
        flags & VIR_DOMAIN_AFFECT_LIVE) {
        /* If we are affecting both CONFIG and LIVE
         * create a deep copy of device as adding
         * to CONFIG takes one instance.
         */
        dev_copy = virDomainDeviceDefCopy(dev, vm->def, caps, driver->xmlopt);
        if (!dev_copy)
            goto endjob;
    }

    if ((dev->type == VIR_DOMAIN_DEVICE_NET) && (flags & VIR_DOMAIN_AFFECT_LIVE)) {
        if (virDomainDefCheckDuplicateMacAddress(vm->def, dev->data.net)) {
            char mac_address[VIR_MAC_STRING_BUFLEN];
            virMacAddrFormat(&dev->data.net->mac, mac_address);
            virReportError(VIR_ERR_CONFIG_UNSUPPORTED,
                           _("the mac address %s is used more than one time"), mac_address);
            goto endjob;
        }
    }

    if (priv->qemuCaps)
        qemuCaps = virObjectRef(priv->qemuCaps);
    else if (!(qemuCaps = virQEMUCapsCacheLookup(driver->qemuCapsCache, vm->def->emulator)))
        goto cleanup;

    if (flags & VIR_DOMAIN_AFFECT_CONFIG) {
        /* Make a copy for updated domain. */
        vmdef = virDomainObjCopyPersistentDef(vm, caps, driver->xmlopt);
        if (!vmdef)
            goto endjob;

        if (dev->type == VIR_DOMAIN_DEVICE_NET) {
            if (virDomainDefCheckDuplicateMacAddress(vmdef, dev->data.net)) {
                char mac_address[VIR_MAC_STRING_BUFLEN];
                virMacAddrFormat(&dev->data.net->mac, mac_address);
                virReportError(VIR_ERR_CONFIG_UNSUPPORTED,
                               _("the mac address %s is used more than one time"), mac_address);
                goto endjob;
            }
        }

        if (virDomainDefCompatibleDevice(vmdef, dev,
                                         VIR_DOMAIN_DEVICE_ACTION_ATTACH) < 0)
            goto endjob;

        if ((ret = qemuDomainAttachDeviceConfig(qemuCaps, vmdef, dev,
                                                dom->conn)) < 0)
            goto endjob;
    }

    if (flags & VIR_DOMAIN_AFFECT_LIVE) {
        if (virDomainDefCompatibleDevice(vm->def, dev_copy,
                                         VIR_DOMAIN_DEVICE_ACTION_ATTACH) < 0)
            goto endjob;

        if ((ret = qemuDomainAttachDeviceLive(vm, dev_copy, dom)) < 0)
            goto endjob;
        /*
         * update domain status forcibly because the domain status may be
         * changed even if we failed to attach the device. For example,
         * a new controller may be created.
         */
        if (virDomainSaveStatus(driver->xmlopt, cfg->stateDir, vm, driver->caps) < 0) {
            ret = -1;
            goto endjob;
        }
    }
```
该函数的三个参数在4.4接口描述已经说明过了，其中flags很关键，它会影响后续的行为，以下讨论xml为内存设备并且flags=3的情况；这个函数主要作了以下几件事情：

1. 解析xml，生成相应的设备对象，设备对象包含了xml描述的所有属性；
2. 更新当前设备的xml
3. 热插内存，也就是给qemu发qmp消息

## QEMU热插内存流程
上文提到Libvirt向QEMU下发了两条qmp命令，QEMU侧的处理函数分别是qmp_object_add 和 qmp_device_add，前者创建了QOM对象，后者创建了设备并将其初始化，以下的分析从这两个函数展开。
### qmp_object_add
创建一个新的设备对象(memory后端设备)，dimm设备的创建在qmp_device_add流程中
当后端设备类型为ram时调用栈如下：
``` c
qmp_object_add
    user_creatable_add_type //check是否继承了TYPE_USER_CREATABLE接口，是否为虚类等
        object_new
        object_property_set 设置size等属性
        object_property_add_child
        user_creatable_complete
            host_memory_backend_memory_complete(ucc->complete)
                ram_backend_memory_alloc(bc->alloc)
                    object_get_canonical_path_component
                    memory_region_init_ram
                        qemu_ram_alloc
                            ram_block_add
                                last_ram_offset
                                find_ram_offset(寻找新内存条可以映射的物理地址空间)
                                qemu_anon_ram_alloc(phys_mem_alloc) 为内存条真正申请内存下面
```
来看object_add主要创建了Host Memory Backend:
``` c
void qmp_object_add(const char *type, const char *id,
                    bool has_props, QObject *props, Error **errp)
{
    QDict *pdict;
    Visitor *v;
    Object *obj;

    if (props) {
        pdict = qobject_to_qdict(props);
        if (!pdict) {
            error_setg(errp, QERR_INVALID_PARAMETER_TYPE, "props", "dict");
            return;
        }
        QINCREF(pdict);
    } else {
        pdict = qdict_new();
    }

    v = qobject_input_visitor_new(QOBJECT(pdict), true);
    obj = user_creatable_add_type(type, id, pdict, v, errp);
    visit_free(v);
    if (obj) {
        object_unref(obj);
    }
    QDECREF(pdict);
}
```
参考qmp命令的格式就知道这几个入参是什么了
``` c
2018-02-27T17:23:13.817362+08:00|info|qemu[5063]|[5063]|do_qmp_dispatch[109]|: qmp_cmd_name: object-add, 
arguments: {"qom-type": "memory-backend-ram", "props": {"size": 1073741824}, "id": "memdimm2"}
```
配置了大页时memory-backend类型是不同的：
``` c
2018-02-14T16:59:05.542215+08:00|info|qemu[17053]|[17053]|do_qmp_dispatch[109]|: qmp_cmd_name: object-add, arguments: {"qom-type": "memory-backend-file", "props": {"share": true, "prealloc": true, "size": 67108864, "mem-path": "/dev/hugepages/libvirt/qemu/11-redhat_7.1"}, "id": "memdimm0"}
```

user_creatable_complete用来创建“memory-backend-ram”设备，这个设备的相关定义在backends/hostmem-ram.c：
``` c
#define TYPE_MEMORY_BACKEND_RAM "memory-backend-ram"


static void
ram_backend_memory_alloc(HostMemoryBackend *backend, Error **errp)
{
    char *path;

    if (!backend->size) {
        error_setg(errp, "can't create backend with size 0");
        return;
    }

    path = object_get_canonical_path_component(OBJECT(backend));
    memory_region_init_ram(&backend->mr, OBJECT(backend), path,
                           backend->size, errp);
    g_free(path);
}

static void
ram_backend_class_init(ObjectClass *oc, void *data)
{
    HostMemoryBackendClass *bc = MEMORY_BACKEND_CLASS(oc);

    bc->alloc = ram_backend_memory_alloc;
}

static const TypeInfo ram_backend_info = {
    .name = TYPE_MEMORY_BACKEND_RAM,
    .parent = TYPE_MEMORY_BACKEND,
    .class_init = ram_backend_class_init,
};

static void register_types(void)
{
    type_register_static(&ram_backend_info);
}

type_init(register_types);
```

首先object_new，根据已有知识，这是对象的”构造函数”，对于”memory-backend-ram”来说没做什么，只是把对象类的相关函数注册了一下。
然后是object_property_set ，设置了相关属性，比如size等；再将”memory-backend-ram”对象作为child加入到container对象的属性hash表中，建立起两者的父子关系。(Object之间的关系见 https://wiki.qemu.org/Features/QOM)

``` c
void object_property_add_child(Object *obj, const char *name,
                               Object *child, Error **errp)
{
    Error *local_err = NULL;
    gchar *type;
    ObjectProperty *op;

    if (child->parent != NULL) {
        error_setg(errp, "child object is already parented");
        return;
    }

    type = g_strdup_printf("child<%s>", object_get_typename(OBJECT(child)));

    op = object_property_add(obj, name, type, object_get_child_property, NULL,
                             object_finalize_child_property, child, &local_err);
    if (local_err) {
        error_propagate(errp, local_err);
        goto out;
    }

    op->resolve = object_resolve_child_property;
    object_ref(child);
    child->parent = obj;

out:
    g_free(type);
}
```
然后调用user_creatable_complete，实际调用的是host_memory_backend_memory_complete，在这个函数中为内存条真正申请内存、将qemu申请的虚拟内存与host numa绑定。
初始化后端内存对应的mr并完成内存申请。

``` c
static void
ram_backend_memory_alloc(HostMemoryBackend *backend, Error **errp)
{
    char *path;

    if (!backend->size) {
        error_setg(errp, "can't create backend with size 0");
        return;
    }

    path = object_get_canonical_path_component(OBJECT(backend));
    memory_region_init_ram(&backend->mr, OBJECT(backend), path,
                           backend->size, errp);
    g_free(path);
}
```

``` c
void memory_region_init_ram(MemoryRegion *mr,
                            Object *owner,
                            const char *name,
                            uint64_t size,
                            Error **errp)
{
    memory_region_init(mr, owner, name, size);
    mr->ram = true;
    mr->terminates = true;
    mr->destructor = memory_region_destructor_ram;
    mr->ram_block = qemu_ram_alloc(size, mr, errp);
    mr->dirty_log_mask = tcg_enabled() ? (1 << DIRTY_MEMORY_CODE) : 0;
}
```

申请内存
``` c
static
RAMBlock *qemu_ram_alloc_internal(ram_addr_t size, ram_addr_t max_size,
                                  void (*resized)(const char*,
                                                  uint64_t length,
                                                  void *host),
                                  void *host, bool resizeable,
                                  MemoryRegion *mr, Error **errp)
{
    RAMBlock *new_block;
    Error *local_err = NULL;

    size = HOST_PAGE_ALIGN(size);
    max_size = HOST_PAGE_ALIGN(max_size);
    new_block = g_malloc0(sizeof(*new_block));
    new_block->mr = mr;
    new_block->resized = resized;
    new_block->used_length = size;
    new_block->max_length = max_size;
    assert(max_size >= size);
    new_block->fd = -1;
    new_block->page_size = getpagesize();
    new_block->host = host;
    if (host) {
        new_block->flags |= RAM_PREALLOC;
    }
    if (resizeable) {
        new_block->flags |= RAM_RESIZEABLE;
    }
    ram_block_add(new_block, &local_err);
    if (local_err) {
        g_free(new_block);
        error_propagate(errp, local_err);
        return NULL;
    }
    return new_block;
}
```
last_ram_offset找到当前ram所占的物理地址空间的大小，find_ram_offset寻找新的后端内存可以安插的物理地址，qemu_anon_ram_alloc_noreserve真正申请内存，如果新的内存物理地址范围大于原来的物理地址空间的范围，就去更新两个bitmap(与热迁移相关)。
至此object_add的流程就分析完了。

### qmp_device_add
创建dimm 设备并插入vm
调用栈如下：
``` c
qmp_device_add
    qdev_device_add
        DEVICE(object_new(driver)) 创建dimm设备对象
            pc_dimm_init
        qemu_opt_foreach(opts, set_property, dev, &err) 设置属性 包括 addr slot size等
        object_property_set_bool(OBJECT(dev), true, "realized", &err)
                    device_set_realized
                        pc_dimm_realize(dc->realize)
                        pc_machine_device_plug_cb(hotplug_handler_plug)
                            pc_dimm_plug
                                pc_dimm_memory_plug
                                    memory_region_add_subregion
                                        memory_region_add_subregion_common
                                            memory_region_update_container_subregions
                                                memory_region_transaction_commit
                                piix4_device_plug_cb(hhc->plug)
                                    acpi_memory_plug_cb
                                        acpi_send_event
                                            piix4_send_gpe
                                                acpi_send_gpe_event
                                                    acpi_update_sci
```

既然要讨论dimm设备的创建，那么首先要了解下这个设备的模型，相关代码在hw/mem/pc-dimm.c
``` c
static TypeInfo pc_dimm_info = {
    .name          = TYPE_PC_DIMM,
    .parent        = TYPE_DEVICE,
    .instance_size = sizeof(PCDIMMDevice),
    .instance_init = pc_dimm_init,
    .class_init    = pc_dimm_class_init,
    .class_size    = sizeof(PCDIMMDeviceClass),
};
```

类初始化
``` c
static void pc_dimm_class_init(ObjectClass *oc, void *data)
{
    DeviceClass *dc = DEVICE_CLASS(oc);
    PCDIMMDeviceClass *ddc = PC_DIMM_CLASS(oc);

    dc->realize = pc_dimm_realize;
    dc->unrealize = pc_dimm_unrealize;
    dc->props = pc_dimm_properties;
    dc->desc = "DIMM memory module";

    ddc->get_memory_region = pc_dimm_get_memory_region;
    ddc->get_vmstate_memory_region = pc_dimm_get_vmstate_memory_region;
}
```
dimm类具有的属性包括addr(物理地址) slot(内存槽号) node(numa节点号)：
``` c
static Property pc_dimm_properties[] = {
    DEFINE_PROP_UINT64(PC_DIMM_ADDR_PROP, PCDIMMDevice, addr, 0),
    DEFINE_PROP_UINT32(PC_DIMM_NODE_PROP, PCDIMMDevice, node, 0),
    DEFINE_PROP_INT32(PC_DIMM_SLOT_PROP, PCDIMMDevice, slot,
                      PC_DIMM_UNASSIGNED_SLOT),
    DEFINE_PROP_END_OF_LIST(),
};
```
在对象初始化的时候添加了size memory_backend属性，size是内存大小，memory_backend可以是ram也可以是文件，也就是说在dimm设备初始化后它具有了5个属性。
``` c
static void pc_dimm_init(Object *obj)
{
    PCDIMMDevice *dimm = PC_DIMM(obj);

    object_property_add(obj, PC_DIMM_SIZE_PROP, "int", pc_dimm_get_size,
                        NULL, NULL, NULL, &error_abort);
    object_property_add_link(obj, PC_DIMM_MEMDEV_PROP, TYPE_MEMORY_BACKEND,
                             (Object **)&dimm->hostmem,
                             pc_dimm_check_memdev_is_busy,
                             OBJ_PROP_LINK_UNREF_ON_RELEASE,
                             &error_abort);
}
```
device_add的qmp命令参数：
``` c
2018-02-27T17:23:13.819014+08:00|info|qemu[5063]|[5063]|do_qmp_dispatch[109]|: qmp_cmd_name: device_add, arguments: {"memdev": "memdimm2", "driver": "pc-dimm", "slot": "2", "node": "0", "id": "dimm2"}
```
qdev_device_add是添加设备的入口函数，在虚拟机启动时创建设备以及热插设备时都会进这个函数，它的主要作用是新建一个设备并将它插入到对应的总线中去。以下针对dimm设备来分析一下。
1. 首先解析命令参数获得driver(“pc-dimm”)，bus(NULL)等；
2. object_new创建设备；
3. object_property_add_child 加入到/peripheral为根的qom-tree中去；
4. qemu_opt_foreach(opts, set_property, dev, &err) 将解析出来的参数设置为dimm设备的属性；
5. object_property_set_bool 设置realized属性来激活设备，这里面有热插内存的动作。
  追随着回调函数的脚步来到device_set_realized，主要调用了dc->realize 和 hotplug_handler_plug，对应的回调函数是pc_dimm_realize 和 pc_machine_device_plug_cb(pc_dimm_plug)，前者只是一些错误检查，后者比较关键。

``` c
static void pc_dimm_plug(HotplugHandler *hotplug_dev,
                         DeviceState *dev, Error **errp)
{
    HotplugHandlerClass *hhc;
    Error *local_err = NULL;
    PCMachineState *pcms = PC_MACHINE(hotplug_dev);
    PCMachineClass *pcmc = PC_MACHINE_GET_CLASS(pcms);
    PCDIMMDevice *dimm = PC_DIMM(dev);
    PCDIMMDeviceClass *ddc = PC_DIMM_GET_CLASS(dimm);
    MemoryRegion *mr = ddc->get_memory_region(dimm);
    uint64_t align = TARGET_PAGE_SIZE;

    if (memory_region_get_alignment(mr) && pcmc->enforce_aligned_dimm) {
        align = memory_region_get_alignment(mr);
    }

    if (!pcms->acpi_dev) {
        error_setg(&local_err,
                   "memory hotplug is not enabled: missing acpi device");
        goto out;
    }

    pc_dimm_memory_plug(dev, &pcms->hotplug_memory, mr, align, &local_err);
    if (local_err) {
        goto out;
    }

    if (object_dynamic_cast(OBJECT(dev), TYPE_NVDIMM)) {
        nvdimm_plug(&pcms->acpi_nvdimm_state);
    }

    hpms = &pcms->hotplug_memory;
    hhc = HOTPLUG_HANDLER_GET_CLASS(pcms->acpi_dev);
    hhc->plug(HOTPLUG_HANDLER(pcms->acpi_dev), dev, &error_abort);
out:
    error_propagate(errp, local_err);
}
```
1. memory_region_get_alignment 获得页面大小；
2. pc_dimm_memory_plug 创建dimm设备，更新guest物理地址空间拓扑，与KVM交互等；
3. piix4_device_plug_cb(hhc->plug)，acpi发生一些事情
  我们比较关心的当然是pc_dimm_memory_plug了，做了以下几件事：
4. 根据当前虚拟机的状况计算得到dimm设备的物理地址以及slot号；
5. 添加新的mr，更新内存视图及拓扑结构，通过ioctl(KVM_SET_USER_MEMORY_REGION)给kvm下发地址空间的变化；
6. 将新的mr关联到guest numa节点
  memory_region_add_subregion 就是更新mr的拓扑等，涉及到很多QEMU侧内存虚拟化的知识，非常复杂，不展开说了，可以参考http://bobao.360.cn/learning/detail/4092.html。
