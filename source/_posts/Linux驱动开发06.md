---
title: Linux驱动开发（六）——内存使用
date: 2016-12-20 06:08:08
categories: 驱动开发
tags: [驱动, 内存, IO, 硬件通信]
---
Linux 使用的是一个虚拟内存系统，意味着用户程序所使用的地址和硬件使用的物理地址是不等同的。本篇博客首先会简单描述相应的概念，然后介绍下内存存取的相应接口，最后讨论下和硬件IO相关的知识。
<!--more-->

## 一、Linux 内存管理
### 1. 地址类型
我们会听到很多种的地址类型，而每种类型都有自己的语义，相应的地址类型的解释如下：
* **用户虚拟地址** 
  在用户空间看到的常规地址，每个进程都有自己的虚拟地址空间。
* **物理地址**
  在处理器和系统内存之间使用。
* **总线地址**
  在外围总线和内存之间使用，通常和处理器使用的物理地址相同。
* **内核逻辑地址**
  内核的常规地址空间，映射了部分内存。逻辑地址与物理地址的不同仅仅是之间存在一个固定的偏移量。使用 `kmalloc` 获取的地址就是内核逻辑地址。
* **内核虚拟地址**
  内核虚拟地址类似于逻辑地址，但是虚拟地址不一定是一对一的映射关系。
### 2. 内存的使用
系统对内存的使用是基于页的，页是物理地址分成的离散单元。页的大小一般位 4096 个字节。内存的地址都被分为了页号和一个偏移量。
处理器一般会提供一个内存管理单元（MMU），MMU 提供了虚拟地址和物理地址的映射、内存访问权限保护、Cache 缓存控制等硬件支持。详细的 MMU 的工作模式不在这里详细描述。
对于一个Linux 进程来讲，4GB 的内存空间被分为了两个部分：用户空间（0~3GB/PAGE_OFFSET）和内核空间（3~4GB）。内核空间又被分为了物理地址映射区（896MB）、专用页面映射区（FIXADDR_START~FIXADDR_TOP）、系统保留映射区(FIXADDR_TOP~4GB)。
一般来讲，系统内存映射区最大位896MB，低于这个值的内存拥有逻辑地址，当大于这个值的时候，内核必须映射到高端页面映射区。一般来讲，内核数据都被放置在低端内存中，高端内存更趋向于为用户空间映射保留。

## 二、内存存取
在用户空间使用 `malloc()` 和 `free()` 函数进行内存的申请和释放，而内核空间往往使用下面的几个函数来进行内存的动态申请：

1. `kmalloc`
   ````C
   void *kmalloc(size_t size, init flags);````
   功能：从物理内存映射区域申请一块内存。
   参数：`size` 申请的块的大小；`flags` 分配标志，常用的标志有：
        `GFP_KERNEL` 在内核空间的进程中申请内存。
        `GFP_ATOMIC` 如果不存在空闲页，直接返回。
   使用 `kmalloc` 申请的内存应搭配使用 `kfree` 来释放。

2. `__get_free_pages()`
   ````C
   //申请
   get_zeroed_page(unsigned int flags);
   __get_free_page(unsigned int flags);
   __get_free_pages(unsigned int flags, unsigned int order);
   //释放
   void free_page(unsigned long addr);
   void free_pages(unsigned long addr, unsigned int order);````

   使用 GFP 等函数，申请的空间是以页为单位的，第一个函数返回一个指向新页的指针并将该页清零，第二个函数返回一个指向新页的指针但不清零，第三个函数返回一个指向多个页的首地址，分配的页数为2的 `order` 次幂。

3. `vmalloc()`
   ````C
   void *vmalloc(unsigned long size);
   void vfree(void *addr);````

   `vmalloc` 在虚拟内存空间给出一块连续的内存区，这个函数不能用于原子上下文中。

   和 `vmalloc` 类似，还有一对 `ioremap` `iounmap` 函数，用于映射I/O内存到虚拟空间，下面会使用到这个函数。 


4. `slab`
   针对返回使用的同一大小的内存块，内核提供了后备高速缓存。slab 算法就是针对上述特点设计的。
   ````C
   //创建 slab 缓存
   struct kmem_cache *kmen_cache_create(const char *name, size_t size,
            size_t align, unsigned long flags,
            void (*ctor)(void *, struct kmem_cache *, unsigned long),
            void (*dtor)(void *, struct kmem_cache *, unsigned long));
   //分配 slab 缓存
   void kmem_cache_alloc(struct kmem_cache *cachep, gfp_t flags);
   //释放 slab 缓存
   void kmem_cache_free(struct kmem_cache *cachep, void *objp);
   //收回 slab 缓存
   int kmem_cache_destroy(struct kmem_cache *cachep);````


## 二、硬件通信
设备通常会有相应的寄存器来控制相应的功能，这些寄存器可能存在在IO空间，也可能存在在内存空间。需要注意的是不管是IO端口还是IO内存都存在边界效应。
  
### 1. I/O端口的使用
1. I/O端口的分配
   ````C
   struct resource *request_region(unsigned long first, unsigned long n,
                                   const char *name);//申请操作的端口
   void release_region(unsigned long start, unsigned long n);//释放使用的端口
   int check_region(unsigned long first, unsigned long n);//检查端口是否可用
   ````
   申请端口成功返回非NULL值，失败返回NULL。
2. I/O端口的操作
   ````C
   //读写8位端口
   unsigned inb(unsigned port);
   void outb(unsigned char byte, unsigned port);
   //读写16位端口
   unsigned inw(unsigned port);
   void outw(unsigned short word, unsigned port);
   //读写32位端口
   unsigned inl(unsigned port);
   void outl(unsigned long byte, unsigned port);
   //读写一串字节
   void insb(unsigned port, void *addr, unsigned long count);
   void outsb(unsigned port, void *addr, unsigned long count);
   //读写一串字
   void insw(unsigned port, void *addr, unsigned long count);
   void outsw(unsigned port, void *addr, unsigned long count);
   //读写一串长字
   void insl(unsigned port, void *addr, unsigned long count);
   void outsl(unsigned port, void *addr, unsigned long count);````
   上述函数的 `port` 类型依赖于硬件平台。

### 2. I/O内存的使用
1. I/O内存的分配映射
   ````C
   struct resource *request_mem_region(unsigned long start, unsigned long len, 
                                       char *name);//分配I/O内存区域
   void release_mem_region(unsigned long start, unsigned long len);//释放I/O内存
   int check_mem_region(unsigned long start, unsigned long len);//检查是否可用
   ````
   进行IO内存的申请和释放后，并不是意味着得到的指针可以使用，应使用上面提到的 `ioremap` 函数来将相应的IO内存映射到虚拟空间内。

2. I/O内存的访问
   I/O内存访问使用下述函数，不再详细描述。
   ````C
   unsigned int ioread8(void *addr);
   unsigned int ioread16(void *addr);
   unsigned int ioread32(void *addr);
   void iowrite8(u8 value, void *addr);
   void iowrite16(u16 value, void *addr);
   void iowrite32(u32 value, void *addr);````


### 3. 映射I/O端口
有一对函数可以把I/O端口映射为I/O内存。
````C
void *ioport_map(unsigned long port, unsigned int count);
void ioport_unmap(void *addr);````

