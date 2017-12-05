---
title: C++ string、char *和int的转换
date: 2017-02-28 09:52:26
categories: 工作总结
tags: [string, int, char *, 类型转换]
---
工作中多次遇到 `string` `char *` 和 `int` 类型的互相转换，现在把自己掌握的相关转换方法记录在这里。
<!--more-->
## 一、`int` 类型转换其它类型
### 1.1 `int` 转 `char *`

* 第一种方法：使用 `itoa()` 函数
 这个很简单，但是不幸的是这并不是一个标准函数。

* 第二种方法：利用 ASCII 码进行转换
  这个需要自己编写相应的转换函数：
  ````C 
  const char* intToChar(char* buf, int m)
  {
    char temp[16];      //建立存储的空间,32位系统最大值为2147483647，所以不会超过16位。
    int isNegtive = 0;  //设置正负值
    int index;          //保存相应的位数
    if (m < 0)
    {
        isNegtive = 1;
        m = -m;
    }                    //判断正负值
    tmp[15] = '\0';      //字符串尾部为0
    index = 14;
    do 
    {
        tmp[index--] = m % 10 + '0';
        m /= 10;
    } while (m > 0);     //依次取出int的每一位。
    if(isNegtive)
        tmp[index--] = '-'; //加相应的符号
    strcpy(buf, tmp + index + 1);  //拷贝
    return buf;
  }````

* 第三种方法：使用spritf()
  这种方法最简单，也最普遍使用。
  ````C
  #include <stdio.h>
  int sprintf( char *buffer, const char *format, [ argument] … );````
  实例：
  ````C
  char buff[16];
  int m = 1234;
  sprintf(buff, "%d", m);````

### 1.2 `int` 转 `string`

* 第一种方法：使用 `to_string()`
  这个函数使用很方便，但是是 C++11 加入的。
* 第二种方法：使用 `sstream`
  这种方式也很方便，但是对格式的控制上不是那么严格，示例代码如下：
  ````C
  #include <sstream>
  #include <string>
  using namespace std;
  string intToString(int ss)
  {
    string str;
    stringstream st;
    st << ss;
    st >> str;
    return str;
  }````

## 二、`char *` 类型转换为其它类型
### 2.1 `char *` 转 `int`

* 第一种方法：使用 `atoi()` 函数
  和上面说的 `itoa()` 不同，这个函数是C库中的标准函数，所以，可以正常使用。
  ````C
  #include <stdlib.h>
  int atoi(const char *nptr);````
* 第二种方法：使用 ASCII码转换
  和上述类似，但是想必没有人在实际工程中这么做吧
* 第三种方法：使用 `sscanf()` 函数
  ````C
  #include <stdio.h>
  int sscanf(const char *buffer,const char *format,[argument ]...);````

### 2.2 `char *` 转 `string`
这个很简单，C风格字符串可以直接赋值给string类型。

## 三、`string` 类型转换为其它类型

### 3.1 `string` 类型转换为 `int` 类型

* 使用 `sstream`
  和上面的 `int` 转 `string` 类似，也可以相应转换。
* 使用 `stoi`
  这个函数和 `to_string` 类似，也是 C++11 独有的。

### 3.2 `string` 类型转换为 `char *` 类型

* 使用 `data()` 函数
* 使用 `c_str()` 函数
上述两个函数使用方法类似，需要注意两个函数返回的类型是`const char *`的，另外，相应的字符数组中的内容是会随着 `string` 调用其它方法改变的。
* 使用 `copy()` 函数

## 四、总结
上述分别讲了相应的直接转换，但是使用过程中很多情况下我们会使用 `char *` 作为中间量，来进行 `string` 和 `int` 的转换，同时 `ssprintf` 和 `sscanf` 是最推荐使用的。
