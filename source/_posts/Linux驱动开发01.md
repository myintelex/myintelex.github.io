---
title: Linux驱动开发（一）——驱动开发基础
date: 2016-12-20 00:08:01
categories: 驱动开发
tags: [驱动, 内核, 设备文件系统, udev]
---

从本篇博客开始，尝试给自己掌握的 Linux 设备驱动开发进行一个总结，这并不是一个很简单的工作，因为自己的确目前对 Linux 驱动开发也只是一知半解。尽自己所能吧。今天是第一天，先来谈谈 Linux 驱动开发的基础知识。
<!--more-->
## 一、设备驱动程序简介
设备驱动程序是十分重要的，它是连接硬件与软件的桥梁。无操作系统的设备需要单独开发设备驱动，并以相应的驱动模块存在，这样可以更好的划分应用工程师的工作。而在有操作系统的设备上，驱动则是**连接硬件和内核的桥梁**。
Linux 设备驱大致可以分为三类： 
  **字符设备** 可以顺序访问的设备，通常字符设备至少要实现 `open()`、`close()`、`read()`、`write()` 等系统调用。
  **块设备** 块设备和字符设备类似，他们的区别是块设备上可以容纳文件系统，块设备和字符设备在内核内部管理数据的方式不同。
  **网络设备** 网络设备和字符以及块设备完全不同，它是围绕数据包的传输和接收来设计的。
所有的块设备和字符设备均被映射到 LInux 的文件系统中去。

## 二、Linux 内核简介
### 2.1 内核功能的划分
根据内核完成任务的不同，可以将内核功能划分为以下几部分：
* **进程管理**
  进程管理功能负责创建和销毁进程，并处理他们和外部世界的连接，不同进程的通信，CPU调度等功能。
* **内存管理**
  内存管理的策略是一个决定系统性能的关键因素。
* **文件系统**
  Unix 的每个对象几乎都可以当作文件来看待，Unix 没有在结构的硬件上构造文件系统，而是抽象在整个系统中广泛使用。
* **设备控制**
  内核必须为系统中的每件外设嵌入相关的驱动程序。
* **网络功能**
  内核负责在应用程序和网络接口之间传递数据包，路由和地址解析功能也是内核处理的工作。

### 2.2 内核的源代码

针对上面的相应功能，Linux内核源码的各个目录大致与此相对应，其组成如下： 
 * **arch** 目录包括了所有和体系结构相关的核心代码。它下面的每一个子目录都代表一种Linux支持的体系结构。 
 * **include** 目录包括编译核心所需要的大部分头文件。 
 * **init** 目录包含核心的初始化代码（不是系统的引导代码），有main.c和Version.c两个文件。
 * **mm** 目录包含了所有的内存管理代码。与具体硬件体系结构相关的内存管理代码位于arch/*/mm目录下。 
 * **drivers** 目录中是系统中所有的设备驱动程序。
 * **ipc** 目录包含了核心进程间的通信代码。 
 * **modules** 目录存放了已建好的、可动态加载的模块。 
 * **fs** 目录存放Linux支持的文件系统代码。
 * **Kernel** 内核管理的核心代码放在这里，如进程调度，定时器等。
 * **net** 目录里是核心的网络部分代码，其每个子目录对应于网络的一个方面。 
 * **lib** 目录包含了核心的库代码。 
 * **scripts** 目录包含用于配置核心的脚本文件。 
 * **documentation** 目录下是一些文档，是对每个目录作用的具体说明。 

内核源码的编译系统由三个部分组成：分散在源码各个目录的 Makefile 文件，内核的配置文件，以及我们的配置工具。具体的步骤为：
 * 首先使用 menuconfig 等配置工具配置内核。
 * 然后进行相应的编译工作

### 2.3 内核的引导过程
整体的 Linux 系统的开机过程如下：
加载BIOS -> 读取MBR -> Boot Loader -> 用户层init依据inittab文件来设定运行等级 -> init进程执行rc.sysinit -> 启动内核模块 ->执行不同运行级别的脚本程序 -> 执行/etc/rc.d/rc.local -> 执行/bin/login程序，进入登录状态
而详细的内核启动流程为：
1. 实模式的入口函数 `_start()`：在 header.S 中，这里会进入众所周知的 main 函数，它拷贝 bootloader 的各个参数，执行基本硬件设置，解析命令行参数。
2. 保护模式的入口函数 `startup_32()`：在 compressed/header_32.S 中，这里会解压 bzImage 内核映像，加载 vmlinux 内核文件。
3. 内核入口函数 `startup_32()`：在 kernel/header_32.S 中，这就是所谓的进程0，它会进入体系结构无关的 `start_kernel()` 函数，即众所周知的 Linux 内核启动函数。`start_kernel()` 会做大量的内核初始化操作，解析内核启动的命令行参数，并启动一个内核线程来完成内核模块初始化的过程，然后进入空闲循环。
4. 内核模块初始化的入口函数 `kernel_init()` ：在 init/main.c 中，这里会启动内核模块、创建基于内存的 rootfs、加载 initramfs 文件或cpio-initrd，并启动一个内核线程来运行其中的 init 脚本，完成真正根文件系统的挂载。
5. 根文件系统挂载脚本`/init`：这里会挂载根文件系统、运行 /sbin/init，从而启动众所周知的进程1。
6. init进程的系统初始化过程：执行相关脚本，以完成系统初始化，如设置键盘、字体，装载模块，设置网络等，最后运行登录程序，出现登录界面。

## 三、Linux 内核编程
### 3.1 Linux 内核模块的构成
一个 Linux 大致由下面几个部分构成：
 * 模块加载函数
   Linux 模块加载函数一般以 `__init` 标识声明，并以 `module_init(函数名)` 的形式被指定，典型的模块加载函数如下所示：
   ````c
   static int __init initialization_function(void) {
     /*初始化代码*/
   }
   module_init(initialization_function);````
   `module_init` 成功返回0，失败会返回相应的错误编码。

 * 模块卸载函数
   Linux 模块卸载函数一般以 `__exit` 标识声明，并以 `module_exit(函数名)` 的形式被指定，典型的模块卸载函数如下所示：
   ````c
   static int __exit cleanup_function(void) {
     /*初始化代码*/
   }
   module_exit(cleanup_function);````
   `module_exit` 不返回任何值。

 * 模块参数
   我们可以用 `moudule_param(参数名, 参数类型, 参数读写权限)` 为模块定义一个参数。
   在装载模块的时候，我们可以使用 `insmode 模块名 参数名=参数值` 的方式为模块传递参数。

 * 模块导出符号 
   模块可以使用相应的宏导出符号到内核符号表。
   ````c
   EXPORT_SYMBOL(符号名);
   EXPORT_SYMBOL_GPL(符号名);````
   导出的符号可以被其他模块使用。

 * 模块信息声明
   我们可以使用相应的宏来声明模块的作者等信息：
   ````c
   MODULE_AUTHOR(author);
   MODULE_DESCRIPTION(description);
   MODULE_VERSION(version);
   MODULE_DEVICE_TABLE(table_info);
   MODULE_ALIAS(alternate_name);````
   `MODULE_DEVICE_TABLE` 是用来为 USB PCI 等设备驱动声明模块支持设备的。

 * 模块许可证声明
   大多数情况下，我们遵循 GPL兼容许可证，使用 `MODULE_LICENSE("Dual BSD/GPL")` 来声明模块采用 BSD/GPL 双 LICENSE。

## 四、Linux文件系统
字符设备和块设备都良好的体现了 UNIX **一切皆文件** 的设计思想。我们写的所有驱动最终都是通过操作系统的文件操作来调用访问，所以我们必须了解 Linux 的文件系统组成，和设备驱动与文件系统的关系。

### 4.1 Linxu 文件系统结构
1. `/` 
文件系统的入口，最高一级目录；

2. `/bin`
基础系统所需要的命令位于此目录，是最小系统所需要的命令，如：ls, cp, mkdir等。
这个目录中的文件都是可执行的，一般的用户都可以使用。

3. `/sbin`
包含系统的命令，如 modprobe、hwclock、ifconfig 等，大多是涉及系统管理的命令。
这个目录中的文件都是可执行的。

3. `/boot` 
包含Linux内核及系统引导程序所需要的文件，比如 vmlinuz initrd.img 文件都位于这个目录中。
在一般情况下，GRUB或LILO系统引导管理器也位于这个目录；

4. `/dev` 
设备文件存储目录，应用程序可以通过对这些文件的读写和控制来访问实际的设备。

5. `/etc` 
存放系统程序或者一般工具的配置文件。
如 /etc/init.d 这个目录是用来存放系统或服务器以System V模式启动的脚本，这在以System V模式启动或初始化的系统中常见。
如apache2的/etc/init.d apache2 start|stop|restart MySQL为/etc/init.d mysql start|stop|restart 6. /home 普通用户默认存放目录 Linux 是多用户环境，所以每一个用户都有一个只有自己可以访问的目录（当然管理员也可以访问）。它们以 /home/username 的方式存在。这个目录也保存一些应用对于这个用户的配置，比如 IRC, X 等。

7. `/lib` 
库文件存放目录这里包含了系统程序所需要的所有共享库文件，类似于 Windows 的共享库 DLL 文件。

8. `/lost+found` 
在ext2或ext3文件系统中，当系统意外崩溃或机器意外关机，而产生一些文件碎片放在这里。当系统启动的过程中fsck工具会检查这里，并修复已经损坏的文件系统。 有时系统发生问题，有很多的文件被移到这个目录中，可能会用手工的方式来修复，或移到文件到原来的位置上。
Linux 应该正确的关机。但有时你的系统也可能崩溃掉或突然断电使系统意外关机。那么启动的时候 fsck 将会进行长时间的文件系统检查。Fsck 会检测并试图恢复所发现的不正确的文件。被恢复的文件会放置在这个目录中。所恢复的文件也许并不完整或并不合理，但毕竟提供了一些恢复数据的机会。


9. `/media` 
即插即用型存储设备的挂载点自动在这个目录下创建，比如USB盘系统自动挂载后，会在这个目录下产生一个目录 ；CDROM/DVD自动挂载后，也会在这个目录中创建一个目录，类似cdrom 的目录。这个只有在最新的发行套件上才有. 

10. `/mnt` 
这个目录一般是用于存放挂载储存设备的挂载目录的，比如有cdrom 等目录。有时我们可以把让系统开机自动挂载文件系统，把挂载点放在这里也是可以的。比如光驱可以挂载到/mnt/cdrom 。在这里你可以加载你的文件系统或设备。加载是使一个文件系统对于系统可用的过程。在加载后你的文件可以在加载目录下访问。这个目录通常包含加载目录或用于加载软驱和光驱的子目录。

11. `/opt` 
表示的是可选择的意思，有些软件包也会被安装在这里，也就是自定义软件包，比如在Fedora Core 5.0中，OpenOffice就是安装在这里。有些我们自己编译的软件包，就可以安装在这个目录中；通过源码包安装的软件，可以通过 ./configure --prefix=/opt/，将软件安装到opt目录。

12. `/proc` 
操作系统运行时，进程（正在运行中的程序）信息及内核信息（比如cpu、硬盘分区、内存信息等）存放在这里。/proc目录是伪装的文件系统proc的挂载目录，proc并不是真正的文件系统。这是系统中极为特殊的一个目录，实际上任何分区上都不存在这个目录。它实际是个实时的、驻留在内存中的文件系统。


13. `/root` 
Linux超级权限用户root的家目录；

15. `/tmp` 
临时文件目录，有时用户运行程序的时候，会产生临时文件。 /tmp就用来存放临时文件的。/var/tmp目录和这个目录相似。
许多程序在这里建立lock文件和存储临时数据。有些系统会在启动或关机时清空此目录。

16. `/usr` 
这个是系统存放程序的目录，比如命令、帮助文件等。
这个目录下有很多的文件和目录。
当我们安装一个Linux发行版官方提供的软件包时，大多安装在这里。
如果有涉及服务器配置文件的，会把配置文件安装在/etc目录中。

17. `/var`
var 表示变化的意思，这里通常会存储日志文件。

18. `/sys`
Linxu 2.6 内核所支持的 sysfs 文件系统被映射到这个目录。Linux 设备驱动模型中的总线、驱动和设备都可以在这个文件系统中找到相应的节点。

### 4.2 Linux 文件系统和设备驱动的关系
Linux 系统中应用程序、虚拟文件系统（VFS）、和磁盘文件以及一般的设备文件与设备驱动程序之间的关系如下图所示：
![Linux 文件系统和设备驱动的关系](http://ogf054qp1.bkt.clouddn.com/linux%E6%96%87%E4%BB%B6%E7%B3%BB%E7%BB%9F%E5%92%8C%E8%AE%BE%E5%A4%87%E9%A9%B1%E5%8A%A8%E7%9A%84%E5%85%B3%E7%B3%BB.png)

应用程序使用系统调用调用 VFS 中的文件，而 VFS 是通过 file_operations 结构体的成员函数来调用磁盘文件系统中的文件和普通文件的。
字符设备驱动直接提供了相应的 file_operations 成员函数，而块设备因为上面承载的文件系统中会实现相应的 file_operations 成员函数，所以块设备驱动会将对磁盘文件的访问最终转换为对磁盘上相应柱面和扇面的访问。

而与设备驱动开发紧密相关的两个对象是 file 结构体和 inode 结构体。
file 结构体的定义是：
````C
struct file {
  union {
    struct list_head             fu_list;      //文件对象链表指针linux/include/linux/list.h
    struct rcu_head              fu_rcuhead;   //RCU(Read-Copy Update)是Linux 2.6内核中新的锁机制
  } f_u;
  struct path                    f_path;       //包含dentry和mnt两个成员，用于确定文件路径
#define f_dentry                 f_path.dentry //f_path的成员之一，当前文件的dentry结构
#define f_vfsmnt                 f_path.mnt    //表示当前文件所在文件系统的挂载根目录
  const struct file_operations  *f_op;         //与该文件相关联的操作函数
  atomic_t                       f_count;      //文件的引用计数(有多少进程打开该文件)
  unsigned int                   f_flags;      //对应于open时指定的flag
  mode_t                         f_mode;       //读写模式：open的mod_t mode参数
  off_t                          f_pos;        //该文件在当前进程中的文件偏移量
  struct fown_struct             f_owner;      //该结构的作用是通过信号进行I/O时间通知的数据。
  unsigned int                   f_uid, f_gid; //文件所有者id，所有者组id
  struct file_ra_state           f_ra;         //在linux/include/linux/fs.h中定义，文件预读相关
　unsigned long                  f_version;
#ifdef CONFIG_SECURITY
  void                          *f_security;
#endif
　　
  void                          *private_data;  //私有数据指针
#ifdef CONFIG_EPOLL
  struct list_head               f_ep_links;
  spinlock_t                     f_ep_lock;
#endif
　struct address_space *f_mapping;
};
````
其中设备驱动最为关心的几个内容是：读写模式 `f_mode`，标志 `f_flags`，而 `private_data` 私有数据指针通常被用来指向设备驱动自定义用于描述设备的结构体。

inode 结构体的具体定义如下：
````C
struct inode {
  ...
  umode_t imode;                //inode 的权限
  uid_t i_uid;                  //inode 拥有者的id
  gid_t i_gid;                  //inode 所属群组的id
  dev_t i_rdev;                 //如果是设备文件，这里代表设备的设备号
  loff_t i_size;                //inode 所代表的文件的大小

  struct timespec i_atime;      //inode 最近一次的存取时间
  struct timespec i_mtime;      //inode 最近一次的修改时间
  struct timespec i_ctime;      //inode 的产生时间

  unsigned long i_blksize;      //inode 在做I/O时的区块大小
  unsigned long i_blocks;       //inode 所使用的block数

  struct block_device *i_bdev;  //如果是块设备，对应block_device结构体指针
  struct cdev *i_cdev;          //如果是字符设备，对应cdev结构体指针
}

inode 中的 i_rdev 代表设备号，Linux 2.6 设备编号分为主设备号和次设备号，前者为 dev_t 的高12位，后者为低12位。可以使用下面的函数来获取相应的设备号：
````C
unsigned int iminor(struct inode *inode);
unsigned int imajor(struct inode *inode);````

设备号是与驱动对应的概念，同一类设备一般使用相同的主设备号，因为同一驱动往往可以支持多个同类的设备。

### 4.3 Linux 设备文件系统
Linux 2.6 引入了 `sysfs` 文件系统，该文件系统是一个虚拟的文件系统。它可以产生一个包括所有系统硬件层级的视图。
`sysfs` 把连接在系统上的设备和总线组织成了一分级的文件。它的顶级目录如下：
 * **block** 包含所有的块设备
 * **device** 包含所有的设备
 * **bus** 包含所有的总线类型
 * **drivers** 包含内核中所有已注册的设备驱动程序
 * **class** 包含系统中的设备类型

在 Linux 内核中，分别使用 `bus_type` `device_driver` `device` 来描述总线/驱动/设备。后两者都必须依赖于一种总线，其相应的结构体中都含有一个 `bus_type` 指针。而驱动和设备是分开注册的，他们通过 `bus_type` 结构体中的 `match()` 成员函数进行配对。

Linux 2.6 使用了 udev 设备模型，udev 根据系统的硬件设备状态变化动态更新设备文件，进行设备的创建和删除等。因此，/dev目录就可以只包含系统中真正存在的文件了。 udev设备在设备被发现的时候加载驱动模块。udev 设计达到下面的目标：
 1. 在用户空间执行
 2. 动态的创建删除文件
 3. 不关心主次设备号
 4. 提供LSB标准名称
 5. 可以提供固定名称

udev 分为三个模块 namedev、libsysfs、udev。他们的工作过程是：
当内核检测到系统出现了新的设备，内核在 sysfs 文件系统中生成相应的记录，并导出一些设备特定的信息。udev 获取内核导出的信息，调用 namedev 给设备指定名称，调用 libsysfs 给设备指定主次设备号，并分析相应的信息来创建/dev中的设备文件。