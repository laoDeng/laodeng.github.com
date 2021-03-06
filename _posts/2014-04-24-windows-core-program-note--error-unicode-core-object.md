---
layout: post
title: "Windows核心编程笔记：Error、Unicode、内核对象"
description: ""
category: "Tech"
tags: ["Windows","C/C++"]
---
{% include JB/setup %}

Windows核心编程让我体会到一个完全不同的Windows，通过对书的阅读和理解，使我对操作系统编程和操作系统运行的方式有了更深入的理解。目前先要掌握Windows核心编程的思路，然后再进一步去看Linux内核的相关书籍，做到对两种系统心中有数，能够更快的上手各种平台的编程。

对程序错误的处理：Error
=======================

Windows函数常用的返还类型
-------------------------

* `VOID`：一般不使用，即表示一定要返还一个值，才符合Windows编程规范。
* `BOOL`：程序运行失败返回`False`（0）；运行成功返回`True`（1）。
* `HANDLE`：创建不成功返回`NULL`；创建成功返回HANDLE，但也有可能返回`INVALID_HANDLE_VALUE`。
* `PVOID`：失败返回`NULL`；成功返回`VOID`，标识数据块的内存地址。
* `LONG/DWORD`：具体查询PlatformSDK。

Microsoft编译了一个错误代码列表 ，为每一个错误代码均分配了一个32位的代码。

Windows错误的表达方式
---------------------

在Windows编程中主要使用`DWORD GetLastError()`函数来指明发生的错误。

ERROR主要由以下几种表达形式：

* 消息ID
* 消息文本
* 一个号码

一般情况下，当运行失败时，就立即调用`GetLastError()`函数来获取发生的错误，并作相应处理。
在Visual Studio中可以通过Watch窗口观察`@err,hr`变量来捕获错误信息。

往往通过`GetLastError()`函数获取的仅仅是错误的代码，还要用'FormatMessage'函数来对错误代码转换成文本描述，方便判断。

Windows采用的编码：Unicode
==========================

DBCS表示双字节字符集，用来扩充ANSI，但是有的字符是单字节，有的字符是双字节，导致处理起来非常复杂。
所以Windows采用了一种完全双字节的编码，Unicode编码。
它既是宽字节字符集，每一个字符都采用16位表示。

C运行期库对Unicode的支持
------------------------

C运行库就采用`wchar_t`数据类型来表示Unicode类型。

并且重定义了一系列的字符串处理函数：

* `strcat` -> `wcscat`
* `strchr` -> `wcsxhr`
* `strcmp` -> `wcscmp`
* `strcpy` -> `wcscpy`
* `strlen` -> `wcslen`

`wcs`：即为宽字符串（Wide Character String）的缩写。

`TChar.h`：ANSI/Unicode通用库。

用大写字母`L`来修饰字符串，告诉编译器，字符串作为Unicode字符串来编译。
最好是采用`_TEXT`来修饰字符串，这样会自动的判断是否采用Unicode来编译字符串。

Windows对Unicode的支持
----------------------

Windows定义的Unicode数据类型：

* `WCHAR`：Unicode字符
* `PWSTR`：指向Unicode字符指针
* `PCWSTR`：指向const字符串的指针

Windows的函数名结尾用大写`W`表示支持Unicode的函数，用大写`A`表示支持ANSI的函数。

Windows提供的字符串函数

* `lstrcat`：字符串连接函数
* `lstrcmp`：区分大小写比较字符串
* `lstrcmpi`：不区分大小写比较字符串
* `lstrcpy`：将字符串拷贝到内存中某位置
* `lstrlen`：返回字符串的长度（按字符数来计量）

不同于C运行库的字符串比较函数，Windows库的字符串比较函数按语义比较字符串，而不是简单的数值比较。

符合ANSI和Unicode应用程序
-------------------------

* 文本视为字符数组，而不是`chars`数组
* 通用数据类型（`TCHAR`、`PTSTR`）来表示文本、字符串
* 显示数据类型（`BYTE`、`PBYTE`）字节和字节指针来表示数据缓存
* TEXT宏用于原义字符和字符串
* 执行全局性替换（`PTSTR`替换`PSTR`）
* 字符中传递缓存大小而不是字节
* 为字符串分配一个内存块，拥有该字符串中的字符数目，记住要按字节数来分配内存

内核对象
========

内核对象是可以供系统和应用程序使用，用来管理各种各样的资源（进程、线程、文件等）。
主要包括符号对象、事件对象、文件对象、文件映射对象、I/O完成端口对象、作业对象、信箱对象、互斥对象、管道对象、进程对象、信标对象、线程对象、互斥对象、等待计时器对象等。

每个内核对象（只能由内核访问，应用程序无法直接操作）只是内核分配的一个内存块，并只能由该内核访问，程序只能通过Windows提供的一组函数操作。
该内存块只是一个数据结构，用以维护该对象的各种信息，信息包括公有信息，也有特性信息。
程序用以鉴别内核对象的数据结构既为句柄，句柄只用于标识内核对象。

除了内核对象，Windows还有很多其他的对象，比如说GDI（图形设备接口对象）、菜单、窗口、鼠标光标、刷子、字体等。

进程的内核对象句柄表
--------------------

进程被初始化时，系统要为其分配一个句柄表（只用于内核对象）。
当创建内核对象时，内核对进程的句柄表进行扫描，找出一个空项，存入创建的内核对象信息（内核对象数据结构的内存地址、掩码、是否继承）。
最后返回与进程相关的句柄，这些句柄可以被在相同进程中运行的任何或所有线程加以使用。

内核对象调用失败有两种可能的返回值：

* `NULL`
* `INVALID_HANDLE_VALUE`

关闭内核对象使用函数`CloseHandle`，直接对句柄进行关闭。
当进程终止运行时，操作系统确保该进程使用的任何资源或全部资源被释放。

跨越进程边界共享内核对象
------------------------

不同进程中运行的线程需要共享内核对象：

* 文件映射对象使在同一台机器上运行的两个进程之间共享数据块。
* 邮箱指定的管道，使得应用程序能够在联网的不同机器上运行的进程之间发送数据块。
* 互斥对象、信标、事件使得不同进程中的线程能偶同步他们的连续运行。

有三种机制可以共享单个内核对象。

###对象句柄的继承

进程具有父子关系时，才能使用对象句柄的继承性，为子进程赋予对父进程的内核对象的访问权。
分以下几步：

1. 创建内核对象时，希望对象的句柄是个可继承的句柄。
2. 生成子进程`CreateProcess`，系统遍历父进程句柄表，加入新句柄表。

继承只有在生成子进程时才能使用。
子进程为了确定它获得的内核对象句柄值。
最常用的是将句柄值作为一个命令行参数传递给子进程，该子进程初始化代码对命令行进行分析。
父进程和子进程的共享内核对象的句柄值是相同的。
将已继承的内核对象句柄值从父进程传递给子进程。

三种父进程传递子进程句柄值的方法：

* 命令行参数
* 消息发送
* 环境变量

###命名对象：给对象命名

创建内核对象时为其赋予一个对象名字，用以标识内核对象。
再使用`Create*`或`Open*`函数，在别的进程中打开该内核对象，调用进程的句柄表被更新，对象的使用计数被递增。

但是系统不会对内核对象名字进行查重处理，所以需要唯一的表示字符串来命名内核对象。

###复制对象句柄

利用`DuplicateHandle`函数，取出一个进程的句柄表中的项目，并将该项目拷贝到另一个进程的句柄表中。
它一系列参数中，`hSourceProcessHandle`和`hTargetProcessHandle`表示进程句柄，每当系统中启动一个新的进程会创建一个进程内核对象。
`hSourceHandle`表示任何内核对象句柄，必须与`hSourceProcessHandle`进程相关。

同继承一样，目标进程并不知道新的内核对象被共享。必须使用窗口消息机制或IPC机制通知进程。

内核对象总结
------------

1. 内核对象即系统的各种资源，但不同于图形设备接口对象（GDI）。

2. 内核对象由句柄标识，且只能由操作系统提供的接口函数操作。

3. 每个进程有自己的句柄表来维护其所能使用的内核对象。

4. 内核对象有三种共享方式

	* 进程继承：
		父进程建立的子进程可以将一部分内核对象继承给子进程，但子进程并不知道有内核对象被父进程传递进来，所以有三种机制通知子进程。
		实质是将父进程的内核对象句柄复制到子进程中（在句柄表相同位置）。
	* 命名对象：
		为内核对象赋上一个名字来保证其创建。
		别的进程通过名字来打开或创建该对象。
		但该方法内核对象没有防重名机制，所以要取唯一值保证正确获取内核对象。
	* 复制对象句柄：
		主动将某一对象句柄复制到另一进程中，但也没有通知机制，只有通过消息知会进程已复制。