# 文档说明

本文档研究开源项目 ： https://sysprog21.github.io/lkmpg/

并记录笔记

# 1.介绍

## 1.5 内核里面有什么modules

去查看你内核里面有什么modules

```shell
sudo lsmod
```

模块都存放在/proc/modules

```shell
sudo cat /proc/modules
```

可以通过管道+grep来找到对应的模块fat

```shell
sudo lsmod | grep fat 
```

# 2. 头文件

当前hp的内核头文件：linux-headers-5.11.0-38-generic - Linux kernel headers for version 5.11.0 on 64 bit x86 SMP

# 3. 例子

都在文件夹 `/mnt/share/hjy_code/lkmpg/examples`

# 4. Hello World

## 4.1 最简单的modules

注意，复制到vscode时，tab会变成4个空格。

这里解释了一下，用户空间的环境变量与sudoer的环境变量有所不一样的问题。

成功编译出.ko文件。

查看ko文件信息：

```shell
modinfo hello-1.ko
```

个人猜测，这原理跟readelf 读elf文件类似，ko文件格式应该也是遵循某种规律。

此时没有加载hello之前

```shell
sudo lsmod | grep hello
```

发现什么都没有，加载一下

```shell
sudo insmod hello-1.ko
```

重新执行

```shell
sudo insmod hello-1.ko
```

可以看到打印：

```shell
hello_1                16384  0
```

你可以删除

```shell
sudo rmmod hello_1
```

可以通过查看日志去看刚才发生了什么

```shell
sudo journalctl --since "1 hour ago" | grep kernel
```

至此，我们已经知道了modules的一些基本概念，创建，编译，安装，移除。

内核模块必须要有2个函数：

- "start" 初始化函数 ， 定义为 ：init_module() ，当模块加载（insmod）的时候调用。
- "end"清理函数，定义为：cleanup_module() ，

其实，我们可以使用任意的名字来定义开始和结束的函数。

典型的，init_module() ， 会注册或者替换内核的处理函数，cleanup_module() ， 会做跟init_module()相反的事情，这样来保证模块被安全卸载。

关于一些头文件的包含

```
#include <linux/init.h> /* Needed for the macros */ 
#include <linux/kernel.h> /* Needed for pr_info() */ 
#include <linux/module.h> /* Needed by all modules */ 
```

module开发需要注意一些问题：

- 我们应该使用tab ，而不是空格....（这是为什么？）
- printk , 通常包含一些优先级，KERN_INFO , KERN_DEBUG 。也可以用 pr_info , pr_debug,替代，这两个函数在文件[include/linux/printk.h](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/tree/include/linux/printk.h)中定义。
- 关于编译。内核模块的编译方式与常规用户空间应用程序略有不同。以前的内核版本要求我们非常关心这些设置，这些设置通常存储在 Makefile 中。尽管按层级组织，许多冗余设置在子级 Makefile 中积累，使它们变得庞大且难以维护。幸运的是，有一种新方法可以做这些事情，称为 kbuild，外部可加载模块的构建过程现在完全集成到标准内核构建机制中。要了解有关如何编译不属于官方内核的模块的更多信息（例如您将在本指南中找到的所有示例），请参阅文件https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/tree/Documentation/kbuild/modules.rst ，https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/tree/Documentation/kbuild/makefiles.rst

小练习：

如果这个初始化失败，返回负数，会怎么样？

答：dmesg还是能看到返回-1之前的log，但是模块加载失败，这需要我们去思考系统怎么看待这个返回值。（带着这个疑问继续阅读）

我们在内核源码中去搜，可以找到这样一个结构体：

`static struct errormap errmap[]`

这个结构体记录了各种不同返回值对应的报错。比如：

```
	{"Operation not permitted", EPERM},
	{"wstat prohibited", EPERM},
	{"No such file or directory", ENOENT},
	{"directory entry not found", ENOENT},
	{"file not found", ENOENT},
	{"Interrupted system call", EINTR},
	{"Input/output error", EIO},
```

我们可以根据不同的初始化失败，来选择返回不同的错误值，从而给调试提供帮助。

## 4.2 hello and goodbye

我们可以用

```
module_init(hello_2_init); 
module_exit(hello_2_exit); 
```

来注册初始函数和退出函数，这应该是修改这个module的函数指针。

真实的makefile

```
obj-y				+= mem.o random.o
obj-$(CONFIG_TTY_PRINTK)	+= ttyprintk.o
obj-y				+= misc.o
obj-$(CONFIG_ATARI_DSP56K)	+= dsp56k.o
obj-$(CONFIG_VIRTIO_CONSOLE)	+= virtio_console.o
obj-$(CONFIG_MSPEC)		+= mspec.o
obj-$(CONFIG_UV_MMTIMER)	+= uv_mmtimer.o
obj-$(CONFIG_IBM_BSR)		+= bsr.o
```

是-y, 还是-m, 由make menuconfig 来决定。其修改的是.config里面的内容。

当在根目录make的时候，这个makefile会读.config里面的内容。

## 4.3 __init 和 __exit 宏

__init宏是用来标记这个函数一旦初始化完毕，就直接把这个函数所占用的内存给释放掉。因为初始完毕，函数就没有任何必要存在于系统中了。

__initdata 这个宏也是类似的，但这是针对初始化变量。

如果模块是编译到内核里面的，那么用__exit宏，可以让我们的卸载函数变得没有作用，但如果是编译成模块，那么就需要卸载函数。

当我们开机的时候，看到释放没有使用的内核存储，其实就是释放__init , __initdata等函数的内存。

## 4.4 许可证和模块文档

这些宏定义在文件：[include/linux/module.h](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/tree/include/linux/module.h).

```
MODULE_LICENSE("GPL"); 
MODULE_AUTHOR("LKMPG"); 
MODULE_DESCRIPTION("A sample driver"); 
```

## 4.5 传递命令行参数给模块

`module_param()`  ， `module_param_array()`  ，`module_param_string()`  宏用来处理传入参数，而不是argc/argv

`MODULE_PARM_DESC`  可以对传入变量进行说明， 这个说明。

```shell
$ sudo insmod hello-5.ko mystring="bebop" myintarray=-1
$ sudo dmesg -t | tail -7
myshort is a short integer: 1
myint is an integer: 420
mylong is a long integer: 9999
mystring is a string: bebop
myintarray[0] = -1
myintarray[1] = 420
got 1 arguments for myintarray.

$ sudo rmmod hello-5
$ sudo dmesg -t | tail -1
Goodbye, world 5

$ sudo insmod hello-5.ko mystring="supercalifragilisticexpialidocious" myintarray=-1,-1
$ sudo dmesg -t | tail -7
myshort is a short integer: 1
myint is an integer: 420
mylong is a long integer: 9999
mystring is a string: supercalifragilisticexpialidocious
myintarray[0] = -1
myintarray[1] = -1
got 2 arguments for myintarray.

$ sudo rmmod hello-5
$ sudo dmesg -t | tail -1
Goodbye, world 5

$ sudo insmod hello-5.ko mylong=hello
insmod: ERROR: could not insert module hello-5.ko: Invalid parameters
```

对于数组，传入多少个参数，是可以知道的，不需要显式的说明。

## 4.6 模块跨越多个文件

有时候，我们会把模块文件分成多个源文件

```
obj-m += startstop.o 
startstop-objs := start.o stop.o
```

在makefile中，这样编写，可以让模块分布在多个文件之中。


## 4.7 为预编译内核构建模块

讲怎么去编译内核代码，已写过类似文档：http://confluence.midea.com/pages/viewpage.action?pageId=237921063


# 5 预赛


## 5.1 模块是怎么开始和结束的？

应用程序，是从main函数开始，执行很多的指令，完成后，退出。

内核模块的工作方式有一些不同。

一般从init_module开始，这是模块的入口函数，它告诉内核这个模块提供什么样的功能，当内核需要调用这些函数的时候，就去运行这里的函数。完成初始化之后，直到模块被调用，内核函数不会做任何事情。

所有的模块都是通过调用cleanup_module 或者module_exit退出函数，有点像析构函数，把init时的操作复原。

每一个模块都必须要有一个入口函数和退出函数，



## 5.2 模块可以用的函数

程序员经常使用非他们定义的函数，比如printf，就是libc的东西，这些库，直到连接时，都不会载入到程序。

与普通的程序不同，模块只能执行内核提供的函数，因为在insmod的时候，我们才加载。那么我们只能使用内核提供的库函数。

那么这些函数符号有哪些呢？我们可以通过cat /proc/kallsyms 来读出来



库函数与系统调用：

库函数是高层的函数，完全是跑在用户空间的，提供一些更方便的接口给程序员去调用系统调用。系统调用 代表用户 运行在内核态 ， 由内核提供。

printf看上去是通用的打印函数，其实是格式化输入数据，然后把数据丢给底层函数系统调用write() ， 这会发送数据给标准输出。


可以用strace 去追踪系统调用。

您甚至可以编写模块来替换内核的系统调用，我们很快就会这样做。破解者经常利用这种东西做后门或木马，但是你可以自己写模块来做更多的善事，比如让内核写Tee hee，痒痒的！每次有人试图删除您系统上的文件时


## 5.3 用户空间与系统空间

内核可以访问所有的资源，显卡，内存，硬盘。

程序经常争夺资源，出现并发问题。内核需要保持这些操作有序进行，而不是给用户随意的访问这些资源。

为此，CPU可以运行在不同的模式。

不同模式给与你不同程度的自由去做你想做的事情。

i386有4种不同的模式，叫做rings ， unix只使用2个rings，highest ring (rings 0 ， supervisor mode) 可以做任何事，lowest ring 我们称之为用户模式。

这种不同的模式，在CPU的指令集里面是有详细的说明的。不同模式的系统调用的指令码是不一样的，至于程序被编译出什么样的指令码，实际上是有操作系统决定的。

在用户模式使用库函数，
