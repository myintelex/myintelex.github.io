---
title: C++学习笔记（三）——C++标准库
date: 2017-03-01T18:14:44.000Z
categories: C++学习笔记
tags:
  - C++
  - STL
  - 标准库
---

C++的标准库的内容在50个头文件中定义。其中最重要的一部分是有关容器类和泛型算法的--------即 STL(Standard Template Library)。

<!-- more -->
包括STL在内的C++头文件大致可以分为10类：

类型         | 头文件名
---------- | ------------------------------------------------------------------------------------
语言支持功能|&lt;cstddef&gt;&lt;limits&gt;&lt;climits&gt;&lt;cfloat&gt;&lt;cstdlib&gt;&lt;new&gt;&lt;typeinfo&gt;&lt;exception&gt;&lt;cstdarg&gt;&lt;csetjmp&gt;&lt;csignal&gt;
流输入/输出 |&lt;iostream&gt;&lt;iomanip&gt;&lt;iOS&gt;&lt;istream&gt;&lt;ostream&gt;&lt;sstream&gt;&lt;fstream&gt;&lt;iosfwd&gt;&lt;streambuf&gt;&lt;cstdio&gt;&lt;cwchar&gt;
诊断功能相关|&lt;stdexcept&gt;&lt;cassert&gt;&lt;cerrno&gt;
定义工具函数|&lt;utility&gt;&lt;functional&gt;&lt;memory&gt;&lt;ctime&gt;
字符串处理|&lt;string&gt;&lt;cctype&gt;&lt;cwctype&gt;&lt;cstring&gt;&lt;cwchar&gt;&lt;cstdlib&gt;
STL----容器|&lt;vector&gt;&lt;list&gt;&lt;deque&gt;&lt;queue&gt;&lt;stack&gt;&lt;map&gt;&lt;set&gt;&lt;bitset&gt;
STL----迭代器|&lt;iterator&gt;
STL----算法|&lt;algorithm&gt;&lt;cstdlib&gt;&lt;ciso646&gt;
数值操作|&lt;complex&gt;&lt;valarray&gt;&lt;numeric&gt;&lt;cmath&gt;&lt;cstdlib&gt;
本地化|&lt;locale&gt;&lt;clocale&gt;


 # 一、IO库

C++对输入输出的处理是通过一组定义在标准库中的类型来处理的。这些类型分别定义在三个头文件中：

- `iostream` 包括 `istream` `ostream` `iostream`
- `fstream` 包括 `ifstream` `ofstream` `fstream`
- `sstream` 包括 `istringstream` `ostingstream` `stringstream` 对 `fstream` 和 `sstream` 的处理和 `iostream` 基本是一致的。需要注意的是：**IO对象是不能进行拷贝或者赋值的**。

## 1.1条件状态

条件状态用来帮助我们访问和操纵流的状态：

函数              | 标志      | 作用
--------------- | ------- | -------------------
bad()           | badbit  | 如果流崩溃返回真
fail()          | failbit | IO操作失败返回真
eof()           | eofbit  | 到达文件结束返回真
good()          | goodbit | 流状态正常返回真
clear()         |         | 清除条件状态，设置为goodbit
clear(flags)    |         | 根据指定的标志位，将对应的条件状态复位
setstate(flags) |         | 根据指定的标志位，将对应的条件状态置位
rdstate()       |         | 返回当前的条件状态

## 1.2输出缓冲

每个输出流都管理一个缓冲区，用来保存程序读写的数据。导致缓冲刷新（即将数据真正的写入到设备文件）的原因有：

- 程序正常结束
- 缓冲区满
- 使用操纵符如 `endl`，来显示的刷新缓冲区
- 使用 `unitbuf` 设置流的内部状态，来清空缓冲区
- **当一个输出流关联到另外一个流时**，当读写关联流时，相应的输出流的缓冲区被刷新

`unitbuf`使用后，所有的输出操作后都会立即刷新缓冲区，此时可以使用 `nounitbuf` 回到正常的缓冲方式。 使用 `tie` 方法可以关联两个流，使用无参版本的 `tie` 会返回当前流管理按的流指针，如果没有关联流则返回空指针。使用带参数的版本绑定相应的流到当前流。

## 1.3 文件输入输出

与标准输入输出流相比， `fstream` 新增了几个操作：

操作                     | 含义
---------------------- | -------------------
fstream fstrm(s)       | 创建fstream，并打开s指定的文件
fstream fstrm(s, mode) | 和上个类似，mode指打开的文件模式
fstrm.open(s)          | 打开s指定的文件
fstrm.close()          | 关闭fastream绑定的文件
fstrm.is_open()        | 返回当前文件是否成功打开

当构造一个fstream对象时，指定了文件，则 `open()` 方法自动被调用，而对象析构的时候， `close()` 方法自动被调用。

上面提到了 fstream 的文件模式：

模式     | 含义            | 备注
------ | ------------- | --------------------------------------
in     | 以只读方式打开       | 只可以对ifstream和fstream对象指定in模式
out    | 以写方式打开        | 只可以对ofstream和fstream对象指定out模式
trunc  | 截断文件          | 只有当out也被设定的时候，才可以设定trunc模式
app    | 每次写操作均定位到文件末尾 | 只要trunc没被设定，就可以设定app模式，且app模式默认打开out模式
ate    | 打开后定位到文件末尾    | 可以和任何搭配
binary | 以二进制方式打开      | 可以和任何搭配

需要注意的是：

1. 以 out 模式打开文件会丢弃已有数据
2. 每次调用 open 都会确定文件模式

## 1.3 string流

string流用于字符串的处理，通常可用于格式化输入输出。

# 二、容器

C++中的容器类包括"顺序存储结构"和"关联存储结构"，前者包括vector，list，deque等；后者包括set，map，multiset，multimap等。若需要存储的元素数在编译器间就可以确定，可以使用数组来存储，否则，就需要用到容器类了。 有一些操作是所有容器都可以使用的：

操作                  | 含义
------------------- | ---------------------------
C c;                | 默认构造函数，构造空容器
C c(c2)             | 构造c2的拷贝c1
C c(b, e)           | 构造c，将迭代器b和e之间的元素拷贝至c
c1 = c2             | 将c1中的元素替换为c2
c1 = ｛a, b, c...｝   | 将c1中的元素替换为列表中的元素
a.swap(b)/swap(a,b) | 交换a和b的元素
c.size()            | 返回c中的元素数目（不支持forward_list）
c.max_size()        | 返回c可以保存的最大元素数目
c.empty()           | 如果c中存储了元素，返回false，否则返回 true
c.begin()           | 返回指向c的第一个元素的迭代器
c.end()             | 返回指向c的超尾的迭代器

## 2.1 顺序容器

顺序容器提供了快速顺序访问元素的能力，但是对与增删操作和非顺序访问操作有所性能折中。标准库中的顺序容器有：

容器名          | 定义
------------ | -------------------
vector       | 可变大小数组
deque        | 双端队列
list         | 双向链表
forward_list | 单向链表
array        | 固定大小数组
string       | 与vector相似，但专门用来保存字符

顺序容器共有的方法有:

- `emplace(C++11 新增)`， 插入一个对象
- `insert`, 插入一个拷贝
- `resize(n)`, 重新设置大小
- `assign`，替换
- `erase`， 删除
- `clear`，清除所有元素
- `front`，返回第一个元素

`vector`, `forward_list`, `list`, `deque` 共有的方法有：

- `back`， 返回最后一个元素
- `push_back`， 插入到末尾
- `pop_back`， 删除最后一个元素
- `emplace_back`， 追加到末尾
- `push_front`，插入到队首
- `pop_front`，删除队首
- `emplace_front`，追加到队首
- `at`，返回相应的对象

`vector` 独有的操作有：

- `capacity`， 返回能够存储的总量
- `reserver`， 提醒至少需要存储相应个元素的空间

`list` 独有的操作有：

- `splice`， 移动指定的元素
- `remove`， 删除相应值的所有元素
- `unique`，删除连续的相同元素，仅保留一个
- `merge`，合并两个链表，并排序
- `sort`，排序
- `reverse`,反转

## 2.2 关联容器

关联容器和顺序容器不同，关联容器中的元素是按照元素关键字来保存和访问的。关联容器有两个主要类型： `map` 和 `set`。

- `map` 中的元素是一些关键字-值的对，关键字起到索引的作用，而值为对应索引的数据
- `set` 中的元素只包含一个关键字。

除了 `map` 和 `set`， 还有对应的 `multimap`和 `multiset`。此外， c++ 还定义了相应的无序集合 `unordered _map`等。

相应的关联容器的操作有

- `key_comp`, 返回比较对象
- `value_comp`，返回 `value_comp` 对象
- `insert`， 插入
- `emplace`， 插入对象
- `erase`， 擦除
- `clear`， 清空
- `find`， 寻找相应的键
- `count`， 返回相应键的数量
- `lower_bound`, 返回不小于指定键的元素迭代器
- `upper_bound`， 返回大于k的元素迭代器
- `equal_range`， 返回第一个成员为 `a.lower_bound(k)` 第二个成员为 `a.upper_bound(k)` 的值对
- `operator`， 返回一个指向yu指定的键关联的值的引用

无序容器的操作有

- `hash_function` 返回使用的哈希函数
- `key_eq` 返回创建时使用的键值相等谓词
- `bucket_count`， 返回桶数

# 三、泛型算法

对于算法，算法本身不执行容器的操作，它们只会运行在迭代器之上，执行迭代器的操作。所以：算法永远不会改变底层容器的大小。

## 3.1 算法分类

1. **查找对象的算法** 相应的算法有 `find`， `count`， `adjacent_find`, `search_n`, `search`, `find_first_of`, `find_end`
2. **其它只读算法** `for_each`, `mismatch`, `equal`
3. **二分搜索算法** `lower_bound`， `upper_bount`， `equal_range`， `binary_search`
4. **写容器操作** `fill`， `generate`， `copy`， `move`， `transform`， `replace_copy`， `merge`， `iter_swap`， `swap_ranges`， `replace`， `copy_backward`， `move_backward`， `inplace`
5. **划分算法** `is_partitionedis`， `partitioned_copy`， `partition_point`， `stable_partition`， `partition`
6. **排序算法** `sort`, `is_sorted`, `partial_sort`, `partial_sort_copy`, `nth_element`
7. **通用重排算法** `remove`， `unique`， `rotate`， `reverse`， `random_shuffle`， `shuffle`
8. **排列算法** `is_permutation`, `next_permutation`, `prev_permutation`
9. **有序序列的集合算法** `set_inion`, `set_intersection`, `set_symmetric_difference`
10. **最小值和最大值** `min`， `minmax`， `max`，
11. **数值算法** `accumulate`， `inner_product`， `partial_sum`， `adjacent_difference`， `itoa`

## 3.2 lambda

一个lambda表示一个可调用的代码单元。他的形式如下： [capture list] (parameter list) -> return type {function body} `capture list` 是lambda所在的函数中定义的局部变量的列表 `return type` `parameter list` `function body` 表示相应的返回值，参数列表，函数体

lambda可以忽略参数列表和返回类型，但是必须包含捕获列表和函数体。

# 四、智能指针

通常我们使用 `new` 和 `delete` 来管理动态内存的。但是，这样的使用很容易出现问题：我们有可能会忘记释放内存造成内存泄漏；或者我们有可能使用了已释放的指针，导致非法访问。因为C++中定义了智能指针。 智能指针其实也是模版，C++中定义了三个相关的类：

- `shared_ptr` 允许多个指针指向同一个对象
- `unique_ptr` 仅允许一个指针指向对象
- `weak_ptr` 弱引用，指向 `shared_ptr` 所管理的对象

智能指针的相应操作有：

函数名                                  | 含义
------------------------------------ | --------------------------------
shared_ptr <T> sp / unique_ptr<T> up | 空智能指针，可以指向类型为T的对象
p                                    | 可以把智能指针本身作为判断，如果p指向了一个对象，则返回true
*p                                   | 解引用，获得指向的对象
p->mem                               | 等价于(*p).mem
swap(p, q) / p.swap(q)               | 交换
make_shared <T>(args)                | 返回一个shared_ptr
shared_ptr<T>p (q)                   | p 是 q 的拷贝
p = q                                | 拷贝
p.use_count()                        | 返回与p共享对象的智能指针对象
p.unique()                           | 如果p.use_count()为1，返回true

当智能指针创建之后，每对shared_ptr的拷贝和赋值，都会使 shared_ptr 的引用计数加一。 当一个shared_ptr被销毁时，shared_ptr 的引用计数会递减。 当动态内存的引用计数为0时， shared_ptr 的析构函数会自动销毁所管理的对象。 我们可能会在下面的三种情况下使用智能指针：

- 程序不知道自己需要使用多少对象
- 程序不知道所需对象的准确类型
- 程序需要在多个对象间共享数据
