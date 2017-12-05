---
title: Linux驱动开发（三）——并发控制
date: 2016-12-20 03:08:01
categories: 驱动开发
tags: [驱动, 并发, 自旋锁, 信号量]
---
**并发**的定义是：多个执行单元同时、并行的执行。并发会导致**竞态**：并发的执行单元对共享资源的访问。以下几种情况会发生竞态：
1. 对称多处理器(SMP)的多个 CPU。
2. 单个 CPU 内部的多个进程。
3. 中断与进程之间。

解决竞态的途径是保证对共享资源的**互斥访问**：一个执行单元访问共享资源的时候，其他的执行单元被禁止访问。我们常用的互斥机制有：
 `中断屏蔽`、`原子操作`、`自旋锁`、`信号量`。
<!--more-->
## 一、中断屏蔽
因为 Linux 的异步 IO、进程调度等很多操作都是通过中断来进行的。所以，最简单的避免竞态的方法就是在进入临界区之前屏蔽系统的中断。但是同样，因为中断有这样重要的作用，长时间的屏蔽中断是很危险的。另外，中断屏蔽只能解决上述三种竞态情况中的后两种，对于第一种竞态是无法解决的。所以中断屏蔽通常和自旋锁搭配使用。
中断屏蔽的使用方法是：
````C 
local_irq_disable() //屏蔽中断
...
critical section /*临界区*/
...
local_irq_enable()````
中断屏蔽通常是和自旋锁联合使用的。

## 二、原子操作
有时候，共享的资源可能正好是一个整数值或者是位操作。内核提供了一种原子的整数类型（位类型）。相应的操作如下：

* 整形原子操作
  ````C
  void atomic_set(atomic_t *v, int i);             //设置原子变量 v 的值为 i
  atomic_t v = ATOMIC_INIT(0);                     //初始化原子变量 v 的值为 0
  atomic_read(atomic_t *v);                        //读取原子变量 v 的值
  void atomic_add(int i, atomic_t *v);             //原子变量 v 的值加 i
  void atomic_sub(int i, atomic_t *v);             //原子变量 v 的值减 i
  void atomic_inc(atomic_t *v);                    //原子变量 v 的值自加1
  void atomic_dec(atomic_t *v);                    //原子变量 v 的值自减1
  int atomic_inc_and_test(atomic_t *v);            //原子变量 v 的值自加1，并测试是否等于0
  int atomic_dec_and_test(atomic_t *v);            //原子变量 v 的值自减1，并测试是否等于0
  int atomic_sub_and_test(int i, atomic_t *v);     //原子变量 v 的值减 i，并测试是否等于0
  int atomic_add_and_return(int i, atomic_t *v);   //原子变量 v 的值加 i，并返回值
  int atomic_sub_and_return(int i, atomic_t *v);   //原子变量 v 的值减 i，并返回值
  int atomic_inc_and_return(atomic_t *v);          //原子变量 v 的值自加1，并返回值
  int atomic_dec_and_return(atomic_t *v);          //原子变量 v 的值自减1，并返回值
  ````
  `atomic_t` 的数据只能通过上述的函数进行访问。

* 位原子操作
  ````C
  void set_bit(nr, void *addr);           //设置addr地址的第nr位为1
  void clear_bit(nr, void *addr);         //设置addr地址的第nr位为0
  void change_bit(nr, void *addr);        //反置addr地址的第nr位
  test_bit(nr, void *addr);               //返回addr地址的第nr位
  int test_and_set_bit(nr, void *addr);   //返回addr地址的第nr位并置该位为1
  int test_and_clear_bit(nr, void *addr); //返回addr地址的第nr位并置该位为0
  int test_and_change_bit(nr, void *addr);//返回addr地址的第nr位并反置该位
  ````

## 三、自旋锁
自旋锁是一个互斥设备，他只能有两个值：锁定和解锁。为了获取一个自旋锁，程序先执行一个原子操作，测试相关的位，如果锁可用，则锁定，代码进入临界区；如果锁不可用，代码则进入循环测试直到该锁可用。

1. 自旋锁的初始化：
   可以使用两种方法进行自旋锁的初始化：
   编译时使用：`spinlock_t my_lock = SPIN_LOCK_UNLOCKED; `
   运行时使用：`void spin_lock_init(spinlock_t *lock);`  

2. 获取锁
   获取锁使用下面的函数：
   ````C
   void spin_lock(spinlock_t *lock);````
   需要注意的是，自旋锁的等待是不可中断的，一旦调用了该函数，在获取锁之前将一直处于自旋状态。

   如果不想阻塞等待可以使用非阻塞版本的获取锁：
   ````C
   void spin_trylock(spinlock_t *lock);````
   该函数在成功获取锁的情况下返回非零值，在未获取锁的情况下返回0

3. 释放锁
   释放锁的函数如下：
   ````C
   void spin_unlock(spinlock_t *lock);````
   这个函数一般与 `spin_lock` 和 `spin_trylock` 搭配使用。

4. 自旋锁和中断
   自旋锁可以保证临界区不受当前CPU和其他CPU的抢占进程打扰，即能解决前面提到的竞态中的前两种，但是依旧可能受到中断的影响，所以自旋锁有下列衍生：
   ````C
   void spin_lock_irq(spinlock_t *lock);   //在获取自旋锁之前禁止中断
   void spin_lock_irqsave(spinlock_t *lock, unsigned long flags); //在获取锁之前屏蔽中断，并将相应的中断状态保存在 flags 中
   void spin_lock_bh(spinlock_t *lock);   //在获取锁之前屏蔽软件中断，但保持硬件中断
   ````
   当然还有与上面几个函数一一对应的 unlock 函数，不再详细描述。

5. 读写自旋锁
   读写自旋锁是对自旋锁的扩展，它允许多个读操作并发执行，但是只能有一个写单元。相应的函数如下：
   ````C
   rwlock_t my_rwlock = RW_LOCK_UNLOCKED;  //静态初始化读写自旋锁
   rwlock_init(rwlock_t *my_rwlock);       //动态初始化读写自旋锁
   void read_lock(rwlock_t *my_rwlock);    //获取读锁
   void read_unlock(rwlock_t *my_rwlock);  //释放读锁
   void write_lock(rwlock_t *my_rwlock);   //获取写锁
   void write_unlock(rwlock_t *my_rwlock); //释放写锁
   ````
   读写自旋锁也有相应的中断衍生版本。

6. 顺序锁
   对于资源较小，频繁被读取但是很少写入的资源，可以使用顺序锁。顺序锁的读执行单元不会被写执行单元阻塞。
   顺序锁的初始化类似于自旋锁：
   ````C
   seqlock_t my_seqlock = SEQLOCK_UNLOCKED;  //静态初始化读写自旋锁
   seqlock_t_init(seqlock_t *my_seqlock);       //动态初始化读写自旋锁
   ````
   对于写单元来说，相应的获取锁和释放锁的机制和自旋锁一致，不再详说，详细说下读单元的执行。
   ````C
   unsigned int seq;
   do {
     seq = read_seqbegin(&my_seqlock);
     /*相应的操作*/
   } while (read_seqretry(&the_lock, seq))````
   `read_seqbegin`会返回当前顺序锁的顺序号，`read_seqretry` 会检查当前的顺序号是否改变。
   通常不能使用顺序锁来保护数据结构中含有指针的数据。

7. 读-拷贝-更新
   `RCU`（read-copy-update，读-拷贝-更新）是基于原理命名的，在这里不再详细介绍
   
## 四、信号量
信号量和自旋锁类似，只有得到信号量的进程才能执行临界区代码。但是与自旋锁不同的是，当获取不到信号量的时候，进程不会原地打转而是会进入休眠等待状态。

1. 信号量的定义和初始化
   有三种方法可以初始化一个信号量：
   ````C
   void sema_init(struct semaphore *sem, int val); //初始化信号量 sem，并将 sem 的值设为 val
   void init_MUTEX(struct semaphore *sem);         //初始化信号量 sem 为0
   void init_MUTEX_LOCKED(struct semaphore *sem);  //初始化信号量 sem 为1
   DECLARE_MUTEX(name);                            //声明并初始化一个名为 name 的信号量为0
   DECLARE_MUTEX_LOCKED(name);                     //声明并初始化一个名为 name 的信号量为1
   ````
   对于含有 `LOCKED` 的初始化方法，信号量的初始状态就是锁定的。

2. 信号量的获取
   信号量的获取使用下列方式：
   ````C
   void down(struct semaphore *sem);               //会导致睡眠，不能被信号打断
   int down_interruptible(struct semaphore *sem);  //会导致睡眠，并能被信号打断
   int down_trylock(struct semaphore *sem);        //不导致睡眠
   ````
   使用 `down_interruptible` 函数被信号打断和使用 `down_trylock` 函数未获取信号量，函数会返回非零值。否则返回0。

3. 信号量的释放
   ````C
   void up(struct semaphore *sem);
   ````
   该函数会释放信号量，唤醒等待者。

4. 完成量
   completion (完成量) 用于一个执行单元等待另一个执行单元执行完成某事。相应的使用方法如下：
   ````C
   struct completion my_completion; 
   init_completion(&my_completion); //定义并初始化 completion
   /*
   DECLARE_COMPLETION(my_completion); //另一种创建 completion 的方法
   */
   void wait_for_completion(struct completion *c); //等待 completion 被唤醒
   ...
   void complete(struct completion *c); //唤醒一个等待 c 的执行单元
   void complete_all(struct completion *c); //唤醒所有等待 c 的执行单元
   INIT_COMPLETION(struct completion my_completion); //用于重新初始化一个信号量
   void completion_and_exit(struct completion *c, long retval); //
   ````
   对于 `completion_and_exit` 的用法还有不清楚的地方，等验证完成后再来补充。

5. 读写信号量
   读写信号量类似与读写自旋锁，使用方法如下：
   ````C
   struct rw_semaphore my_rws; //定义读写信号量
   void init_rwsem(struct rw_semaphore *sem); //初始化读写信号量
   void down_read(struct rw_semaphore *sem); 
   void down_read_trylock(struct rw_semaphore *sem); //读信号量获取
   void up_read(struct rw_semaphore *sem); //读信号量释放
   void down_write(struct rw_semaphore *sem);
   void down_write_trylock(struct rw_semaphore *sem); //写信号量获取
   void up_write(struct rw_semaphore *sem); //写信号量释放
   ````

## 五、互斥体
   互斥体简单实现了互斥的功能：
   ````C
   struct mutex my_mutex;
   mutex_init(&my_mutex);  //初始化互斥体
   void inline __sched mutex_lock(struct mutex *lock);
   int __sched mutex_lock_interruptible(struct mutex *lock);
   int __sched mutex_trylock(struct mutext *lock);   //获取互斥体
   void __sched mutex_unlock(struct mutext *lock);   //释放互斥体
   ````