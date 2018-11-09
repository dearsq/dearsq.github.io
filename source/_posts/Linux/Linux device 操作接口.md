title: Linux device 操作接口
date: 2018-11-9 21:00:00
tags: Linux

---
## device 操作 API

### 将新的 device 注册到设备模型 
device_create->device_create_vargs->device_register->device_add 
```
struct device *device_create(struct class *class, struct device *parent,
                 dev_t devt, void *drvdata, const char *fmt, ...)
{
    ......
    dev = device_create_vargs(class, parent, devt, drvdata, fmt, vargs);
    ......
    return dev;
}
```
```
struct device *device_create_vargs(struct class *class, struct device *parent,
                   dev_t devt, void *drvdata, const char *fmt,
                   va_list args)
{
    ......

    dev->devt = devt;
    dev->class = class;
    dev->parent = parent;
    dev->release = device_create_release;
    dev_set_drvdata(dev, drvdata);
    ......
    retval = device_register(dev);
    ......
}
```
```
int device_register(struct device *dev) {
	device_initialize(dev); // 调用初始化 device 结构
	return device_add(dev); // 真正将 device 对象 dev 插入设备模型中
}
```
`device_initialize()` 初始化嵌入的 kobject 结构 dev->kobj, 初始化列表中的孩子列表 kobj->klist_children，初始化 DMA 冲池 dev->dma_pools, 初始化自旋锁 dev->devres_lock等
`device_add()`首先是通过kboject_add()函数将它添加到kobject层次，再把它添加都全局和兄弟链表中，最后添加到其他相关的子系统的驱动程序模型，完成device对象的注册。

### 从设备模型销毁 device
device_destroy() 用于从linux内核系统设备驱动程序模型中移除一个设备，并删除/sys/devices/virtual目录下对应的设备目录及/dev/目录下对应的设备文件
```
void device_destroy(struct class *dev, dev_t devt);
```

### 补充 device create register add 区别
可以参考这篇文章 : https://blog.csdn.net/zifehng/article/details/73844845

### 补充 class_create 和 device_create 区别
条件: 如果用户空间支持 **udev** 
可以利用内核为我们提供的一组 API , 实现在模块加载的时候自动在 /dev 目录下创建相应设备节点，并在卸载模块时删除该节点.
内核中定义了struct class结构体，其实例化的对象对应一个类，
内核同时提供了class_create(…)函数，可以用它来创建一个类，这个类存放于sysfs下面，
一旦创建好了这个类，再调用device_create(…)函数来在/dev目录下创建相应的设备节点。
这样，加载模块的时候，用户空间中的udev会自动响应device_create(…)函数，去/sysfs下寻找对应的类从而创建设备节点。

SOP 如下:
1.  `mem_class = class_create(THIS_MODULE,"younix_class_char");` 
建立逻辑设备类, 在 /sys/class/ 下建立了 younix_class_char 目录
2.  `device_create(mem_class,NULL,MKDEV(MEM_MAJOR,MEM_MINOR),NULL,"younix_class_char");` 
建立设备节点, 在 /dev/ 下自动建立 younix_class_char 建立设备节点

Demo 可以参考 : https://blog.csdn.net/tq384998430/article/details/54342044 中 字符设备驱动程序 的 Demo

## struct device 
```
/**
 * struct device - The basic device structure
 * @parent:	The device's "parent" device, the device to which it is attached.
 * 		In most cases, a parent device is some sort of bus or host
 * 		controller. If parent is NULL, the device, is a top-level device,
 * 		which is not usually what you want.
 * @p:		Holds the private data of the driver core portions of the device.
 * 		See the comment of the struct device_private for detail.
 * @kobj:	A top-level, abstract class from which other classes are derived.
 * @init_name:	Initial name of the device.
 * @type:	The type of device.
 * 		This identifies the device type and carries type-specific
 * 		information.
 * @mutex:	Mutex to synchronize calls to its driver.
 * @bus:	Type of bus device is on.
 * @driver:	Which driver has allocated this
 * @platform_data: Platform data specific to the device.
 * 		Example: For devices on custom boards, as typical of embedded
 * 		and SOC based hardware, Linux often uses platform_data to point
 * 		to board-specific structures describing devices and how they
 * 		are wired.  That can include what ports are available, chip
 * 		variants, which GPIO pins act in what additional roles, and so
 * 		on.  This shrinks the "Board Support Packages" (BSPs) and
 * 		minimizes board-specific #ifdefs in drivers.
 * @driver_data: Private pointer for driver specific info.
 * @power:	For device power management.
 * 		See Documentation/power/devices.txt for details.
 * @pm_domain:	Provide callbacks that are executed during system suspend,
 * 		hibernation, system resume and during runtime PM transitions
 * 		along with subsystem-level and driver-level callbacks.
 * @pins:	For device pin management.
 *		See Documentation/pinctrl.txt for details.
 * @numa_node:	NUMA node this device is close to.
 * @dma_mask:	Dma mask (if dma'ble device).
 * @coherent_dma_mask: Like dma_mask, but for alloc_coherent mapping as not all
 * 		hardware supports 64-bit addresses for consistent allocations
 * 		such descriptors.
 * @dma_pfn_offset: offset of DMA memory range relatively of RAM
 * @dma_parms:	A low level driver may set these to teach IOMMU code about
 * 		segment limitations.
 * @dma_pools:	Dma pools (if dma'ble device).
 * @dma_mem:	Internal for coherent mem override.
 * @cma_area:	Contiguous memory area for dma allocations
 * @archdata:	For arch-specific additions.
 * @of_node:	Associated device tree node.
 * @acpi_node:	Associated ACPI device node.
 * @devt:	For creating the sysfs "dev".
 * @id:		device instance
 * @devres_lock: Spinlock to protect the resource of the device.
 * @devres_head: The resources list of the device.
 * @knode_class: The node used to add the device to the class list.
 * @class:	The class of the device.
 * @groups:	Optional attribute groups.
 * @release:	Callback to free the device after all references have
 * 		gone away. This should be set by the allocator of the
 * 		device (i.e. the bus driver that discovered the device).
 * @iommu_group: IOMMU group the device belongs to.
 *
 * @offline_disabled: If set, the device is permanently online.
 * @offline:	Set after successful invocation of bus type's .offline().
 *
 * At the lowest level, every device in a Linux system is represented by an
 * instance of struct device. The device structure contains the information
 * that the device model core needs to model the system. Most subsystems,
 * however, track additional information about the devices they host. As a
 * result, it is rare for devices to be represented by bare device structures;
 * instead, that structure, like kobject structures, is usually embedded within
 * a higher-level representation of the device.
 */
struct device {
	struct device		*parent; //指向父设备

	struct device_private	*p; 

	struct kobject kobj;         //内嵌的 kobj 对象
	const char		*init_name; /* initial name of the device */
	const struct device_type *type;

	struct mutex		mutex;	/* mutex to synchronize calls to
					 * its driver.
					 */

	struct bus_type	*bus;		/* type of bus device is on */
	struct device_driver *driver;	/* which driver has allocated this
					   device */
	void		*platform_data;	/* Platform specific data, device
					   core doesn't touch it */
	void		*driver_data;	/* Driver data, set and get with
					   dev_set/get_drvdata */
	struct dev_pm_info	power;
	struct dev_pm_domain	*pm_domain;

	u64		*dma_mask;	/* dma mask (if dma'able device) */
	u64		coherent_dma_mask;/* Like dma_mask, but for
					     alloc_coherent mappings as
					     not all hardware supports
					     64 bit addresses for consistent
					     allocations such descriptors. */
	unsigned long	dma_pfn_offset;

	struct device_dma_parameters *dma_parms;

	struct list_head	dma_pools;	/* dma pools (if dma'ble) */

	struct dma_coherent_mem	*dma_mem; /* internal for coherent mem
					     override */
	/* arch specific additions */
	struct dev_archdata	archdata;

	struct device_node	*of_node; /* associated device tree node */
	struct acpi_dev_node	acpi_node; /* associated ACPI device node */

	dev_t			devt;	/* dev_t, creates the sysfs "dev" */
	u32			id;	/* device instance */

	spinlock_t		devres_lock;
	struct list_head	devres_head;

	struct klist_node	knode_class;
	struct class		*class;
	const struct attribute_group **groups;	/* optional groups */

	void	(*release)(struct device *dev);
	struct iommu_group	*iommu_group;

	bool			offline_disabled:1;
	bool			offline:1;
};

```

## Demo
以一个简单的led设备字符设备驱动为例 , 主要关注 **创建设备节点** 的实现 , 这里是以 device_create 进行创建
```
static class *led_class;

static int __init led_init(void)
{
    int ret;
    dev_t devno;
    struct cdev *cdev;
    struct dev *dev;

    /* 注册设备号 */
    ret = alloc_chrdev_region(&devno, 0, 1, "led");
    if (ret < 0) 
        return ret;

    /* 分配、初始化、注册cdev*/
    cdev = cdev_alloc();
    if (IS_ERR(cdev)) {
        ret = PTR_ERR(cdev);
        goto out_unregister_devno;
    }
    cdev_init(&cdev, &led_fops);
    cdev.owner = THIS_MODULE;
    ret = cdev_add(&cdev, devno, 1);    
    if (ret) 
        goto out_free_cdev;

    /* 创建设备类 */
    led_class = class_create(THIS_MODULE, "led_class");
    if (IS_ERR(led_class)) {
        ret = PTR_ERR(led_class);
        goto out_unregister_cdev;
    } 

    /* 创建设备节点 */
    dev = device_create(led_class, NULL, devno, NULL, "led");
    if (IS_ERR(dev)) {
        ret = PTR_ERR(dev);
        goto out_del_class;
    }

    return 0;

out_del_class:
    class_destroy(c78x_class); 
out_unregister_cdev:
    cdev_del(cdev);
out_free_cdev:
    kfree(cdev);
out_unregister_devno:
    unregister_chrdev_region(devno, 1);

    return ret;
}

module_init(led_init);
```

如果使用 device_register 进行创建, 对应 创建设备节点部分应该修改为:
```
/* 创建设备节点 */
    dev = kzalloc(sizeof(*dev), GFP_KERNEL);
    if (!dev) {
        ret = -ENOMEM;
        goto out_del_class;
    }

    dev->class = led_class;         // 关联设备类
    dev->parent = NULL;             
    dev->devt = devno;              // 关联设备号
    dev_set_drvdata(dev, NULL);     
    dev_set_name(dev, "led");       // 设置节点名字
    dev->release = device_create_release;

    ret = device_register(dev);
    if (ret) 
        goto out_put_dev;

    return 0;

out_put_dev:
    put_device(dev);
    kree(dev);
```

如果是使用 device_add() 进行创建, 进行如下修改:
```
/* 创建设备节点 */
    dev = kzalloc(sizeof(*dev), GFP_KERNEL);
    if (!dev) {
        ret = -ENOMEM;
        goto out_del_class;
    }

    dev->class = led_class;         // 关联设备类
    dev->parent = NULL;             
    dev->devt = devno;              // 关联设备号
    dev_set_drvdata(dev, NULL);     
    dev_set_name(dev, "led");       // 设置节点名字
    dev->release = device_create_release;

    device_initialize(dev);         // 多了这一步
    ret = device_add(dev);
    if (ret) 
        goto out_put_dev;

    return 0;

out_put_dev:
    put_device(dev);
    kree(dev);
```