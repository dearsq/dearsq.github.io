title: Linux Input 子系统分析
date: 2018-11-08 21:00:00
tags: Linux

---

[TOC]

## 框架结构
Input 子系统从上到下分为三层实现, 分别为 事件处理层 EventHandler / 核心层 InputCore / 驱动层 Driver
![](http://my.csdn.net/uploads/201207/29/1343543828_9978.png)

**Driver 层**: 硬件设备读写访问, 中断设置 , 将硬件产生的事件转换为核心层定义的规范提交给 EventHandler.
**InputCore 层**: 为设备驱动层提供了规范和接口。
设备驱动层只要关心如何驱动硬件并获得硬件数据（例如按下的按键数据），然后调用核心层提供的接口，核心层会自动把数据提交给事件处理层。 
**EventHandler 层**: 用户编程的接口（设备节点），并处理驱动层提交的数据处理。

## 驱动层流程
驱动开发的工作:
* 设置 input 设备支持的事件类型 (EV_SYN / EV_KEY 等)
* 注册中断处理函数
* 输入设备注册到输入子系统中

整体的流程是:
1. 注册初始化, button_init 将被用于 insmod 或者 内核引导过程中 的调用
主要负责硬件设备资源 ( 内存 / IO内存 / 中断 / DMA ) 的获取.
```
module_init(button_init)
```

2. 在 button_init 中获取中断, 注册中断 
当有键被按下 / 松开时 , 调用 button_interrupt 中断处理函数获取按键值 BUTTON_PORT
```
request_irq(BUTTON_IRQ, button_interrupt, 0, "button", NULL)
```
在 `button_interrupt` 中完成按键事件的上报需要两步: 1. `input_report_key` 2.`input_sync` .
这里因为只有一个按键事件, 所以体现不出 input_sync 的作用.
但是如果是 TP 事件, 在上报了 x y 坐标后, 需要 input_sync 将 x 和 y 完整地传给 input 子系统.
```
static irqreturn_t button_interrupt(int irq, void *dummy)
{
	input_report_key(button_dev, BTN_0, inb(BUTTON_PORT) & 1);
	input_sync(button_dev);
	return IRQ_HANDLED;
}
```

3. 注册input设备 
由 InputCore (input.c) 中的 API 分配一个输入设备
```
static struct input_dev *button_dev;
button_dev=input_allocate_device();
```

4. 设置事件产生时候的事件类型
`evbit` 事件产生时的事件类型(EV_KEY)
`keybit`事件上报的按键值
```
button_dev->evbit[0]= BIT(EV_KEY);
button_dev->keybit[LONG(BTN_0)]= BIT(BTN_0); //#defineLONG(x) ((x)/BITS_PER_LONG) //返回位x的索引
```

5. 注册为输入设备
```
input_register_device(button_dev);
```

## Demo
Demo 是只有一个按键的设备. 当按压或者释放按键的时候, 会发出 BUTTON_IRQ 中断.
```
#include <linux/input.h>
#include <linux/module.h>
#include <linux/init.h>

#include <asm/irq.h>
#include <asm/io.h>

static struct input_dev *button_dev;

static irqreturn_t button_interrupt(int irq, void *dummy)
{
	input_report_key(button_dev, BTN_0, inb(BUTTON_PORT) & 1);
	input_sync(button_dev);
	return IRQ_HANDLED;
}

static int __init button_init(void)
{
	int error;
	//注册中断
	if (request_irq(BUTTON_IRQ, button_interrupt, 0, "button", NULL)) {
                printk(KERN_ERR "button.c: Can't allocate irq %d\n", button_irq);
                return -EBUSY;
        }
		
	//
	button_dev = input_allocate_device();
	if (!button_dev) {
		printk(KERN_ERR "button.c: Not enough memory\n");
		error = -ENOMEM;
		goto err_free_irq;
	}

	button_dev->evbit[0] = BIT_MASK(EV_KEY);
	button_dev->keybit[BIT_WORD(BTN_0)] = BIT_MASK(BTN_0);
	
	//注册input设备
	error = input_register_device(button_dev);
	if (error) {
		printk(KERN_ERR "button.c: Failed to register device\n");
		goto err_free_dev;
	}

	return 0;

 err_free_dev:
	input_free_device(button_dev);
 err_free_irq:
	free_irq(BUTTON_IRQ, button_interrupt);
	return error;
}

static void __exit button_exit(void)
{
        input_unregister_device(button_dev);
	free_irq(BUTTON_IRQ, button_interrupt);
}

module_init(button_init);
module_exit(button_exit);
```

## 深入分析
### Input Initial
1. InputCore 初始化 
2. InputHandler 初始化 
3. InputDevice 初始化 
4. Core 将 Handler 和 Device 进行关联
#### 1. InputCore 初始化 
```
static int __init input_init(void)
{
	int err;
	//1. 在设备模型 /sys/class 目录注册设备类
	err = class_register(&input_class);
	//2. 在 /proc/bus/input 目录产生设备信息
	err = input_proc_init();
	//3. 字符设备驱动框架注册 input 子系统的接口操作集合
	err = register_chrdev_region(MKDEV(INPUT_MAJOR, 0),
				     INPUT_MAX_CHAR_DEVICES, "input");
	...
}
```
#### 2. InputHandler 初始化
拿 TP 的 event-handler 为例 (evdev.c)
```
static const struct input_device_id evdev_ids[] = { //设备 ID
	{ .driver_info = 1 },	/* Matches all devices */
	{ },			/* Terminating zero entry */
};
MODULE_DEVICE_TABLE(input, evdev_ids);
static struct input_handler evdev_handler = {
	.event		= evdev_event,	 
	.events		= evdev_events,  // 消息处理接口
	.connect	= evdev_connect, // InputCore 匹配到 device 后调用本接口 , 后面会详细分析
	.disconnect	= evdev_disconnect,
	.legacy_minors	= true, 
	.minor		= EVDEV_MINOR_BASE, //64 event-handler 处理的次设备号的起始点
	.name		= "evdev",          //handler 的名称
	.id_table	= evdev_ids,
};

static int __init evdev_init(void)
{
	// input-core 提供的接口, 供 input-handler 注册自己
	return input_register_handler(&evdev_handler);
}
```
```
input_register_handler
    INIT_LIST_HEAD
	list_add_tail//加入input_handler_list全局链表
	//遍历input_dev_list全局设备链表,寻找该handler支持的设备
	list_for_each_entry(dev, &input_dev_list, node)input_attach_handler(dev, handler); 
```

`evdev_connect` 的调用流程, 关键是注册了 fops.
```
evdev_connect
    input_get_new_minor //获取 minor 号
	device_initialize
    input_register_handle //注册新的 input handle
	cdev_init(&evdev->cdev, &evdev_fops); //很关键, 注册fops, fops我们在后面单独说
	cdev_add
	device_add
```

fops 如下, 包括了
```
static const struct file_operations evdev_fops = {
	.owner		= THIS_MODULE,
	.read		= evdev_read,
	.write		= evdev_write,
	.poll		= evdev_poll,
	.open		= evdev_open,
	.release	= evdev_release,
	.unlocked_ioctl	= evdev_ioctl,
#ifdef CONFIG_COMPAT
	.compat_ioctl	= evdev_ioctl_compat,
#endif
	.fasync		= evdev_fasync,
	.flush		= evdev_flush,
	.llseek		= no_llseek,
};
```
#### 3. InputDevice 初始化
这里因设备而异. 比如拿汇顶的 TouchScreen IC GT911 来举例.
```
tpd_driver_init
    tpd_get_dts_info
	tpd_driver_add(&tpd_device_driver) < 0)
	
tpd_probe
    input_register_device
```
`tpd_driver_add(&tpd_device_driver) < 0)` 
将 tpd_device_driver 添加到 tpd_driver_list 链表中, 关键是`tpd_driver_list[i].tpd_local_init = tpd_drv->tpd_local_init;`
在 tp device 和 driver match 的时候, 会执行 tpd_probe , 通过 tpd_driver_list 中的 tpd_local_init 找到初始化过程中 device需要进行的动作 (初始化寄存器等)

#### 4. InputCore 和 InputHander 关联匹配
在 input_register_handler 和 input_register_device 最后都会使用 input_attach_handler 接口来匹配输入设备和对应的事件处理者。
```
static int input_attach_handler(struct input_dev *dev, struct input_handler *handler)
{
	const struct input_device_id *id;
	int error;
	//匹配 handler 和 device 的 ID, event handler 默认处理所有事件类型的设备
	id = input_match_device(handler, dev);
	//匹配成功后调用 handler 的 connect 接口 (evdev_connect)
	error = handler->connect(handler, dev, id);
}
```
