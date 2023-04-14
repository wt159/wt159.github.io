---
title: Linux驱动开发之misc设备驱动
tags: linux 驱动开发 misc设备驱动
---

## 概述

misc设备驱动是Linux内核中的一种特殊类型的驱动程序，它用于管理那些没有与特定硬件设备相关联的设备。这些设备通常是由用户空间的应用程序创建和控制的，例如设备文件/dev/null和/dev/random。

miscdevice本质上也是字符设备，只是在miscdevice核心层的misc_init（）函数中，通过register_chrdev（MISC_MAJOR，"misc"，&misc_fops）注册了字符设备，而具体miscdevice实例调用misc_register（）的时候又自动完成了device_create（）、获取动态次设备号的动作。

miscdevice的主设备号是固定的，MISC_MAJOR定义为10，在Linux内核中，大概可以找到200多处使用miscdevice框架结构的驱动。

## 相关头文件、结构体、API

1.头文件：

- `<linux/miscdevice.h>`：定义了miscdevice结构体和相关的函数原型。

2.结构体：

- struct miscdevice：用于描述misc设备的属性和操作函数。

miscdevice结构体定义如下：

```c
struct miscdevice {
    int minor;                   // 设备的次设备号
    const char *name;           // 设备名称
    const struct file_operations *fops; // 设备的操作函数
    struct list_head list;      // 内核链表
    struct device *parent;      // 父设备指针
    struct device *this_device; // 设备指针
    const struct attribute_group **groups; // 属性组
    const char *nodename;       // 设备节点名称
    umode_t mode;               // 设备节点权限
};
```

3.API：

- misc_register()：用于将misc设备注册到系统中。

函数原型如下：

```c
int misc_register(struct miscdevice *misc);
```

- misc_deregister()：用于将misc设备从系统中注销。

函数原型如下：

```c
void misc_deregister(struct miscdevice *misc);
```

## 模板

```c
#include <linux/cdev.h>
#include <linux/ide.h>
#include <linux/init.h>
#include <linux/kernel.h>
#include <linux/miscdevice.h>
#include <linux/module.h>
#include <linux/of.h>
#include <linux/types.h>

struct xxx_dev_t {
    struct miscdevice miscdev;
};
static struct xxx_dev_t xxx_dev;

static int xxx_open(struct inode* inode, struct file* filp)
{
    filp->private_data = &xxx_dev;
    return 0;
}

static ssize_t xxx_read(struct file* filp, char __user* buf, size_t cnt, loff_t* offt)
{
    return 0;
}

static ssize_t xxx_write(struct file* filp, const char __user* buf, size_t cnt, loff_t* offt)
{
    return 0;
}

static long xxx_ioctl(struct file *filp, unsigned int cmd, unsigned long arg)
{
    return 0;
}

static int xxx_release(struct inode* inode, struct file* filp)
{
    return 0;
}

static struct file_operations xxx_fops = {
    .owner = THIS_MODULE,
    .open = xxx_open,
    .read = xxx_read,
    .write = xxx_write,
    .release = xxx_release,
    .unlocked_ioctl = xxx_ioctl,
};

static int __init xxx_init(void)
{
    int ret = 0;
    xxx_dev.miscdev.minor = MISC_DYNAMIC_MINOR;
    xxx_dev.miscdev.name = "xxx";
    xxx_dev.miscdev.fops = &xxx_fops;
    ret = misc_register(&xxx_dev.miscdev);
    return ret;
}

static void __exit xxx_exit(void)
{
    misc_deregister(&xxx_dev.miscdev);
}

module_init(xxx_init);
module_exit(xxx_exit);
MODULE_LICENSE("GPL");
MODULE_AUTHOR("author");
```
