---
title: Linux学习总结（二）——Shell编程
date: 2016-12-16 09:24:39
categories: Linux学习记录
tags: [Linux, Shell]
---

Shell 是 Unix 系统下的一个命令解释器，主要用于系统和用户的交互。在 Unix 上有各种不同版本的 Shell，Bash是Linux标准默认的Shell，它是BourneAgain Shell的缩写。我们这里主要讨论的也是 Bash。

<!--more-->
Linux常用命令表 ( 约60个常用命令)|

|分类|命令|
|-------------|---|
|文件相关|`cd`, `whereis`, `pwd`, `ls`, `file`, `echo`, `mkdir/rmdir`, `cat/more`, `cp/mv/rm`, `chowm/chgrp`, `chmod`, `grep`, `find`, `locate`, `ln`, `gzip`, `tar`, `diff`, `patch`|
|磁盘管理|`df`, `du`, `mount/umount`, `fdisk`, `mkfs`|
|用户管理|`useradd`, `userdel`, `usermod`, `groupadd`, `groupdel`, `groupmod`, `groups`, `passwd`, `id`, `who`, `whoami`|
|系统及相关|`su`, `sudo`, `export`, `shutdown`, `poweroff`,`halt`, `reboot`, `ps`, `top`, `uname`, `uptime`, `clear`, `cal`, `date/time`|
|网络配置|`netstat`, `nslookup`, `finger`, `ping`, `ifconfig`, `ftp`, `telnet`, `ssh`|

## Shell 编程
Shell 脚本（shell script），是一种为shell编写的脚本程序。
> 简单的几点 shell 语法：
* `#！`  指定sh解释程序
* `#；`  注释
* 关于空格， `=` 不加空格， 运算符要加空格。


## Shell 变量
### 用户自定义变量
变量类型只支持字符串，不支持整形，字符，浮点;
* 等号前后不要有空格
* 一般变量命名用全大写
* `unset` 命令删除变量赋值
* `readonly` 标定只读变量
* `export` 来指定global变量

### 预定义变量
* `$0` 与键入的命令行一样，包含脚本文件名
* `$1, $2,...$9` 分别包含第一个到第九个命令行参数
* `$#` 命令行参数的个数
* `$@` 所有命令行参数
* `$?` 前一个命令的退出状态
* `$*` 所有命令行参数
* `$$` 正在执行的进程ID号

### 环境变量
* `HOME`： 当前用户的主目录
* `PATH`: shell搜索路径
* `TERM`: 终端程序名称
* `UID`: 当前用户的识别字，取值是由数位构成的字串。
* `PWD`: 当前工作目录的绝对路径名，该变量的取值随cd命令的使用而变化。
* `PS1`: 主提示符，在特权用户下，默认的主提示符是#，在普通用户下，默认的主提示符是$。# , $
* `PS2`: 在Shell接收用户输入命令的过程中，如果用户在输入行的末尾输入“\”然后回车，或者当用户按回车键时Shell判断出用户输入的命令没有结束时，就显示这个辅助提示符，提示用户继续输入命令的其余部分，默认的辅助提示符是>

### 字符串
字符串是shell编程中最常用最有用的数据类型（除了数字和字符串，也没啥其它类型好用了），字符串可以用单引号，也可以用双引号，也可以不用引号。

* 单引号
 ````
 str='this is a string'````
 单引号字符串的限制：
  * 单引号里的任何字符都会原样输出，单引号字符串中的变量是无效的；
  * 单引号字串中不能出现单引号（对单引号使用转义符后也不行）。

* 双引号
 ````shell
 your_name='qinjx'
 str="Hello, I know your are \"$your_name\"! \n"
 ````
 双引号的优点：
  * 双引号里可以有变量
  * 双引号里可以出现转义字符

* 拼接字符串
 ````shell
 your_name="qinjx"
 greeting="hello, "$your_name" !"
 greeting_1="hello, ${your_name} !"
 echo $greeting $greeting_1
 ````

* 获取字符串长度
 ````shell
 string="abcd"
 echo ${#string} #输出 4````

* 提取子字符串
 以下实例从字符串第 `2` 个字符开始截取 `4` 个字符：
 ````shell
 string="runoob is a great site"
 echo ${string:1:4} # 输出 unoo````

* 查找子字符串
 查找字符 `i` 或 `s` 的位置：
 ````
 string="runoob is a great company"
 echo `expr index "$string" is`  # 输出 8````
 注意： 以上脚本中 ` 是反引号，而不是单引号，不要看错了哦。

### Shell 数组
bash支持一维数组（不支持多维数组），并且没有限定数组的大小。
类似与C语言，数组元素的下标由0开始编号。获取数组中的元素要利用下标，下标可以是整数或算术表达式，其值应大于或等于0。

* 定义数组
 在Shell中，用括号来表示数组，数组元素用"空格"符号分割开。定义数组的一般形式为：
 `数组名=(值1 值2 ... 值n)`
 例如：
 ````shell
 array_name=(value0 value1 value2 value3)````
 或者
 ````shell
 array_name=(
 value0
 value1
 value2
 value3
 )````
 还可以单独定义数组的各个分量：
 ````shell
 array_name[0]=value0
 array_name[1]=value1
 array_name[n]=valuen````
 可以不使用连续的下标，而且下标的范围没有限制。

* 读取数组
 读取数组元素值的一般格式是：
 `${数组名[下标]}`
 例如：
 `valuen=${array_name[n]}`
 使用@符号可以获取数组中的所有元素，例如：
 `echo ${array_name[@]}`

* 获取数组的长度
 获取数组长度的方法与获取字符串长度的方法相同，例如：
 ````shell
# 取得数组元素的个数
length=${#array_name[@]}
# 或者
length=${#array_name[*]}
# 取得数组单个元素的长度
lengthn=${#array_name[n]}````

## Shell 语句
### 基本运算符
Shell 和其他编程语言一样，支持多种运算符，原生bash不支持简单的数学运算，但是可以通过其他命令来实现，例如 awk 和 expr，expr 最常用。
expr 是一款表达式计算工具，使用它能完成表达式的求值操作。
例如，两个数相加(注意使用的是反引号 ` 而不是单引号 ')：
````shell
#!/bin/bash
val=`expr 2 + 2`
echo "两数之和为 : $val"````
执行脚本，输出结果如下所示：
> 两数之和为 : 4

注意：
 * 表达式和运算符之间要有空格，例如 2+2 是不对的，必须写成 2 + 2，这与我们熟悉的大多数编程语言不一样。
 * 完整的表达式要被 \` \` 包含，注意这个字符不是常用的单引号，在 Esc 键下边。
 * 乘号(*)前边必须加反斜杠(\\)才能实现乘法运算.

常用的运算符如下：

|运算符|说明|举例|
|--|--|--|
|+|加法|expr $a + $b 结果为 30。|
|-|减法|expr $a - $b 结果为 -10。|
|*|乘法|expr $a \* $b 结果为  200。|
|/|除法|expr $b / $a 结果为 2。|
|%|取余|expr $b % $a 结果为 0。|
|=|赋值|a=$b 将把变量 b 的值赋给 a。|
|==|相等。用于比较两个数字，相同则返回 true。|[ $a == $b ] 返回 false。|
|!=|不相等。用于比较两个数字，不相同则返回 true。|[ $a != $b ] 返回 true。|
|-eq|检测两个数是否相等，相等返回 true。|[ $a -eq $b ] 返回 false。|
|-ne|检测两个数是否相等，不相等返回 true。|[ $a -ne $b ] 返回 true。|
|-gt|检测左边的数是否大于右边的，如果是，则返回 true。|[ $a -gt $b ] 返回 false。|
|-lt|检测左边的数是否小于右边的，如果是，则返回 true。|[ $a -lt $b ] 返回 true。|
|-ge|检测左边的数是否大于等于右边的，如果是，则返回 true。|[ $a -ge $b ] 返回 false。|
|-le|检测左边的数是否小于等于右边的，如果是，则返回 true。|[ $a -le $b ] 返回 true。|
|!|非运算，表达式为 true 则返回 false，否则返回 true。|[ ! false ] 返回 true。|
|-o|或运算，有一个表达式为 true 则返回 true。|[ $a -lt 20 -o $b -gt 100 ] 返回 true。|
|-a|与运算，两个表达式都为 true 才返回 true。|[ $a -lt 20 -a $b -gt 100 ] 返回 false。|
|&&|逻辑的 AND|[[ $a -lt 100 && $b -gt 100 ]] 返回 false|
|II|逻辑的 OR|[[ $a -lt 100 II $b -gt 100 ]] 返回 true|
|=|检测两个字符串是否相等，相等返回 true。|[ $a = $b ] 返回 false。|
|!=|检测两个字符串是否相等，不相等返回 true。|[ $a != $b ] 返回 true。|
|-z|检测字符串长度是否为0，为0返回 true。|[ -z $a ] 返回 false。|
|-n|检测字符串长度是否为0，不为0返回 true。|[ -n $a ] 返回 true。|
|str|检测字符串是否为空，不为空返回 true。|[ $a ] 返回 true。|
|-b file|检测文件是否是块设备文件，如果是，则返回 true。|[ -b $file ] 返回 false。|
|-c file|检测文件是否是字符设备文件，如果是，则返回 true。|[ -c $file ] 返回 false。|
|-d file|检测文件是否是目录，如果是，则返回 true。|[ -d $file ] 返回 false。|
|-f file|检测文件是否是普通文件（既不是目录，也不是设备文件），如果是，则返回 true。|[ -f $file ] 返回 true。|
|-g file|检测文件是否设置了 SGID 位，如果是，则返回 true。|[ -g $file ] 返回 false。|
|-k file|检测文件是否设置了粘着位(Sticky Bit)，如果是，则返回 true。|[ -k $file ] 返回 false。|
|-p file|检测文件是否是有名管道，如果是，则返回 true。|[ -p $file ] 返回 false。|
|-u file|检测文件是否设置了 SUID 位，如果是，则返回 true。|[ -u $file ] 返回 false。|
|-r file|检测文件是否可读，如果是，则返回 true。|[ -r $file ] 返回 true。|
|-w file|检测文件是否可写，如果是，则返回 true。|[ -w $file ] 返回 true。|
|-x file|检测文件是否可执行，如果是，则返回 true。|[ -x $file ] 返回 true。|
|-s file|检测文件是否为空（文件大小是否大于0），不为空返回 true。|[ -s $file ] 返回 true。|
|-e file|检测文件（包括目录）是否存在，如果是，则返回 true。|[ -e $file ] 返回 true。|

### shell 内部命令
* `echo` 输出
* `exec`
* `exit` 退出
* `read` 从标准输入读取一行并且赋值给后面变量
 ````shell
 read var
 read var1 var2 var3 ````
* `expr` 常见的算术运算 `+ ， - ， \* ， / ， %`
  注意，运算符左右两边都需要有空格，否则会视为字符串连接
* `test` 测试结果也常常用来作为判断条件及结果
 ````
 test "$answer" = "yes"
 test $num -eq 18
 test -d temp````
 可以用[]代替test  ， 但需要左右留有一个空格 ，比如
  ```` 
  [ "$answer" = "yes" ]
  if [ $num -eq 18 ]````
 
### 结构性语句
* 条件
 1. `if`语句
 ````
 if...then...fi
 if [exp]
 then [command]
 fi````
 2. `if else` 语句
  ````
 if [exp]
 then [command]
 else [command]
 if````
 3. `case...esac` 语句
 ````
 case [var] in
 [param1])
 [command]
 ;;
 [param2])
 [command]
 ;;
 [paramn])
 [command】
 ;;
 esac
 注意：var只能是字符串型变量  ````


* 循环
 1.	for...do...done
  ````
 for [var] in [list]
 do
  [command]
 done````
 2.	while...do...done
  ````
  while [exp]
  do
  [command]
  done````
 3.	until...do...done
  ````
  until [exp]
  do
  [command]
  done````
 4. break, continue

## Shell 函数
 * 定义格式
 ````
function_name()
{
  command1
  ...
  commandn
}
 
function function_name()
{
  command1
  ...
  commandn
}
 ````
 说明：
  * 1、可以带function fun() 定义，也可以直接fun() 定义,不带任何参数。
  * 2、参数返回，可以显示加：return 返回，如果不加，将以最后一条命令运行结果，作为返回值。 return后跟数值n(0-255
 * 调用函数
 ````
[var]=`function_name [arg1,arg2...]` 
do(fdf,dfdfd)
do fdf fdfdd
function_name [arg1,arg2...]
echo $?````
 说明：
  * 函数返回值在调用该函数后通过 $? 来获得。
  * 所有函数在使用前必须定义。调用函数仅使用其函数名即可。
  * 调用函数时可以向其传递参数。在函数体内部，通过 $n 的形式来获取参数的值，当n>=10时，需要使用${n}来获取参数。

## Shell 文件包含
Shell 文件包含的语法格式如下：
````shell
. filename # 注意点号(.)和文件名中间有一空格
或
source filename````




