---
title: Linux学习总结（五）——高级I/O
date: 2016-12-16 14:54:55
categories: Linux学习记录
tags: [I/O, 阻塞式I/O, select, poll, epoll]
---

高级I/O包含了很多内容，本篇会首先解释下几个相关的概念：阻塞、非阻塞、同步、异步等的概念；接着介绍下 Linux I/O 操作的具体过程；最后讨论下多路复用、记录锁等几个相关函数。
<!--more-->

## 一、概念说明
Linux 的每个进程都是拥有自己的虚拟内存空间的，而一个进程的虚拟内存空间分为内核空间和用户空间两个部分，当进程的执行过程中期待的某种事情没有发生：请求系统资源失败、等待某种操作的完成、新数据尚未到达等。对于一次IO访问（以read举例），数据会先被拷贝到操作系统内核的缓冲区中，然后才会从操作系统内核的缓冲区拷贝到应用程序的地址空间。所以说，当一个read操作发生时，它会经历两个阶段：

  1. 等待数据准备 (Waiting for the data to be ready)
  2. 将数据从内核拷贝到进程中 (Copying the data from the kernel to the process)

正式因为这两个阶段，Linux系统产生了下面五种I/O:

### 1. 阻塞式I/O
 通常来说，从普通文件读数据，无论你是采用 `fscanf`、 `fgets` 也好，`read` 也好，一定会在有限的时间内返回。但是如果你从设备，比如终端（标准输入设备）读数据，只要没有遇到换行符(`\n`)，`read` 一定会**堵**在那而不返回。还有比如从网络读数据，如果网络一直没有数据到来，`read` 函数也会一直堵在那而不返回。
 `read` 的这种行为，称之为 block，一旦发生 block，本进程将会被操作系统投入睡眠，直到等待的事件发生了（比如有数据到来），进程才会被唤醒。
 系统调用 `write` 同样有可能被阻塞，比如向网络写入数据，如果对方一直不接收，本端的缓冲区一旦被写满，就会被阻塞。
 
### 2. 非阻塞式I/O
 当用户进程发出 `read` 操作时，如果 kernel 中的数据还没有准备好，那么它并不会 block 用户进程，而是立刻返回一个 error。从用户进程角度讲 ，它发起一个 `read` 操作后，并不需要等待，而是马上就得到了一个结果。用户进程判断结果是一个 error 时，它就知道数据还没有准备好，于是它可以再次发送 `read` 操作。一旦 kernel 中的数据准备好了，并且又再次收到了用户进程的 system call，那么它马上就将数据拷贝到了用户内存，然后返回。
 
 > 所以:
 非阻塞式I/O的特点是用户进程需要不断的主动询问 kernel 数据好了没有。
 阻塞非阻塞是文件本身的特性，不是系统调用read/write本身可以控制的。

### 3. I/O多路复用
 I/O多路复用（`IO multiplexing`）就是我们说的 `select`、 `poll`、 `epoll`，指的是单个 `process` 可以同时处理多个 IO 操作。它的基本原理就是 `select`、 `poll`、 `epoll` 会不断的轮询所负责的所有 I/O，当某个 I/O 有数据到达了，就通知用户进程。
 
 > 所以:
 I/O 多路复用的特点是通过一种机制一个进程能同时等待多个文件描述符，而这些文件描述符（套接字描述符）其中的任意一个进入读就绪状态，`select()` 函数就可以返回。
 
### 4. 异步I/O
 当用户进程发起 `read` 操作之后，立刻就可以开始去做其它的事。而另一方面，从 kernel 的角度，当它受到一个 asynchronous read 之后，首先它会立刻返回，所以不会对用户进程产生任何 block。然后，kernel 会等待数据准备完成，然后将数据拷贝到用户内存，当这一切都完成之后，kernel 会给用户进程发送一个 signal，告诉它 read 操作完成了。

 > 同步I/O 和 异步I/O 的区别：
 两者之间的区别就在于同步I/O 做 I/O操作的时候会将进程阻塞，所以，按照这个定义，之前的**阻塞I/O、非阻塞I/O、I/O多路复用都属于同步I/O**。
 非阻塞I/O 之所以属于同步I/O 是因为其非阻塞是因为并没有进行相应的IO操作，在其进行IO操作的时候，依旧是阻塞的。

5. 信号驱动I/O

## Linux 的 I/O 操作过程
### 1. GNU Linux I/O操作类别
 Linux 的文件操作并不仅仅是对我们通常意义上的文件的读写，基于**一切接文件**的思想，Linux的I/O操作类别包含一下几类：

  * 文件及流的标准输入输出
  * 底层输入输出
  * 文件系统接口
  * 管道及FIFO（先入先出队列）
  * Socket
  * 底层终端接口（tty）

### 2. 主要数据结构介绍
1. `FD`
  对于内核而言，所有打开文件都由文件描述符引用。
  文件描述符是一个非负整数。当打开一个现存文件或创建一个新文件时，内核向进程返回一个文件描述符。当读、写一个文件时，用`open`或`creat`返回的文件描述符`fd`标识该文件，将其作为参数传送给`read`或`write`。在 POSIX.1 应用程序中，文件描述符为常数 `0`、 `1` 和 `2` 分别代表 `STDIN_FILENO`、`STDOUT_FILENO` 和 `STDERR_FILENO`，意即标准输入，标准输出和标准出错输出，这些常数都定义在头文件 `<unistd.h>`中。文件描述符的范围是 `0~OPEN_MAX`，在目前常用的linux系统中，是32位整形所能表示的整数，即65535，64位机上则更多。

1. 进程中文件相关结构体
  * `struct file` 结构体定义在`include/linux/fs.h` 中。该结构体代表一个打开的文件，系统中每一个打开的文件在内核空间中都有一个关联的 `struct file`。它由内核在打开文件的时候创建，并传递给在该文件上进行操作的热河函数，在该文件的所有实例都关闭后，内核释放该数据结构。
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
  　　
    void                          *private_data;
  #ifdef CONFIG_EPOLL
    struct list_head               f_ep_links;
    spinlock_t                     f_ep_lock;
  #endif
  　struct address_space *f_mapping;
  };
  ````
  * `struct dentry`
  `dentry` 是 Linux 文件系统中某个索引节点（`inode`）的链接。`inode` 对应于物理磁盘上的具体对象，`dentry` 是一个内存上的实体，其中的`d_inode` 指向对应的 `inode`。一个 `inode` 可以在运行的时候链接多个 `dentry`，而 `d_count` 记录了链接的具体数量。
  ````C
  struct dentry {
    atomic_t                  d_count;    //目录项对象使用计数器,可以有未使用态,使用态和负状态
  　unsigned int              d_flags;    //目录项标志
  　struct inode *            d_inode;    //与文件名关联的索引节点
  　struct dentry *           d_parent;   //父目录的目录项对象
  　struct list_head          d_hash;     //散列表表项的指针
  　struct list_head          d_lru;      //未使用链表的指针
  　struct list_head          d_child;    //父目录中目录项对象的链表的指针
  　struct list_head          d_subdirs;  //对目录而言，表示子目录目录项对象的链表
  　struct list_head          d_alias;    //相关索引节点(别名)的链表
  　int                       d_mounted;  //对于安装点而言，表示被安装文件系统根项
  　struct qstr               d_name;     //文件名
  　unsigned long             d_time;
  　struct dentry_operations *d_op;       //目录项方法
  　struct super_block       *d_sb;       //文件的超级块对象
  　vunsigned long            d_vfs_flags;
  　void                     *d_fsdata;   //与文件系统相关的数据
  　unsigned char             d_iname [DNAME_INLINE_LEN]; //存放短文件名
  };````
  * `struct files_struct`
  对于每个进程，包含一个`files_struct` 结构，用来记录文件描述符的使用情况。
  ````C
  struct files_struct
  {
    atomic_t               count;               //使用该表的进程数
  　struct fdtable        *fdt;
  　struct fdtable         fdtab;
  　spinlock_t             file_lock ____cacheline_aligned_in_smp;
  　int                    next_fd;              //数值最小的最近关闭文件的文件描述符,下一个可用的文件描述符
  　struct embedded_fd_set close_on_exec_init;   //执行exec时需要关闭的文件描述符初值集合　struct embedded_fd_set open_fds_init;        //文件描述符的屏蔽字初值集合
  　struct file           *fd_array[NR_OPEN_DEFAULT]; 默认打开的fd队列
  };
  struct fdtable {
  　unsigned int      max_fds;
  　struct file     **fd;            //指向打开的文件描述符列表的指针，开始的时候指向fd_array，
  　当超过max_fds时，重新分配地址
  　fd_set           *close_on_exec; //执行exec需要关闭的文件描述符位图(fork，exec即不被子进程继承的文件描述符)
  　fd_set           *open_fds;      //打开的文件描述符位图
  　struct rcu_head   rcu;
  　struct fdtable   *next;
  };````
  * `struct fs_struct`
  `fs_struct` 是文件系统相关信息结构体。
  ````C
  struct fs_struct {
  atomic_t          count;     //共享表的进程个数
  rwlock_t          lock;      //自旋锁
  int               umask;     //文件权限掩码
  struct dentry    *root,      //根目录目录项
                    *pwd,       //当前目录目录项
                    *altroot;   //模拟根目录目录项
  struct vfsmount  *rootmnt,   //根目录文件系统对象
                    *pwdmnt,    //
                    *altrootmnt;//
  };````

  每个进程都有一个 `task_struct` 结构体，其中包含了一个 `fs_struct` 和一个 `files_struct` 结构体，其中 `files_struct` 中的 `fd_array` 记录了所有该进程打开的文件的 `file` 结构体，每个 `file` 结构体中的 `f_entry` 指向了当前文件的 `dentry` 结构体，`debtry` 结构体实际指向了相应的文件 `inode`。 

3. `inode`
  inode包含文件的元信息，具体来说有以下内容： 
  * 文件的字节数 文件的字节数 　　 　　
  * 文件拥有者的 文件拥有者的User ID 　　 　　
  * 文件的 文件的Group ID 　　 　　
  * 文件的读、写、执行权限 文件的读、写、执行权限 　　 　　
  * 文件的时间戳，共有三个： 文件的时间戳，共有三个：
    ctime指 指inode上一次变动的时间， 上一次变动的时间，
    mtime指文件内容上一次变动的时间， 指文件内容上一次变动的时间，
    atime指文件上一 指文件上一 次打开的时间。 次打开的时间。 　　 　　
  * 链接数，即有多少文件名指向这个 链接数，即有多少文件名指向这个inode 　　 　　
  * 文件数据 文件数据block的位置 

### 3. I/O操作过程
  1. 打开文件
  一个应用程序通过要求内核打开相应文件，宣告他要访问一个I/O设备 ，内核返回一个非负整数，叫描述符号(Descriptor）；

  2. 改变文件位置
  对于每个打开的文件，内核保持一个文件位置k，初始为0，这个文件位置是从文件头开始的偏移量。通过执行seek操作，显式地设置当前位置为k

  3. 读写文件
  读：从文件拷贝n>0个字节到存储器，写：从存储器拷贝n>0字节到文件

  4. 关闭文件
  通知内核关闭文件，作为响应，内核释放文件打开时创建的数据结构

## 三、记录锁
记录锁解决的是多个进程共同操作一个文件的问题，记录锁分为两种：
 * 建议性锁：建议性锁要求每个相关程序在访问文件前检查是否有锁存在，并尊重已有的锁。
 * 强制性锁：强制性锁是由内核执行的锁，当一个文件被上锁进行写操作时，内核将阻止任何其它的程序进行该文件的读写操作。

我们通常使用的是强制性锁，强制性锁的上锁函数是：
````C
int fcntl(int fd, int cmd, ...);
````
第一个参数 `fd` 显然指的是需要操作的文件描述符，第二个参数 `cmd` 是 `F_GETLK` / `F_SETLK` / `F_SETLKW`，当进行锁操作的时候，第三个参数是一个指向 `flock` 结构的指针。
````C
struct flock {
  short l_type;   //希望的锁类型 F_RDLCK(读锁) F_WRLCK（写锁） F_UNLCK（解锁）
  short l_whence; //区域的起始位置 SEEK_SET SEEK_CUR SEEK_END
  off_t l_start;  //区域的起始字节
  off_t l_len;    //区域的字节长度
  pid_t l_pid;    //持有锁的进程IO
}  ````

当 `cmd` 是 `F_GETLK` 时，函数会检查当前锁是否能够创建，如果可以创建，则将 `1_type` 设置为 `F_UNLCK`，否则则将当前锁的信息重写。
当 `cmd` 是 `F_SETLK` 时，设置相应的锁，如果不能创建，返回失败代码。
当 `cmd` 是 `F_GETLK` 时，如果当前设置的锁无法设置，则休眠等待锁创建。

记录锁的几个注意点：

1. 检查锁是否存在，和加锁过程并不是原子操作，所以，当检查当前锁不存在后加锁，依旧有可能会失败。
2. 如果两个进程相互等待对方持有并不释放（锁定）的资源时，造成死锁。
3. 当进程终止时，它所建立的所有锁释放
4. 当文件描述符关闭的时候，该文件描述符上的所有锁释放
5. `fork` 不继承任何锁。

## 四、I/O复用

I/O多路复用就是通过一种机制，一个进程可以监视多个描述符，一旦某个描述符就绪（一般是读就绪或者写就绪），能够通知程序进行相应的读写操作。

### 1. `select`
````C
int select (int n, fd_set *readfds, fd_set *writefds, fd_set *exceptfds, struct timeval *timeout);
````
`select` 函数监视的文件描述符分3类，分别是 `writefds`、`readfds`、和 `exceptfds`。调用后 `select` 函数会阻塞，直到有描述副就绪（有数据 可读、可写、或者有except），或者超时（`timeout`指定等待时间，如果立即返回设为null即可），函数返回。当 `select` 函数返回后，可以 通过遍历 `fdset`，来找到就绪的描述符。

函数的第一个参数 `n` 指的是**最大描述符编号加1**，即需要监视的3类文件描述符中的最大值加1.
函数中间的三个参数指函数监视的3类文件描述符的集合，分别是 `writefds`（可写）、`readfds`（可写）、和 `exceptfds`（处于异常）。有几个相应的接口可以设置这三个集合：
 ````C
 int FD_ISSET(int fd, fd_set *fdset); //测试描述符集中的某一位是否开启
 int FD_CLR(int fd, fd_set *fdset);   //清除描述符集中的某一位
 int FD_SET(int fd, fd_set *fdset);   //开启描述符集中的某一位
 int FD_ZERO(fd_set *fdset);          //清空描述符集中的所有位
 ````
函数的最后一个参数 `timeout` 指定等待的时间，当为 `NULL` 的时候，一直等待；当为 0 的时候，不等待。

函数有三个可能的返回值：
  * 返回-1；表示出错。
  * 返回0；表示没有描述符准备好。
  * 一个正的返回值：表示已经准备好的描述符数。
    已经准备好指的是：相应的读写没有阻塞，或者某个描述符存在未决异常条件。

### 2. `poll`
````C
int poll (struct pollfd *fds, unsigned int nfds, int timeout);
````
不同与 `select` 使用三个位图来表示三个 `fdset` 的方式，poll使用一个 pollfd的指针实现。
````C
struct pollfd {
    int fd;          /* file descriptor */
    short events;    /* requested events to watch */
    short revents;   /* returned events witnessed */
};````
`pollfd` 结构包含了要监视的 event 和发生的 event，不再使用 `select` **参数-值**传递的方式。
`pollfd` 并没有最大数量限制（但是数量过大后性能也是会下降）。`timeout` 参数只当了我们等待的时间，为-1表示永远等待，为0表示不等待，为正表示等待的时间（毫秒）。
和 `select` 函数一样，`poll` 返回后，需要轮询 `pollfd` 来获取就绪的描述符。


### 3. epoll
`epoll` 是在2.6内核中提出的，是之前的 `select` 和 `poll` 的增强版本。相对于 `select` 和 `poll` 来说，`epoll` 更加灵活，没有描述符限制。`epoll` 使用一个文件描述符管理多个描述符，将用户关系的文件描述符的事件存放到内核的一个事件表中，这样在用户空间和内核空间的 copy 只需一次。

`epoll` 操作过程需要三个接口，分别如下：
````C
int epoll_create(int size)； //创建一个epoll的句柄，size用来告诉内核这个监听的数目一共有多大
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event)；
int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout);
````
* `int epoll_create(int size);`
    创建一个 `epoll` 的句柄，`size` 用来告诉内核这个监听的数目一共有多大，这个参数不同于 `select()` 中的第一个参数，参数 `size` 并不是限制了epoll所能监听的描述符最大个数，只是对内核初始分配内部数据结构的一个建议。
    当创建好 `epoll` 句柄后，它就会占用一个fd值，在 Linux 下如果查看 `/proc/进程id/fd/`，是能够看到这个 `fd` 的，所以在使用完 `epoll` 后，必须调用 `close()` 关闭，否则可能导致fd被耗尽。

* `int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event)；`
    函数是对指定描述符 `fd` 执行 `op` 操作。
    - `epfd`：是 `epoll_create()` 的返回值。
    - `op`：表示 op 操作，用三个宏来表示：添加 `EPOLL_CTL_ADD`，删除 `EPOLL_CTL_DEL`，修改 `EPOLL_CTL_MOD`。分别添加、删除和修改对fd的监听事件。
    - `fd`：是需要监听的 `fd`（文件描述符）
    - `epoll_event`：是告诉内核需要监听什么事，`struct epoll_event`结构如下：
    ````C
    struct epoll_event {
      __uint32_t events;  /* Epoll events */
      epoll_data_t data;  /* User data variable */
    };````
  events可以是以下几个宏的集合：
  `EPOLLIN`：表示对应的文件描述符可以读（包括对端SOCKET正常关闭）；
  `EPOLLOUT`：表示对应的文件描述符可以写；
  `EPOLLPRI`：表示对应的文件描述符有紧急的数据可读（这里应该表示有带外数据到来）；
  `EPOLLERR`：表示对应的文件描述符发生错误；
  `EPOLLHUP`：表示对应的文件描述符被挂断；
  `EPOLLET`： 将EPOLL设为边缘触发(Edge Triggered)模式，这是相对于水平触发(Level Triggered)来说的。
  `EPOLLONESHOT`：只监听一次事件，当监听完这次事件之后，如果还需要继续监听这个socket的话，需要再次把这个socket加入到EPOLL队列里
* `int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout);`
    等待 `epfd` 上的 I/O 事件，最多返回 `maxevents` 个事件。
    参数 `events` 用来存储从内核得到事件的集合，`maxevents` 告之内核这个 `events` 有多大，这个 `maxevents` 的值不能大于创建 `epoll_create()` 时的 `size`，参数`timeout` 是超时时间（毫秒，0会立即返回，-1将不确定，也有说法说是永久阻塞）。该函数返回需要处理的事件数目，如返回0表示已超时。