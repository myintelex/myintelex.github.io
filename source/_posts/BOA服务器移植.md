---
title: BOA服务器移植
date: 2017-12-20 14:47:26
categories: Linux 应用
tags: [Boa]
---
在很多场合内，我们需要在嵌入式设备中集成 Web 服务器，来方便我们对设备的配置更改。这边文章对嵌入式服务器之——**Boa**服务器的移植和使用进行简单说明。
<!-- more -->
## 原理介绍
BOA是一款小巧的Web服务器，支持CGI通用网关接口技术，特别适合应用在嵌入式系统中。BOA服务器是基于HTTP超文本传输协议的，Web网页是Web服务最基本的传输单元。嵌入式Web服务的工作基于客户机/服务器计算模型，由Web浏览器(客户机)和Web服务器(服务器)构成。

由于嵌入式设备资源一般都比较有限，并且也不需要能同时处理很多用户的请求，因此不会使用Linux下最常用的如Apache 等服务器，而需要使用一些专门为嵌入式设备设计的Web服务器，这些Web服务器在存贮空间和运行时所占有的内存空间上都会非常适合于嵌入式应用场合。典型的嵌入式Web服务器有[Boa](http://www.boa.org)、[thttpd](http://www.acme.com/software/thttpd)、 lighttpd、shttpd、mathopd、minihttpd、appweb、goahead等，它们和Apache等高性能的Web服务器主要的区别在于它们一般是 单进程服务器，只有在完成一个用户请求后才能响应另一个用户的请求，而无法并发响应，但这在嵌入式设备的应用场合里已经足够了。

BOA的工作原理是运行于客户端的浏览器首先要与嵌入式Web服务器BOA端建立连接，打开一个套接字虚拟文件，此文件建立标志着SOCKET连接建立成功然后客户端浏览器通过套接字SOCKET以GET或者POST参数传递方式向Web服务器提交请求，Web浏览器提交请求后，通过HTTP协议传送给Web服务器。Web服务器接到请求后，根据请求的不同进行事务处理，返回HTML文件或者通过CGI调用外部应用程序，返回处理结果。服务器通过CGI与外部应用程序和脚本之间进行交互，根据客户端浏览器在请求时所采用的方法，服务器会搜集客户所提供的信息，并将该部分信息发送给指定的CGI扩展程序，CGI扩展程序进行信息处理并将结果返回给服务器，然后服务器对信息进行分析，并将结果发送回客户端在浏览器上显示出来。

## 移植
### 1、下载源码
BOA官方网址：http://www.boa.org/
从官网下载源码包后解压，进行下一步。

### 2、支持工具下载
在BOA安装的过程中需要用到 `yacc` 和 `lex`，因此，需要安装 `bison` 和 `flex`。执行以下命令：
````
yacc——sudo apt-get install bison
lex——sudo apt-get install flex
````

### 3、修改文件
#### 1、修改 `src/compat.h`
找到
````
#define TIMEZONE_OFFSET(foo) foo##->tm_gmtoff
````
修改成
````
#define TIMEZONE_OFFSET(foo) (foo)->tm_gmtoff
````
 
#### 2、修改 `src/log.c`
如果不需要保存错误日志，可以将错误日志设置为`/dev/null`，或直接注释掉，这时候需要注释以下代码：

````
if (dup2(error_log, STDERR_FILENO) == -1) {
                         DIE("unable to dup2 the error log");
                   }
````

否则会出现错误：
````
log.c:73 unable to dup2 the error log:bad file descriptor
````

### 4、生成
执行`./config`生成 Makefile 文件，执行`make`，生成可执行文件。

## 参数配置
在下载目录下已有一个示例boa.conf，可以在其基础上进行修改。如下：

|内容|配置项|
|---|----|
|1 端口|Port 80
|2 客户端身份|User nobody
|3 客户端组|Group nogroup
|4 错误日志|ErrorLog /var/log/boa/error_log
|5 存取日志|AccessLog /var/log/boa/access_log
|6 HTML根文件|  DocumentRoot /var/www
|7 用户目录|UserDir public_html
|8 预生成目录|DirectoryIndex index.html
|9 生成目录的程序|DirectoryMaker /usr/lib/boa/boa_indexer
|10 请求数量|KeepAliveMax 1000
|11 等待时间|KeepAliveTimeout 10
|12 mimetypes文件|MimeTypes /etc/mime.types
|13 mimetypes类型|DefaultType text/plain
|14 CGI $path变量|CGIPath /bin:/usr/bin:/usr/local/bin
|15 路径别名|Alias /doc /usr/doc
|16 脚本虚拟路径|ScriptAlias /cgi-bin/ /usr/lib/cgi-bin/
