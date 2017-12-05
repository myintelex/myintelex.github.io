---
title: 在windows下打出tar.gz的尝试
date: 2016-11-21 16:20:46
categories: 工作记录
tags: [tar,7z,gz]
---
作为一名嵌入式开发人员，Linux是必须使用的，但是不可避免的有些时候手头上没有装有Linux的电脑，或者不想去打开虚拟机。尤其是当你仅仅只是需要打个tar包的时候。这时候当然你可以使用7z的桌面版程序，就可以解决这种简单的需求。但是目前我遇到了一个这样的问题：公司需要对外提供一个软件打包工具，而用户通常是没有Linux系统的。这时候需要在程序中打出tar.gz的压缩包。

<!--more-->
# 第一种方法：使用gnu tar for windows

和其他很多GNU的软件一样，tar提供了Windows的版本，可以从[gnu tar for windows](http://gnuwin32.sourceforge.net/packages/gtar.htm "gnu tar for windows")这里下载，官方分别提供了二进制可执行文件、文档、源代码的安装包和压缩包，可以根据需要进行下载。
因为时间要求比较紧，所以我直接使用了调用二进制文件的方式进行打包，使用命令`tar.exe cvf xxx.tar file`即可简单的打出tar包，需要注意的是，使用tar.exe的同时需要提供`libintl-2`和`libiconv-2 `两个文件。
但是遗憾的是，使用tar.exe仅仅只能打出tar包，对与gz暂时无法提供：
>The Win32 port can only create tar archives, but cannot pipe its output to other programs such as gzip or compress, and will not create tar.gz archives; you will have to use or simulate a batch pipe. BsdTar does have the ability to direcly create and manipulate .tar, .tar.gz, tar.bz2, .zip, .gz and .bz2 archives, understands the most-used options of GNU Tar, and is also much faster; for most purposes it is to be preferred to GNU Tar. 

# 第二种方法：使用7z
作为很好的一种压缩软件，7z拥有很多的用户，同时7z也是一款开源软件，当然也提供了源代码和相应的命令行软件。从[这里](http://www.7-zip.org/sdk.html "7z")可以下载相应的文件。
同样使用调用二进制文件的方式，7z的使用很简单。命令如下：

	7z.exe a archive files

7z可以直接通过archive 的后缀判断需要生成的压缩包的类型。使用7z打出tar.gz需要以下两步：

	7z.exe a archive.tar files
	7z.exe a archive.tar.gz archive.tar

这样圆满的解决了我遇到的问题。当然这是一种很取巧的方式，在程序中更提倡使用的还是使用源代码调用相应的方法。之后，我会继续研读tar和7z的源码。