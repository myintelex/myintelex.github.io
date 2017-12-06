---
title: Linux驱动开发（二）——字符设备驱动模型
date: 2016-12-20 01:08:01
categories: 驱动开发
tags: [驱动, file_operations, 驱动加载]
---
本篇对字符设备驱动模型进行总结。
<!--more-->
## 一、驱动结构
如上篇博客所说驱动开发关注的是两个重要的结构体：`file_operations`、`inode`，而对字符驱动设备来讲，我们关注的是 `inode` 中的 `cdev` 结构体和 `file_operations` 中的相关的操作函数。
`cdev` 结构体的定义如下：
````struct cdev {
    struct kobject kobj;         //内嵌的kobj对象
    struct module *owner;        //所属模块
    struct file_operations *ops; //文件操作结构体
    struct list_head list;
    dev_t dev;                   //设备号
    unsigned int count;
}````

其中的 `dev_t` 成员定义了设备号，我们可以使用相应的宏从中获取主次设备号，或者使用相应的设备号生成 `dev_t`：
````c 
MAJOR(dev_t dev);  //获取主设备号
MINOR(dev_t dev);  //获取次设备号
MKDEV(int major, int minor) //生成 dev_t
````

操作 `cdev` 结构体的函数如下所示：
````c
void cdev_init(struct cdev *, struct file_operations *);  //初始化 cdev 
struct cdev *cdev_alloc(void);                            //动态申请 cdev 内存
void cdev_put(struct cdev *p);                            //释放 cdev 内存
int cdev_add(struct cdev *, dev_t, unsigned);             //添加一个 cdev 字符:设备注册
int cdev_del(struct cdev *)；                             //删除一个 cdev 字符:设备注销
````

在设备注册和注销前，还需要向系统申请注册和释放设备号：
````c
int register_chrdev_region(dev_t from, unsigned count, const char *name); //已知起始设备号注册设备
int alloc_chrdev_region(dev_t *dev, unsigned baseminor, unsigned count, 
                        const char *name);                                 //动态申请设备号
int unregister_chrdev_region(dev_t form, unsigned count);                  //注销设备号
````

## 二、加载和卸载

字符设备驱动模块加载函数中应实现设备号的申请和 cdev 的注册，在卸载函数中应实现设备号的释放和 cdev 注销。相应的代码如下：
````C
//设备结构体，通常把该设备涉及到的cdev、私有数据及信号量等信息全包含进去。
struct xxx_dev_t {
    struct cdev cdev;
    ...
} xxx_dev;  
//设备驱动模块加载函数
static int __init xxx_init(void) {
    ...
    cdev_init(&xxx_dev.cdev, &xxx_fops); //初始化cdev
    xxx_dev.cdev.owner = THIS_MODULE;
    if (xxx_major) {
        register_chrdev_region(xxx_dev_no, 1, DEV_NAME);
    } else {
        alloc_chrdev_region(&xxx_dev_no, 0, 1, DEV_NAME);
    }  //获取字符设备号
    ret = cdev_add(&xxx_dev.cdev, xxx_dev_no, 1); //注册设备
} 
//设备驱动模块卸载函数
static void __exit xxx_exit(void) {
    unregister_chrdev_region(xxx_dev_no, 1); //释放占用的设备号
    cdev_del(&xxx_dev.cdev); //注销设备
}````

## 三、`file_operations` 结构体函数
字符设备驱动通常要实现字符设备 `file_operations` 结构体中的读写函数，相应的结构体模板是：
````c
struct file_operations xxx_fops = {
    .owner = THIS_MODULE;
    .read = xxx_read;
    .write = xxx_write;
    .ioctl = xxx_ioctl;
    ...
};````

我们需要将我们实现的操作函数在 `file_operations` 结构体中指定其实现函数。相应的实现函数模板是：
````c
//打开设备
int xxx_open(struct inode *inode, struct file *filp) {
    return 0;
}
//释放设备文件
int xxx_release(struct inode *inode, struct file *filp) {
    return 0;
}
//读设备
ssize_t xxx_read(struct file *filp, char __user *buf，size_t count, loff_t *f_pos) {
    ...
    copy_to_user(buf, ..., ...);
}
//写设备
ssize_t xxx_write(struct file *filp, char __user *buf，size_t count, loff_t *f_pos) {
    ...
    copy_from_user(buf, ..., ...);
}
//文件定位
static loff_t xxx_llseek(struct file *filp, loff_t offset, int orig) {
    ...
}
//设备控制
ssize_t xxx_ioctl(struct inode *inode, struct file *filp, unsigned int cmd, unsigned long arg) {
    ...
    switch (cmd) {
    case xxx_CMD1:
        ...
        break;
    case xxx_CMD1:
        ...
        break;
    default:
        return -ENOTTY;
    }
    return 0;
}````

分别介绍写通常要使用的实现函数：
### 3.1 打开函数 
`int xxx_open(struct inode *inode, struct file *filp)`
  打开函数中给驱动初始化，通常我们应实现以下工作：
   * 检查设备的错误
   * 如果设备初次打开，对其初始化
   * 如果必要，更新操作函数指针
   * 分配私有数据指针
  `inode` 参数指的是我们打开设备的文件结构体，其中包含了我们的字符设备cdev结构体指针。如上文所说，我们通常把该设备涉及到的cdev、私有数据及信号量等信息全包含进设备结构体中。这里有个技巧可以从 `inode` 指针中获取我们的设备结构体，使用宏 `container_of`，使用该宏可以从结构体成员指针找到对应的结构体指针:
  `container_of(pointer, container_type, container_field)`
  其中，
  `pointer` 是我们已知的指针，这里我们可以知道设备结构体中的 `cdev` 成员的指针为 `inode->i_cdev`；
  `container_type` 是我们想要获取的结构体类型，我们这里想要获取的结构体是自定义的设备结构体 `xxx_dev_t`
  `container_field` 是 `pointer` 的类型，这里 `cdev` 的类型是 `cdev`。
  这样，该宏就返回了我们想要的自定义的设备结构体指针，使用该指针可以很方便的找到我们的私有数据指针等和设备相关的信息。

### 3.2 关闭函数 
`int xxx_release(struct inode *inode, struct file *filp)`
   关闭函数和打开函数相反，完成了以下的工作：
   * 释放由 `open` 分配的，保存在 `filp->private_data` 中的所有数据。
   * 在最后一次关闭操作的时候关闭设备。

### 3.3 读写函数
  在设备读写函数中，`filp` 是文件结构体指针，`buf` 是用户空间内存地址，`count` 是要读写的字节数，`f_ops` 是读写的位置偏移量。
  因为内核空间和用户空间不能直接互相访问，所以需要在读写函数中实现用户空间到内核空间的复制。使用下述两个函数：
  ````c
  unsigned long copy_from_user(void *to, const void __user *from, unsigned long count);
  unsigned long copy_to_user(void __user *to, const void *from, unsigned long count);
  ````
  上述函数返回不能复制的字节数，如果完全复制成功，返回0。 `__user` 是一个宏，表明后面的指针指向用户空间。

### 3.4 ioctl函数
  ioctl 是设备驱动程序中对设备的I/O通道进行管理的函数。所谓对I/O通道进行管理，就是对设备的一些特性进行控制，例如串口的传输波特率、马达的转速等等。在 `xxx_ioctl` 函数中的 `cmd` 参数我们可以自己来直接指定，它可能仅仅是一个整数集，但 Linux 中的 ioctl 命令号都是有特定含义的，因此通常我们不么做。Linux 建议以 `设备类型/幻数（8bit）+序列号（8bit）+方向（2bit）+数据尺寸（13/14bit）` 的方式来定义 `cmd`。
  Linux 内核提供了相应的宏来自动生成 ioctl 命令号： 
  ````C
  _IO(type,nr) //没有参数的命令
  _IOR(type,nr,size) //该命令是从驱动读取数据
  _IOW(type,nr,size) //该命令是从驱动写入数据
  _IOWR(type,nr,size) //双向数据传输````
  上面的命令已经定义了方向，我们要传的是幻数(type)、序号(nr)和大小(size)。在这里szie的参数只需要填参数的类型，如int，上面的命令就会帮你检测类型的正确然后赋值 sizeof(int)。 
  有生成cmd的命令就必有拆分cmd的命令：
  ````C
  _IOC_DIR(cmd) //从命令中提取方向
  _IOC_TYPE(cmd) //从命令中提取幻数
  _IOC_NR(cmd) //从命令中提取序数
  _IOC_SIZE(cmd) //从命令中提取数据大小````

### 3.5 定位函数
`static loff_t xxx_llseek(struct file *filp, loff_t offset, int orig)`
通过对 `orig` 的判断可以将起始地址设为文件开头（SEEK_SET， 0）、当前位置（SEELK_CUL，1）和文件尾（SEEK_END，2）。
