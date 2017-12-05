---
title: Linux驱动开发（五）——中断和时钟
date: 2016-12-20 06:08:01
categories: 驱动开发
tags: [驱动, 中断, 定时器, 延时]
---
除了部分仅使用 I/O 寄存器的设备之外，大部分的设备都需要与外部打交道。这些外部的工作通常都是在与处理器完全不同的时间周期内完成的，且通常比处理器的处理速度要慢，所以一直让 CPU 等待设备的处理是不能令人满意的。这时候我们就需要使用中断。
<!--more-->



## 一、中断编程
中断指的是 CPU 在执行的过程中，出现了某些突发事件基带处理，CPU 必须暂停执行当前的程序，转去处理相应的突发事件，处理完毕后返回程序被中断的位置并继续执行。

设备的中断会打断内核内部正常的进程调度和运行，因此要求中断处理程序必须尽可能的短小。但是，往往现实中需要中断到来的时候完成大量的工作。为了解决这个冲突，Linux 将中断处理程序分为了两个部分：中断的顶半部和底半部。
中断的顶半部完成尽可能少的工作，往往是简单的读取寄存器中的中断状态，并清除中断标志后就进行登记中断的工作。登记中断是将底半部的处理程序挂到设备的底半部执行队列中。因此，底半部是中断处理工作的重心，且可以被新的中断打断。

### 1. 申请和释放中断
申请中断使用下面的函数：
````C
int request_irq(unsigned int irq, irq_handlert handler, 
                unsigned long irqflags, const char *devname,
                void *dev_id);````
功能：申请中断。
参数：`irq` 申请的硬件中断号；`handler` 顶半部中断处理函数；`irqflags` 中断处理的属性；`devname` 用来显示中断拥有者；`dev_id` 传递给`handler` 用于中断共享的中断信号线。`irqflags` 可能的值有：
* `SA_INTERRUPT`
  表示快速中断处理，快速中断处理程序被调用时将屏蔽中断。
* `SA_SHIRQ`
  表示中断共享

返回值：成功返回0，返回`-EINVAL` 表示中断号无效，或者处理函数指针为 NULL，返回`-EBUSY` 表示中断已经被占用。

相应的 `handler` 定义如下：
````C
typedef irqreturn_t (*irq_handlert_t)(int, void *);
typedef int irq_handlert_t;````

释放中断使用下面的函数：
````C
void free_irq(unsigned int irq, void *dev_id);````
相应的参数定义和 `request_irq` 类似。

### 2. 使能和屏蔽中断
使用下面的函数屏蔽单个中断
````C
void disable_irq(int irq);
void disable_irq_nosync(int irq);
void enable_irq(int irq);
````
第一个函数会禁止相应的中断，并等待正在进行的中断处理例程完成。
第二个函数会立即返回。

使用下面的两个函数可以屏蔽当前 CPU 的所有的中断：
````C
void local_irq_save(unsigned long flags);
void local_irq_disable(void);````
第一个函数会把当前的中断状态保存在 `flags` 中，然后禁用当前处理器上的中断发送。第二个函数直接禁用所有的中断。相应的恢复函数是：
````C
void local_irq_restore(unsigned long flags);
void local_irq_enable(void);````

### 3. 底半部机制


1. `tasklet`
   `tasklet` 是一个基于软中断的特殊函数，它的使用很简单，就是定义一个 `tasklet` 和它的处理函数，并将他们两个关联起来。相应的模板代码如下：
   ````C
   //定义 tasklet 和底半部函数并关联
   void xxx_do_tasklet(unsigned long arg);   //定义一个 tasklet 处理函数
   DECLARE_TASKLET(xxx_tasklet, xxx_do_tasklet, 0); //关联 tasklet 处理函数
   //中断处理底半部
   void xxx_do_tasklet(unsigned long arg) {
     ...
   }
   //中断处理顶半部
   irqreturn_t xxx_interrupt(int irq, void *dev_id) {
     ...
     tasklet_schedule(&xxx_tasklet);  //调用 tasklet 处理函数
     return IRQ_HANDLED;
   }
   //设备驱动加载函数
   int __init xxx_init(void) {
     ...
     //申请中断
     result = request_irq(xxx_irq, xxx_interrupt,
                          SA_INTERRUPT, "xxx", NULL);
     ...
   }
   //设备驱动卸载函数
   void __exit xxx_exit(void) {
     ...
     free_irq(xxx_irq, xxx_interrupt);
     ...
   }````

   `tasklet` 是运行于软中断上下文的，属于原子上下文的一种，所以，是不能在 `tasklet` 处理函数中睡眠的。

2. 工作队列
   工作队列的使用方法和 `tasklet` 类似，模板代码如下：
   ````C
   //定义 tasklet 和底半部函数并关联
   struct work_struct xxx_wq;   //定义一个工作队列
   void xxx_do_work(unsigned long); //定义一个处理函数
   //中断处理底半部
   void xxx_do_tasklet(unsigned long arg) {
     ...
   }
   //中断处理顶半部
   irqreturn_t xxx_interrupt(int irq, void *dev_id) {
     ...
     schedule_work(&xxx_wq); //调用工作队列处理函数
     return IRQ_HANDLED;
   }
   //设备驱动加载函数
   int __init xxx_init(void) {
     ...
     //申请中断
     result = request_irq(xxx_irq, xxx_interrupt,
                          SA_INTERRUPT, "xxx", NULL);
     ...
     INIT_WORK(&xxx_wq, (void (*)(void *)) xxx_do_work, NULL);//绑定工作队列和处理函数
   }
   //设备驱动卸载函数
   void __exit xxx_exit(void) {
     ...
     free_irq(xxx_irq, xxx_interrupt);
     ...
   }````
   工作队列运行于进程上下文，所以可以在工作队列处理函数中睡眠。


### 4. 中断共享
因为设备上的中断线总是不够用的，所以 Linux 支持所有的总线的中断共享。使用中断共享的方法和之前介绍的类似，有几点不同：
1. 请求中断，调用 `request_irq()` 时，应设置相应的 `irqflags` 为 `SA_SHIRQ`。
2. `dev_id` 参数必须是唯一的。且不能为 NULL。

当中断到来时，会遍历执行所有共享该中断的所有中断处理函数，应该在相应的中断处理函数顶半部中根据传入的 `dev_id` 值判断是否为当前设备的中断，是应返回 `IRQ_HANDLED` ，如果不是则返回 `IRQ_NONE`。

## 二、时间操作
中断一个十分重要的应用是定时器中断，内核使用其来跟踪时间流。定时器中断是有系统定时硬件以周期性的间隔产生，这个间隔是内核根据 `HZ` 的值来决定的。每次当内核定时器发生中断时，内核内部计数器就加1，这个计数器的值在系统引导的时候设置为0，因此它的值就是系统启动以来的时钟滴答数。这个值保存在一个 64 位的变量 `jiffies_64` 中，但我们更多使用的是一个 `jiffies` 变量。

一个内核定时器是一个数据结构，他告诉内核在用户定义的时间点使用用户定义的参数来执行一个用户定义的函数。它的结构如下：
````C
struct timer_list {
  struct list_head entry;          //定时器列表
  unsigned long expires;           //定时器到时时间
  void (*function)(unsigned long); //定时器处理函数
  unsigned long data;              //处理函数参数
  struct timer_base_s *base;
  ...
}````

### 2.1 定时器的使用
1. 初始化定时器
  ````C
  //定义一个定时器
  struct timer_list my_timer;
  //初始化定时器   
  void init_timer(struct timer_list *timer);
  TIMER_INITIALIZER(_function, _expires, _data);
  //定义并初始化定时器
  DEFINE_TIMER(_name, _function, _expires, _data);
  //初始化定时器并赋值
  static inline void setup_timer(struct timer_list *timer,
                                void (*function)(unsigned long),
                                unsigned long data);````
  其中，`init_timer` 函数初始化 `timer` 并将 `timer_list` 结构体中的 `entry` 的 `next` 设置为 NULL，将 `base` 赋值。

2. 增加定时器
  ````C
  void add_timer(struct timer_list *timer);````
  用来将定时器加入到内核定时器列表中。

3. 删除定时器
  ````C
  int del_timer(struct timer_list *timer);````
  用于删除定时器。

4. 修改定时器
  ````C
  int mod_timer(struct timer_list *timer, unsigned long expires);````
  用于修改定时器的到期时间。

### 2.2 内核延时

1. 短延时
  使用下面三个函数实现纳秒、微秒、毫秒延时：
  ````C
  void ndelay(unsigned long nsecs);
  void udelay(unsigned long usecs);
  void mdelay(unsigned long msecs);````
  这三个函数的实现是通过一个类似 `while(timer--)` 的循环完成的，因此是需要耗费 CPU 资源的。

  对于微秒级以上延时有下列函数：
  ````C
  void msleep(unsigned int millisecs);
  unsigned long msleep_interruptible(unsigned int millisecs);
  void ssleep(unsigned int seconds);
  ````
  `msleep` 和 `ssleep` 分别用于毫秒和秒延时，这三个函数都会将相应的调用进程睡眠，`msleep_interruptible` 允许进程睡眠被打断。

2. 长延时（忙等待）
   长延时的实现通常是比对当前和目标的 `jiffies` 的值，相应的实现代码是：
   ````C
   while (time_before(jiffies, j1))
    cpu_relax();
   ````
   上面的代码是忙等待模式的：循环会占用整个延时期间的处理器，严重降低系统性能，而且如果在进入延时前禁用了中断，系统将一直处于延时状态。

3. 睡着延时
   睡着延时是在等待时间到来之前让进程处于睡眠状态的一种延时方式，使用方法如下：
   ````C
   //1. 设置进程的状态。
   set_current_state(TASK_INTERRUPTIBLE);
   //2. 等待延时时间。
   schedule_timeout(delay);
   ````

  




