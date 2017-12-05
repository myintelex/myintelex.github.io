---
title: Linux驱动开发（七）——设备驱动模型
date: 2016-12-20 06:08:09
categories: 驱动开发
tags: [驱动, 内存, IO, 硬件通信]
---

设备驱动模型提供了硬件的抽象，内核使用这样的抽象可以完成很多硬件重复的工作，这些抽象工作包括几个部分：
1. **电源管理**
   设备驱动模型使电源管理子系统可以以正确的顺序遍历系统上的设备。
2. **热插拔设备**
   设备驱动模型自动捕捉插拔信号，加载驱动程序，使内核容易和设备通信。
3. **与用户空间通信**
   用户空间程序通过 sysfs 虚拟文件系统访问设备的相关信息。通过对 sysfs 文件系统的操作，就能够控制设备，或者从设备中读取相应的信息。

<!--more-->
## 一、sysfs 文件系统
sysfs 文件系统是 Linux 文件系统中的一个。它的设计目的是：
 1. 显示设备驱动模型中相关数据结构之间的关系；
 2. 显示相应数据结构的信息。

sysfs 文件系统与驱动模型结构的对应关系如下表所示：

|类型	|所包含的内容	|对应内核数据结构|	对应/sys目录|
|-----|------------|--------------|-------------|
|设备(Devices)	|设备是此模型中最基本的类型，以设备本身的连接按层次组织|	struct device|	/sys/devices/*/*/.../|
|设备驱动(Device Drivers)|在一个系统中安装多个相同设备，只需要一份驱动程序的支持|struct device_driver|	/sys/bus/pci/drivers/*/|
|总线类型(Bus Types)|在整个总线级别对此总线上连接的所有设备进行管理|	struct bus_type|	/sys/bus/*/|
|设备类别(Device Classes)|	这是按照功能进行分类组织的设备层次树|struct class|	/sys/class/*/|

其中的 sysfs 中的目录主要有
├── block      //系统中的块设备目录
├── bus        //系统中的总线目录
├── class      //按照设备功能分类的设备模型
├── dev        //系统中的发现的设备
├── devices    //系统中所有设备的分层次表达
├── firmware   //系统加载固件机制接口
├── fs         //系统中的文件系统
├── kernel     //内核可调整模块
├── module     //系统中所有的模块信息
└── power      //系统的电源选项

### 1.1 kobject 结构体
sysfs 文件系统树形结构的每一个目录都与一个 `kobject` 对象相对应，其中包含了该目录的组织结构和名字等信息。它的功能是：提供引用计数、维持父子和平级目录结构。
````C
struct kobject {
        const char              *name;       //kobject的名称
        struct list_head        entry;       //连接下一个 kobject 结构
        struct kobject          *parent;     //指向父级 kobject
        struct kset             *kset;       //指向 kset 集合
        struct kobj_type        *ktype;      //指向 kobject 的类型描述符
        struct sysfs_dirent     *sd;         //对应 sysfs 的文件目录
        struct kref             kref;        //kobject 的引用计数
        unsigned int state_initialized:1;    //是否初始化
	    unsigned int state_in_sysfs:1;       //是否加入sysfs
        unsigned int state_add_uevent_sent:1;
        unsigned int state_remove_uevent_sent:1;
};````
`device`, `device_driver` 等各对象都是以 `kobject` 基础功能之上实现的。相应的 kobject 操作方法有：
````C
void kobject_init(struct kobject *kobj, struct kobj_type *ktype);  //初始化
static void kobject_init_internal(struct kobj_type *ktype); //初始化内部成员
struct kobject *kobject_get(struct kobject *kobj);  //引用计数增加
void kobject_put(struct kobject*kobj); //引用计数减少
int kobject_set_name(struct kobject *kobj, const char *fmt, ...); //设置名字
int kobject_rename(struct kobject *kobj, const char *new_name); //修改名字
````

其中的 `kobj_type` 是 kobject 的属性结构体，里面包含了 kobject 的属性（attribute）和属性操作方法（sysfs_ops）。相应的操作函数是：

### 1.2 kset 结构体
它用来对同类型对象提供一个包装集合，在内核数据结构上它也是由内嵌一个 kboject 实现，因而它同时也是一个 kobject (面向对象 OOP 概念中的继承关系) ，具有 kobject 的全部功能；
````C
struct kset {
        struct list_head list;
        spinlock_t list_lock;
        struct kobject kobj;
        struct kset_uevent_ops *uevent_ops;
};````

在 /sys 根目录之下的都是 kset，它们组织了 /sys 的顶层目录视图；在部分 kset 下有二级或更深层次的 kset；
每个 kset 目录下再包含着一个或多个 kobject，这表示一个集合所包含的 kobject 结构体；
在 kobject 下有属性(attrs)文件和属性组(attr_group)，属性组就是组织属性的一个目录，它们一起向用户层提供了表示和操作这个 kobject 的属性特征的接口；
在 kobject 下还有一些符号链接文件，指向其它的 kobject，这些符号链接文件用于组织上面所说的 device, driver, bus_type, class, module 之间的关系；
不同类型如设备类型的、设备驱动类型的 kobject 都有不同的属性，不同驱动程序支持的 sysfs 接口也有不同的属性文件；而相同类型的设备上有很多相同的属性文件。

## 二、设备驱动模型的三大组件
设备驱动模型有三个重要的组件：总线、设备、驱动。它们三者是紧密联系的。总线将设备和驱动绑定，在系统每注册一个设备的时候，都会寻找相应的驱动，相反的，当系统每注册一个驱动的时候都会寻找与之匹配的设备。
### 2.1 总线
总线的定义如下：
````C
struct bus_type {
    const char *name; /* 总线类型名 */
    struct bus_attribute *bus_attrs; /* 总线的属性 */
    struct device_attribute *dev_attrs; /* 设备属性,为每个加入总线的设备建立属性链表 */
    struct driver_attribute *drv_attrs; /* 驱动属性,为每个加入总线的驱动建立属性链表 */
    int (*match)(struct device *dev, struct device_driver *drv); /* 驱动与设备匹配函数 */
    int (*hotplug)(struct sevice *device, char **envp, int num_envp, 
                   char*buffer, int buffer_size)/*热插拔函数*/
    int (*uevent)(struct device *dev, struct kobj_uevent_env *env);
    int (*probe)(struct device *dev); /* */
    int (*remove)(struct device *dev); /* 设备移除调用操作 */
    void (*shutdown)(struct device *dev);
    int (*suspend)(struct device *dev, pm_message_t state);
    int (*resume)(struct device *dev);
    const struct dev_pm_ops *pm;
    struct subsys_private *p; /* 一个很重要的域，包含了device链表和drivers链表 */
};````
总线中间最重要的是两个方法：`match` 一个新设备或者驱动被添加到这个总线时，这个方法会被调用一次或多次，若指定的驱动程序能够处理指定的设备，则返回非零值。必须在总线层使用这个函数, 因为那里存在正确的逻辑，核心内核不知道如何为每个总线类型匹配设备和驱动程序。`hotplug`在为用户空间产生热插拔事件之前，这个方法允许总线添加环境变量（参数和 kset 的uevent方法相同）。

### 2.2 设备
设备在内核的定义如下：
````C
struct device {
    struct device *parent; /* 父设备，总线设备指定为NULL */
    struct kobject kobj;
    char bus_id[BUS_ID_SIZE];
    const char *init_name; /* 初始默认的设备名,但@device_add调用之后又重新设为NULL */
    struct device_type *type;
    struct mutex mutex; /* mutex to synchronize calls to its driver */
    struct bus_type *bus; /* type of bus device is on */
    struct device_driver *driver; /* which driver has allocated this device */
    void *platform_data; /* Platform specific data, device core doesn't touch it */
    void (*release)(struct device *dev); 
    ...
};````

### 2.3 设备驱动
设备驱动在内核中的表示如下：
````C
struct device_driver {
    const char *name; /* 驱动名称,在sysfs中以文件夹名出现 */
    struct bus_type *bus; /* 驱动关联的总线类型 */
    struct module *owner;
    const char *mod_name; /* used for built-in modules */
    bool suppress_bind_attrs; /* disables bind/unbind via sysfs */
    int (*probe) (struct device *dev);
    int (*remove) (struct device *dev);
    void (*shutdown) (struct device *dev);
    int (*suspend) (struct device *dev, pm_message_t state);
    int (*resume) (struct device *dev);
    struct driver_private *p;
    ...
};
struct driver_private { /* 定义device_driver中的私有数据类型 */
   struct kobject kobj; /* 内建kobject */
   struct klist klist_devices; /* 驱动关联的设备链表，一个驱动可以关联多个设备 */
   struct klist_node knode_bus;
   struct module_kobject *mkobj;
   struct device_driver *driver; /* 连接到的驱动链表 */
};````

