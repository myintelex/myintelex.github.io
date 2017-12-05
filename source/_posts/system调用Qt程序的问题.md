---
title: system调用Qt程序的问题
date: 2016-11-22 17:53:10
categories: 工作记录
tags: [system,Qt]
---

在公司做了一个嵌入式系统的引导程序，进程在系统开始后自启动（这里使用的是 rc.local 添加启动项的方法），随后按照相应的顺序依次启动相应的程序，之前一直没有出现什么问题，直到增加了一个启动 Qt 程序，并监控程序状态的需求后，出现了一些无法解释的问题。
<!--more-->
## 编写代码的过程 
* 第一个需求：进程中调用一个 Qt 程序。

这个代码很简单，为了省事，简单使用了 `system()` 函数。

        system("QtApplication -qws");

* 第二个需求：需要获取 QtApplication 的返回值。

system 函数的执行过程分为三个步骤：
 1. 创建一个子进程，主要是 `fork()` 等过程。
 2. 调用 `/bin/sh` 拉起 shell 脚本。
 3. 执行相应的 shell 脚本，waitpid()。

因此，system 的返回值(status)囊括了以上三个步骤的结果：
 1. 如果调用子进程失败，或者 waitpid() 返回除了 `EINTR` 之外的错误，system 返回 -1；
 2. 如果 shell 拉起失败或未正常执行结束（**只要能够调用到 `/bin/sh`，并且执行shell过程中没有被其他信号异常中断，都算正常结束**），原因值被写入到status的低8~15比特位中。<br>
 系统提供了宏：`WIFEXITED(status)` 来判断 shell 的执行结果。如果 `WIFEXITED(status)` 为真，则说明正常结束。
 3. 如果 shell 脚本正常执行结束，将 shell 返回值填到 status 的低8~15比特位中。<br>
 同样系统提供了宏：`WEXITSTATUS(status)` 来获取相应脚本的执行结果。

一个简单的执行代码如下, 因为程序需要执行一段时间，所以添加了一个轮询：
````C++
int system_call() {
  int status = 9;
  status = system("QtApplication -qws");
  for (int i = 0; i < 6000; i++) {
    if (status != 9) {
      if ( -1 == status) { //status == -1， 子进程创建失败
        printf("system error!"); 
        return -1;
      } else {
        if (WIFEXITED(status)) {//WIFEXITED(status) 为真，QtApplication 成功执行
          if (0 == WEXITSTATUS(status)) {
            printf("QtApplication successfully.\n"); //Qt 程序执行成功
            return 0;
          } else {
            printf("QtApplication failed, exit code: [%d]\n", WEXITSTATUS(status));//Qt 程序执行失败，获取失败返回值。
            return -1;
          }
        } else {
          printf("exit status = [%d]\n", WEXITSTATUS(status)); //WIFEXITED(status) 不为真，shell 调用失败。
          return -1;
        }
      }
    }
    usleep(100);
  }
  return -1;
}````

## 出现的问题
 编写完这个代码后简单的执行，调试，没有发现什么问题，所以开始进行测试，结果出现一个很奇怪的现象：
 当 Qt 程序没有执行完成的时候，system 返回了程序的返回值，并打印了以下的语句：`exit status = [0]`
 
 更为奇怪的是同样的问题，当手动执行 system_call 的时候，并不会出现这样的问题。

## 为什么会出现这样的问题？
经过对系统代码的分析，发现，在系统启动时启动一个拨号程序，而这个程序在系统启动大概200ms的时候杀死了所有的 sh 程序。

## 问题的解决
因为拨号程序不能更改，所以只能先通过 exec 的方法调用程序，毕竟**system()函数用起来很容易出错！！！**

于是，尝试了使用 exec 直接执行的方式来调用子程序，代码如下：
````C++
int system_call()
{
  pid_t pid, ret;
  int status;
  if ((pid = fork()) < 0) {
    printf("Fork Failed!");
    return -1;
  } else if (pid == 0) {
    if ((status = execlp("QtApplication", 
                        "QtApplication", 
                        "-qws", 
                        NULL)) < 0) {
      printf("EXEC Failed!");
    }
  } else {
    for (int i = 0; i < 6000; i++) {
      ret = waitpid(pid, &status, WNOHANG);
      if (ret == pid) {
        if (WIFEXITED(status)) {
          if (0 == WEXITSTATUS(status)) {
            printf("QtApplication successful", 3);
            return 0;
          } else {
            printf("tcu_update failed with CODE[%d]",  
                    WEXITSTATUS(status));
            return -1;
          }
        } else {
          printf("tcu_update start failed with CODE[%d]",  
                  WEXITSTATUS(status));
          return -2;
        }
      }
      usleep(100000);
    }
    return -2;
  }
}````

## 参考:
[【C/C++】Linux下system()函数引发的错误)](https://my.oschina.net/renhc/blog/54582)
[【C/C++】Linux下system()函数引发的错误](https://my.oschina.net/renhc/blog/54582)
