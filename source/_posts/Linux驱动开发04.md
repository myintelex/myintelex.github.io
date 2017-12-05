---
title: Linux驱动开发（四）——I/O操作
date: 2016-12-20 06:08:01
categories: 驱动开发
tags: [驱动, I/O, poll, 等待队列, 异步通知]
---
对于 Linux 设备来说，设备是当作文件来处理的。所以，很多设备的 I/O 操作都是很重要的一个部分。这篇博客总结了 Linux 驱动开发中 I/O 相关的一些内容。包含阻塞和非阻塞 I/O 、I/O 轮询、异步 I/O 等。
<!--more-->

## 一、阻塞和非阻塞 I/O
阻塞是指设备操作时，如果不能获取资源，则挂起相应的操作单元直到满足可操作的条件后进行操作。而非阻塞是指在不能满足操作需求的时候，相应的操作单元并不挂起，而是选择放弃或者不停的查询直到满足条件。
我们可以显式的设置一个设备文件为非阻塞 I/O。我们可以在打开一个文件的时候通过设置 `filp->f_flags` 为 `O_NONBLOCK` 来实现。如果设置为阻塞模式，当使用 `open()` 函数打开一个文件的时候，I/O 操作会在操作条件不满足的时候返回 `EAGAIN`，此时必须严格检查相应的 `errno`。
`
而对于我们的驱动程序来讲，我们更多的应该（默认）阻塞进程，将其置入**休眠状态**。
**休眠状态**：当一个进程被置入休眠状态之后，他会被标记为一种特殊的状态并从 CPU 的运行队列中移走，直到某些情况修改了这个状态。对于休眠状态，有两个重要的规则：
 * 永远不要在原子操作中进入休眠。
 * 我们对唤醒之后的状态不能做任何假定，必须检查以确保我们等待的条件为真。

另外一个需要考虑的问题是，进入休眠的进程必须确保有其他进程会在其他地方唤醒休眠的进程，且休眠的进程能够被找到。

### 1.1 等待队列
如上文所说，我们需要能够找到进入休眠的进程。我们使用一种叫做等待队列的数据结构来实现这个功能。
**等待队列**是以队列为基础数据结构，与进程调度机制相结合，用于实现内核中的异步事件通知的机制。它实际上是一个进程链表，其中包含了等待某个特定事件的所有进程。

1. 初始化等待队列
   等待队列使用一个**等待队列头**来管理，他可以通过下面两种方式来定义并初始化：
    ````C
    /*使用动态方法定义并初始化*/
    wait_queue_head_t my_queue;
    init_waitqueue_head(&my_queue);
    /*使用静态定义并初始化*/
    DECLARE_WAIT_QUEUE_HEAD(name);
    ````
    我们使用这个等待队列头来指代我们的等待队列。

2. 进入休眠
   * 当我们 I/O 遇到了阻塞条件，我们需要将其休眠的时候，我们可以使用 `wait_event` 宏来设置相应的进程进入休眠。
     ````C
    wait_event(queue, condition); 
    wait_event_interruptible(queue, condition); 
    wait_event_timeout(queue, condition, timeout); 
    wait_event_interruptible_timeout(queue, condition, timeout);
    ````
    上面相应的宏中，需要注意的是**`queue` 是相应的等待队列头，它是值传递的**。`condition` 是一个布尔表达式，在表达式为真之前，进程会持续休眠。该表达式有可能被多次求值。
    `wait_event` 将相应的进程置于非中断休眠，我们更多使用的是 `wait_event_interruptible`，他可以被信号中断，当被信号中断的时候，返回一个非零值；后两个函数会设置相应的等待时间，超过等待时间之后，进程返回，返回值为0。

   * 我们也可以手动设置一个进程进入休眠：
     1. 建立一个并初始化一个**等待队列入口**。
        我们可以使用宏 `DEFINE_WAIT(my_wait)` 来静态定义并初始化一个名为 `my_wait` 的等待队列入口，也可以使用下面的动态方法：
        ````C
        wait_queue_t my_wait;
        init_wait(&my_wait);
        ````
        我们更多使用的是静态方法。
     2. 将相应的等待队列入口加入到队列中：
        ````C
        void prepare_to_wait(wait_queue_head_t *queue,
                             wait_queue_t *wait,
                             int state);````
        其中 `queue` 指我们的等待队列头，`wait` 指我们的进程等待队列入口，`state` 是进程的新状态，有两种值 `TASK_INTERRUPTIBLE`（可中断休眠，常用值）`TASK_UNINTERRUPTIBLE`（不可中断休眠）。
     3. 再次检查条件，让出 CPU：
        ````C
        if (condition)
            schedule();
        ````
     4. 进程被唤醒后执行相应的清理工作：
        ````C
        void finish_wait(wait_queue_head_t *q, wait_queue_t *wait);
        ````
        该函数将进程状态更改为 `TASK_RUNNING`，并从等待队列中删除该进程。
     5. 最后，我们需要测试我们是否是被信号唤醒的。

3. 对于阻塞的进程，我们需要能够唤醒它，唤醒函数如下：
````C
void wake_up(wait_queue_head_t *q);
void wake_up_interruptible(wait_queue_head_t *q);````
第一个函数会唤醒等待队列 `q` 上面的所有进程，而第二个只能唤醒等待队列上执行可中断休眠的进程。通常我们是将对应版本的 `wait` 和 `wake` 搭配使用。


### 1.2 轮询操作
在应用层有轮询的概念（I/O 多路复用），指的是在读取多个文件的时候，阻塞进程直到给定的文件描述符集合中任何一个可以进行相应的读写操作。相应的应用层函数有 `select`、`poll`、`epoll`等，他们在底层调用的是 `poll` 函数。它的原型是：
````C
unsigned int (*poll)(struct file *filp, poll_table *wait);
````

这个函数要完成两个工作：
1. 对可能引起设备文件状态变化的等待队列调用 `poll_wait()` 函数，将对应的等待队列添加到 `poll_table`。
2. 返回表示是否能对设备进行无阻塞的读写访问的掩码。

`poll_wait()` 函数的原型如下：
 ````C
 void poll_wait(struct file *filp, wait_queue_head_t *queue, poll_table *wait);
 ````
 功能：将当前进程加入到函数指定的等待列表 `wait`。
 参数：`filp` 打开的文件指针，`queue` 需要加入的 `wait` 的等待队列。

 一个相应的 `poll` 函数实现代码如下：
 ````C
 static unsigned int xxx_poll(struct file *filp, poll_table *wait) {
     struct xxx_dev *dev = filp->private_data;
     unsigned int mask = 0;
     poll_wait(filp, &dev->r_wait, wait);
     poll_wait(filp, &dev->w_wait, wait);
     if (readable)
        mask |= POLLIN|POLLRDNORM;
     if (writable)
        mask |= POLLOUT|POLLWRNORM;
    return mask;
 }````

## 二、异步通知
异步通知的意思是：一旦设备就绪，就主动通知应用程序，这样应用程序就不用查询设备的状态，类似于硬件上的中断的概念。在 Linux 中，异步通知使用信号来实现。为了使设备支持异步通知机制，驱动程序中涉及三项工作：
1. 支持 `F_SETOWN` 命令，能在这个命令处理中设置 `filp->f_owner` 为对应的进程 ID，这部分工作由内核完成。
2. 支持 `F_SETFL` 命令，每当 `FASYNC` 标志改变的时候，驱动程序中的 `fasync()` 函数将得以执行。因此，应在设备驱动中实现 `fasync()` 函数。
3. 在设备资源可以获得的时候，调用 `kill_fasync()` 函数激发相应的信号。

设备驱动的异步通知机制编程涉及到两个函数和一个结构体：

* `fasync_struct` 结构体
* `FASYNC` 标志变更函数
  ````C
  int fasync_helper(int fd, struct file *filp, int mode, struct fasync_struct **fa);````
* 释放信号的函数
  ````C
  void kill_fasync(struct fasync_struct **fa, int sig, int band);````

`fasync_struct` 同样一般定义在相应的设备结构体 `xxx_dev` 中。
设备驱动的 `fasync()` 函数，一般参照下面的模板编写：
````C
static int xxx_fasync(int fd, struct file *filp, int mode) {
    struct xxx_dev *dev = filp->private_data;
    return fasync_helper(fd, filp, mode, &dev->fasync_queue);
}````


在设备删除的时候，还需要将文件从异步通知列表中删除，调用`xxx_fasync(-1, filp, 0);`。
