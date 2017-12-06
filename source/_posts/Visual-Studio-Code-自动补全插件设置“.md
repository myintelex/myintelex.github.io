---
title: |+
  Visual Studio Code 自动补全插件设置

date: 2017-12-06 11:24:11
categories: 工具使用
tags: [VS Code,IDE]
---
本篇是 VS Code 使用的第一篇，简单介绍下自己对于IDE和编辑器的选择，并说明 VS Code 自动补全插件的使用。
<!--more-->

从开始接触编程到现在，不管是编辑器还是IDE都接触过很多，从一开始的 NotePad 到后来 VIM、Qt Creator，每种工具都有自己的特点，但是使用上总有不顺手的地方。当然IDE和编辑器还是两种不同的东西，但是考虑到毕竟IDE使用的过程中，最多的时候还是在和代码打交道，所以，自己还是希望能够有一个统一的编码平台的。

经过一番比较，最后自己还是选定了 VS Code 作为这个代码编辑工具。本系列就是折腾 VS Code 过程中的一些记录。介绍如何能够使 VS Code 更好的进行 C/C++ 的自动补全。

VS Code 的自动补全现在其实也是有很多版本不同的方案的，目前主流的是以下三个插件：
* 微软爸爸自己的 C/C++ 插件
* C++ Intellisense
* C/C++ Clang Command Adapter

但是综合使用之后，还是使用最方便的微软自己的插件来的方便，不管是 C++ Intellisense 还是 C/C++ Clang Command Adapter 都需要下载额外的环境进行配置。C/C++ 这个插件是微软官方提供的，为 VS Code 提供了 C/C++ 语言的支持。

当下载了这个插件之后，VS Code 会在 C/C++ 项目下的 `.vscode` 文件夹下建立一系列的设置文件。

## 智能补全
为了实现智能补全的功能，需要编辑 `c_cpp_properties.json` 这个文件。将你需要添加的 include 文件夹添加在配置项中。

按照以下步骤操作：
* 找到代码中的一个提示
* 点击小黄灯图标
* 点击 **Edit "includePath" setting**
* 找到你所使用的平台，将头文件路径包含在 **browse** 中的 **path** 中

同样，可以使用 `Ctrl+Shift+P` 键快捷唤出控制台，并输入 `C/Cpp:Edit Configurations` 命令来打开 `c_cpp_properties.json`。

## 代码格式化
可以使用 `Ctrl+Shift+F` 键来格式化全文，或使用 `Ctrl+K Ctrl+F` 键来格式化当前选中内容。

可以在工作区创建 `.clang-format` 文件来定义代码格式，如果没有这个文件，格式化样式通过 `C_Cpp.clang_format_fallbackStyle` 设置来指定。默认的样式为 **Visual Studio**。

## 其他快捷键
可以使用 `F12` 来快速跳转到定义位置，使用 `Alt+F12` 来查看声明。 

使用 `Ctrl+P` 打开命令行，输入 `@` 列出当前文件的符号，或输入 `#` 当前项目的所有符号。 

同样可以使用 `Ctrl+Shift+o` 来直接列出当前文件符号。

## 使用 C/C++ Clang Command Adapter 增强代码补全
微软的插件在自动补全的时候会提供大量的无关的符号，这时候其实C/C++ Clang Command Adapter 这个插件使用柑橘更好。
当然，从名字可想而知，首先需要下载 [clang](http://clang.llvm.org/)。
然后将 clang 设置在环境变量中，或通过设置 `clang.executable` 来指定 clang 的路径。
