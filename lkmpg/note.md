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

在用户模式使用库函数，但这些库函数实际上会调用一些系统调用。

## 5.4 命名空间

当你写C语言，你使用变量来帮助读者阅读。如果使用全局变量，那么对其他文件来说也是全局变量，这会导致麻烦。大型项目需要保证命名唯一。

linux模块化编程，最好的办法是定义时，使用static，使用一些词头来进行区分，所有的内核词头都是小写的。

如果你确实不想使用static , 那么你可以使用一张符号表，然后向内核去注册。

在文件：/proc/kallsyms 中，拥有着内核知道的所有符号，你可以访问你的模块，只要他们共享内核代码空间。

## 5.5 代码空间

[内存管理的书： Understanding The Linux Kernel](https://www.oreilly.com/library/view/understanding-the-linux/0596005652/)

内存管理是非常复杂的项目。

但是我们是需要知道一些事实，当我们写一个真正的module的时候。

当进程创建的时候，内核会分配一部分真实内存空间给到进程使用，包括存 执行代码，变量，栈，堆。有虚拟地址空间，

内核也有内存空间，模块代码可以动态的加载和删除，模块跟内核使用相同的代码空间，而不是自己单独的代码空间。如果你写的代码出现段错误， 那么内核就出现段错误，如果你越界，那么可能会导致非常严重的后果。

单片内核：模块跟内核代码在一起。

微内核：模块有自己的地址空间（Fuchsia）

## 设备驱动

有一类模块叫设备驱动，为硬件（例如串口）提供功能。驱动开发者为应用程序提供跟硬件沟通的桥梁。

crw-------   1 root root    241,   0 1月   4 08:54 nvme0
brw-rw----   1 root disk    259,   0 1月   4 08:54 nvme0n1
brw-rw----   1 root disk    259,   1 1月   4 08:54 nvme0n1p1
brw-rw----   1 root disk    259,   2 1月   4 08:54 nvme0n1p2

逗号区分主设备号，和次设备号。每一个驱动都注册为一个主设备号，所有的带有同样主设备号的设备文件，都被同一个驱动控制。

主设备号：用来说明用什么驱动

次设备号：用来区分同一驱动的不同设备。

也就是一个区分类型，一个定位具体设备。

设备分为2种设备：字符设备，块设备。

两者的区别在于：块设备为请求提供一个缓冲区，因此，块设备可以选择最好的执行顺序去响应不同的请求。

这对存储设备是非常重要的，因为读写一些靠得更近的内容是更快一些的。

还有一个区别：块设备只能接收块大小的输入输出，字符设备则可以是任意大小。

可以通过查看文档来确定哪些设备文件已经被使用了：[Documentation/admin-guide/devices.txt](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/tree/Documentation/admin-guide/devices.txt).

当系统安装了之后，所有设备文件都是通过mknod来创建的。

`mknod /dev/coffee c 12 2`

设备文件也不一定要放在dev，在测试时可以放在当前文件夹。

系统不关系次设备号，只有驱动本身会关心设备号，因为他要通过设备号去区分不同的硬件。

这里讲的硬件是抽象硬件，比如一个硬盘，被系统分区为2块盘，那么在系统看来就是2个抽象硬件。

# 6 字符设备驱动

## 6.1 file_operation 结构体

**struct** file_operations 中没有实现的函数都会被标记为NULL。

从 Linux v3.14 开始，通过使用 f_pos 特定锁来保证读、写和查找操作是线程安全的，这使得文件位置更新成为互斥。因此，我们可以安全地实现这些操作而无需不必要的锁定

## 6.2 文件结构

一个file是内核等级的结构，从不出现在用户空间

```cpp
struct file {
	union {
		struct llist_node	fu_llist;
		struct rcu_head 	fu_rcuhead;
	} f_u;
	struct path		f_path;
	struct inode		*f_inode;	/* cached value */
	const struct file_operations	*f_op;

	/*
	 * Protects f_ep, f_flags.
	 * Must not be taken from IRQ context.
	 */
	spinlock_t		f_lock;
	enum rw_hint		f_write_hint;
	atomic_long_t		f_count;
	unsigned int 		f_flags;
	fmode_t			f_mode;
	struct mutex		f_pos_lock;
	loff_t			f_pos;
	struct fown_struct	f_owner;
	const struct cred	*f_cred;
	struct file_ra_state	f_ra;

	u64			f_version;
#ifdef CONFIG_SECURITY
	void			*f_security;
#endif
	/* needed for tty driver, and maybe others */
	void			*private_data;

#ifdef CONFIG_EPOLL
	/* Used by fs/eventpoll.c to link all the hooks to this file */
	struct hlist_head	*f_ep;
#endif /* #ifdef CONFIG_EPOLL */
	struct address_space	*f_mapping;
	errseq_t		f_wb_err;
	errseq_t		f_sb_err; /* for syncfs */
} __randomize_layout
  __attribute__((aligned(4)));	/* lest something weird decides that 2 is OK */


```

这跟libc里面的FILE是不一样的。

这个FILE是从来不会出现在内核中的，这个FILE也存在一定的误导，因为它代表的是一个抽象的打开文件，这跟硬盘中的文件又是完全不一样的。

## 6.3 注册一个设备

写的设备文件要放在/dev文件夹中。

在系统中添加一个驱动，意味着在内核中注册他们。这等价于注册主设备号在模块初始化的时候，

我们一般通过函数：`register_chrdev` 完成这件事。

```
int register_chrdev(unsigned int major, const char *name, struct file_operations *fops);
```

如果返回负数，意味着注册失败。

name 意味着 你可以在/proc/devices 中可以看到。

在你使用一个设备号之前，你应该避免冲突，应该查：[Documentation/admin-guide/devices.txt](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/tree/Documentation/admin-guide/devices.txt)

但其实这也是不科学的，我们其实可以让内核自动的生成一个动态主设备号。

只要传入0就可以。

缺点是您不能提前制作设备文件，因为您不知道主设备号是什么

有几种方法可以做到这一点。首先，驱动程序本身可以打印新分配的编号，我们可以手动制作设备文件。其次，新注册的设备在/proc/devices中会有一个入口，我们既可以手工制作设备文件，也可以编写一个shell脚本来读取文件并制作设备文件。第三种方法是我们可以让我们的驱动程序在注册成功后使用 device_create 函数创建设备文件，并在调用 cleanup_module 期间使用 device_destroy

但是， register_chrdev() 将占用与给定专业相关的一系列次要编号。减少字符设备注册浪费的推荐方法是使用 cdev 接口

较新的接口通过两个不同的步骤完成字符设备注册。首先，我们要注册一个设备号的范围，可以用 register_chrdev_region 或者 alloc_chrdev_region 来完成

```
int register_chrdev_region(dev_t from, unsigned count, const char *name); 
int alloc_chrdev_region(dev_t *dev, unsigned baseminor, unsigned count, const char *name);
```

alloc_chrdev_region 在你使用动态设备号的时候去使用。

初始化：

```
void cdev_init(struct cdev *cdev, const struct file_operations *fops);
```

```
int cdev_add(struct cdev *p, dev_t dev, unsigned count);
```

初始化完成之后，通过cdev_add把设备添加到系统中。第九章有具体的描述。

## 6.4 注销一个设备

我们不能随意注销一个设备。

我们一般通过返回值来告诉函数不能被调用。

但是模块释放函数，返回值是void，因此没有办法这么做。

但是，有一个计数器记录了有多少个进程在使用这个模块。

可以通过2个命令来查看：

```shell
cat /proc/modules
sudo lsmod
```

模块释放函数不需要去确认是否有人在调用，因为系统调用sys_delete_module会帮助你。

我们怎么去查看当前这个模块被多少人调用呢？可以通过一组api完成，但是不建议直接去访问和操作。

```shell
try_module_get(THIS_MODULE) : Increment the reference count of current module.
module_put(THIS_MODULE) : Decrement the reference count of current module.
module_refcount(THIS_MODULE) : Return the value of reference count of current module.
```

如果这个数字有问题，那么就会出现无法卸载这个驱动，那么这个时候就要重启了。

## 6.5 chardev.c

在多线程环境，没有任何保护的话，并发访问同样的内存可能会导致竞争条件，这不会提升性能。

在内核模块，这个问题可能会发生，当多个实例访问共享资源。

因此，一个解决办法是强制独占访问，我们使用原子操作比较和交换来管理状态（**CDEV_NOT_USED** ， **CDEV_EXCLUSIVE_OPEN**）

，去决定这个文件是被其他人打开还是关闭。12章还会重点介绍这个并发的问题。

这里提供了一个demo程序，我们可以通过阅读demo程序以及其注释来理解上文所提及的内容。

## 6.6 为多个内核版本编写模块

系统调用是内核向进程显示的主要接口，通常在不同版本中保持相同

系统大多向后兼容，在大多数情况，设备文件会保持一样，另一方面，内核内部的接口可以在两个版本之间有变化。

有几个宏，可以帮助我们去完成对不同内核版本的区隔。

`LINUX_VERSION_CODE`

`KERNEL_VERSION`

这些宏满足以下条件。

In version **a.b.c** of the kernel, the value of this macro would be ![216a+ 28b+ c  ](https://sysprog21.github.io/lkmpg/lkmpg-for-ht0x.svg)

# 7 虚拟文件系统

在Linux，有一个额外的机制为内核和内核模块发信息给进程——虚拟文件系统。

最初设计是去允许很容易的访问进程的消息。它现在被内核的每一位都使用，这有一些有趣的事情要报告，例如/proc/modules提供modules列表。/proc/meminfo 获取内存使用统计

使用虚拟文件系统的方法跟我们的设备文件很像，包括那些指向处理函数的操作。

比如：read,init_module ,cleanup_module

proc只存在于内存，普通的文件有inode，inode告诉系统设备在哪个位置，包含一些文件信息，比如阅读权限。

/proc/下的文件， 如果被打开，那么就会被移除。

因为文件打开或关闭时我们没有被调用，所以我们在这个模块中没有地方可以放try_module_get和module_put，如果打开文件然后删除模块，后果是没有办法避免的。

通过proc_create来加载module，同时还对该文件的进行配置。返回null说明创建失败。

每次文件被读的时候，读函数被调用，有两个重要的参数，buffer，offset

buffer是返回给读这个文件的进程，比如cat， offset是当前文件位置。

如果这个函数的返回值不是NULL，这个函数会继续被调用，所以要小心这个函数，如果他从不返回0，读函数会无限循环。

调用了一次read之后， offset变化，然后再次调用，读失败，返回0，让函数退出。

[610827.898870] /proc/helloworld created
[610841.692098] procfile read helloworld
[610841.692121] copy_to_user failed

## 7.1 proc_ops 数据结构

相比于file_operations ，proc_ops更省空间，因为有很多接口是用不上的，同时还提高性能。

比如，如果一个proc/下的文件夹从来都不会消失，那么可以通过设置标志位proc_flag ， 从而减少2次原子操作，1次空间分配，1次释放，在每一次预打开，读，关闭序列。

## 7.2 读和写/proc文件

我们已经看了一个非常简单的例子，对于/proc 文件，我们只是读，但其实也可以写，需要从用户空间拷贝到内核空间。

为什么要拷贝呢？因为在英特尔的架构中，是分段的，内核有一个内存段，每个进程有另外的内存段，因此需要拷贝。

如果只是收发一个字符：put_user , get_user 就可以

如果是一些列的字符串： copy_to_user ， copy_from_user

## 7.3 管理/proc文件用标准文件系统

也可以用inode来管理/proc 文件。主要是用先进的功能，比如权限

在linux有一个标准机制在文件系统中注册。

**struct** inode_operations

这个结构体是用来描述文件接口的。它还包含了一个 **struct** proc_ops

file_operations : 处理file本身

inode_operations : 处理文件引用，比如创建连接

在 /proc 中，每当我们注册一个新文件时，我们都可以指定使用哪个结构 inode_operations 来访问它。这是我们使用的机制，一个结构体 inode_operations 包含一个指向结构体 proc_ops 的指针，该结构体包含指向我们的 procf_read 和 procfs_write 函数的指针。

**module_permission** 函数会被调用，如果proc文件被调用时，目前这个能否访问是根据用户的操作（read ,write ,exec）和uid来判定的。但其实这是可以基于我们喜欢的任何内容，比如其他进程是否在使用，一天中哪些时间可以被访问，或者是我们收到的最后一个输入。

注意，在内核中的读写是刚好相反的，我们是从用户的角度去思考读写的，比如写是要去读用户的输入，读是要给用户那边去写。

pr_debug 怎么查看呢？

在makefile中添加：

```
CFLAGS_filename.o := -DDEBUG
```

## 7.4 管理/proc 用seq_file

写一个/proc文件似乎是比较困难的，有一组API 叫做seq_file 帮助我们规范化/proc文件的输出。

这基于顺序，这个顺序由start , next , stop 组成

这个api会在使用者read的时候启动。

有一张图，解释这个顺序流程。

值得注意的是，当stop被调用之后，立马就启动start，但此时返回null，相当于函数在这个地方终止。

如果想了解更多，那么可以读这些内容。

If you want more information, you can read this web page:

* [https://lwn.net/Articles/22355/](https://lwn.net/Articles/22355/)
* [https://kernelnewbies.org/Documents/SeqFileHowTo](https://kernelnewbies.org/Documents/SeqFileHowTo)

You can also read the code of [fs/seq_file.c](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/tree/fs/seq_file.c) in the linux kernel.

# 8 sysfs:Interacting with your module

sysfs 允许你去跟运行的内核相互作用，在内核空间中，通过读，设定module中的变量值，这对于调试来说，太有用了。或者只是一个给应用和脚本的接口 。

你可以找到sysfs的目录和文件，在/sys文件夹中。

这里有一个最简单的sysfs的例子。

通过这个demo，我们确实能很方便的读写一个变量。（有点像共享内存）

```shell
make 
sudo insmod hello-sysfs.ko
sudo lsmod | grep hello_sysfs
cat /sys/kernel/mymodule/myvariable
echo "32" > /sys/kernel/mymodule/myvariable 
cat /sys/kernel/mymodule/myvariable
sudo rmmod hello_sysfs
```

# 9 讨论一下设备文件

设备文件应该代表物理设备，大多数物理设备都是为了输入输出，我们必须提供一些机制，让在内核中的设备驱动去进程中获得输出，发送给设备。在接下来的例子中，这个实现是通过device_write函数的。

但这不总是足够的，想象一下，你有一些串口连接着调制解调器，很自然的，你应该使用设备文件去写东西到调制解调器中，以及读东西从调制解调器中。

然而，你怎么去配置呢？比如去配置波特率。

ioctl 就这么被引入了Unix，每个设备可以有它自己的ioctl命令，可以是读的ioctl，写的ioctl , 可以都有，也可以都没有。


ioctl 函数有3个参数：

1. 文件描述
2. ioctl 数字 ， 用一个宏值来填（_IO,_IOR,_IOW,_IOWR）
3. 一个参数，它是 long 类型，因此您可以使用强制转换来使用它来传递任何东西。（你虽然不能传递一个结构体进来，但是你可以传一个结构体的指针。）


我们应该格外注意并发访问共享资源的问题，我们一般使用原子操作compare-and-swap ，去保证独家的访问

## 9.1 ioctl的例子代码有点复杂，之后再回头看。

# 10系统调用

目前为止，我们使用了系统的API去注册/proc，device处理函数，这就可以了，如果你只是要完成系统想要你完成的事情。

但如果我们想做一些不寻常的事情呢？

这个时候，其实你就要靠自己了。

proc file , device file 都是细枝末节，关键是系统调用。

可以通过strace来查看。


通常来说，因为受到保护模式，页保护，一个进程是不能去访问内核的。

但系统调用是一个例外，当进程填好了正确的参数，执行一个特殊的指令，跳转到之前在内核中定义的位置，在英特尔的CPU中，其实就是中断0x80。硬件知道你跳转到这个位置，就不再是严格意义上的用户模式，而实际上是内核模式。

进程其实跳转到了system_call的地址。在这个地方的程序检测系统调用号，告诉系统需要调用什么服务。然后系统看一下自己的系统调用表。

sys_call_table ， 去看到底是哪个函数应该被调用。然后返回，做一些系统校对，然后返回到进程中。（如果进程的时间片用完了，也可能返回到其他进程中。）


这些代码在 : **arch/$(architecture)/kernel/entry.S**, after the line `ENTRY(system_call)`

那么我们可以利用这个机制，去修改sys_call_table 从而满足我们的个性化需求。

在X86架构，我们可以通过修改CR0寄存器去满足我们的要求，这个寄存器保存了多种控制位，比如WP位在CR0中代表写保护，当这个地方被设之后，处理器不再允许写只读的内容，所以，我们可以不使能WP位，在修改sys_call_table的时候。


从 Linux v5.3 开始，由于安全问题导致敏感的 cr0 位无法使用 write_cr0 函数，攻击者可能会写入 CPU 控制寄存器以禁用 CPU 保护，如写保护。因此，我们必须提供自定义组装例程来绕过它。


但是， sys_call_table 符号没有暴露，以防止误用。但是获取符号、手动符号查找和 kallsyms_lookup_name 的方法很少。这里我们使用两者都依赖于内核版本。


因为正直的控制流，为了防止黑客去重定向执行代码，为了保证间接调用到我们期待的地址和返回我们没有改变的地址。

在linux5.7之后，内核修补了一系列控制流强制执行。


Kprobes 探针。

Kprobes 通过替换被探测指令的第一个字节在函数入口处插入断点探针可以插入一个断点

当 CPU 到达断点时，寄存器被存储，控制权将传递给 Kprobes。它将保存的寄存器的地址和 Kprobe 结构传递给您定义的处理程序，然后执行它。 Kprobes 可以通过符号名称或地址注册。在符号名称中，地址将由内核处理。


**KASLR** (Kernel Address Space Layout Randomization)

每次重启都会改变内核代码和数据地址

`/boot/System.map`

这里面的地址会被随机处理


实例代码是一个内核模块实例，实现的是一个间谍，当用户打开一个文件时，我们去打印一下信息。我们用自己的sys_open函数去替代了系统的sys_open函数。


这里提供了一个修改系统调用的模块。


# 11 阻塞进程和线程

## 11.1 睡眠

如何内核被打扰，他可以告诉进程，让进程先进入睡眠模式，等到内核可以提供进程想要的服务时，再唤醒。


当我们调用一个模块时，如果发现被别人用了，无法使用，进入了系统调用：**wait_event_interruptible** ，这时候进程阻塞，当另外一个执行完毕后，会唤醒其他在等待这个资源的进程，这时进程继续执行， 这意味着该进程仍处于内核模式 - 就进程而言，它发出了 open 系统调用并且系统调用尚未返回。在发出调用和返回之间的大部分时间里，该进程不知道其他人使用了 CPU

然后它可以继续设置一个全局变量来告诉所有其他进程该文件仍然打开并继续它的生命。当其他进程获得一块 CPU 时，它们会看到该全局变量并重新进入睡眠状态。

为了让我们的生活更有趣，module_close 没有垄断唤醒等待访问文件的进程。诸如 Ctrl +c (SIGINT) 之类的信号也可以唤醒进程。这是因为我们使用了 module_interruptible_sleep_on 。我们可以使用 module_sleep_on 代替，但这会导致用户非常愤怒，他们的 Ctrl+c 被忽略。

通过原子操作确保文件只能被一个人打开，通过 `wait_event_interruptible` 函数确保非第一个打开的函数进入阻塞，对于非阻塞的调用，那么可以通过返回值来决定是否要继续等。


## 11.2 完成

内核中的多线程例子。

还有一些关于定时，被中断的设置，但是一般用不上，没必要额外增加复杂度。


# 12 避免冲突和死锁

通过设置正确的锁来避免。

## 12.1 互斥锁

这跟在应用中写的互斥锁没有区别。

## 12.2 自旋锁

自旋锁会拉满CPU资源，超过几个ms的操作就不要用自旋锁。


## 12.3 读写锁

特殊的自旋锁

IRQ **(interrupt request)** , 中断请求

安全的中断请求irqsafe


## 12.4 原子操作

CPU中的原子操作不会被中断

可以通过某些原子操作来确保工作不会被打断。


# 13 代替打印宏

## 13.1 替换

X windows 系统 和 内核编程不能混淆。可以通过写到当前tty来打印调试信息。


## 13.2 刷新键盘的LED

这个实验可以调用键盘的LED，可以通过让键盘闪烁来提供状态信息


# 14 调度任务

运行的任务有两种主要的方式：小任务和工作队列。

小任务是指，快，简单的方式去调度一个简单的函数。比如触发一个中断。

工作队列是更复杂的，但也更适合运行多种任务在一个序列中。


## 14.1 小任务

tasklet有一些不好，只能运行在内核态，不能睡眠，不能去访问用户数据。多个不同的小任务会并行运行。

可以被工作队列，定时器，线程中断来替代 。 

**DECLARE_TASKLET_OLD** 这个宏还是存在，为了让老的api能被调用。


## 14.2 工作队列

给调度器增加一个任务，可以使用工作队列。内核会用CFS去执行工作队列中的工作。


# 15 中断处理

## 15.1 中断处理

CPU 与计算机硬件的其余部分之间有两种类型的交互。

第一种是 CPU 向硬件发出命令，命令是硬件需要告诉 CPU 一些事情。

第二个，称为中断，更难实现，因为它必须在硬件方便时处理，而不是 CPU。硬件设备通常具有非常少量的 RAM，如果您没有在可用时读取它们的信息，它就会丢失。


在linux中，硬件中断叫做IRQ ，interrupt request .

有两种中断请求：

短中断：只需要很短的周期时间，期间整个机器会进入阻塞，其他中断不会被响应。

长中断：占用时间长一些，这种中断发生时，其他中断也可以发生（同一个设备）。

如果都可以时，推荐使用长中断。


当CPU接收到中断时，他会停止他当前的所做的事（除非现在在处理的是更为重要的中断，这样的话，他会先完成当前的中断处理，然后再处理新的中断。），保存当前在栈上的参数，调用中断处理函数。这意味着很多事情不被允许在中断处理函数本身运行。因为系统在一个未知的状态。

内核解决这个问题，通过把中断处理分成两部分。

- 第一部分立即执行并屏蔽中断，硬件中断必须快速处理。
- 第二部分延期处理繁重的工作。

BH ， bottom halves  下半部 ， 用来统计保留的延迟函数。

软中断和更高层的抽象，小任务，代替了BH，从2.3开始。

实现这一点的方法是调用 request_irq() 以在收到相关 IRQ 时调用您的中断处理程序。通过requset_irq()去获得中断处理函数调


在实践中IRQ的处理可能会更复杂一些。

硬件通常以链接两个中断控制器的方式设计，以便来自中断控制器 B 的所有 IRQ 级联（cascaded）到来自中断控制器 A 的某个 IRQ。当然，这需要内核找出它之后真正是哪个 IRQ，这会增加开销。

其他架构提供一些特殊，非常低开销的"Fast IRQ" or FIQs. 要利用它们，需要用汇编程序编写处理程序，因此它们并不真正适合内核。可以使它们的工作方式与其他类似，但在此过程之后，它们不再比“普通”IRQ 快

*SMP*一般指对称多处理。 对称多处理"（Symmetrical Multi-Processing）简称*SMP*

**Advanced Programmable Interrupt Controller** ( **APIC** )  高级可编程中断控制器[https://en.wikipedia.org/wiki/Advanced_Programmable_Interrupt_Controller]()

在多核处理器中，我们需要知道的不光是中断，还要知道在哪个cpu中中断。


这个函数收到IRQ 数字，函数名字，标志位，/proc/interrupts的名字，一个传递给中断处理函数的参数。通常有一定数量的 IRQ 可用，有多少IRQs是由硬件决定的。

标志位可以包含**SA_SHIRQ** 去表面你愿意分享IRQ 跟其他中断处理。

`SA_INTERRUPT` 表面这是一个快中断。这个函数只有在没有处理函数挂在这个IRQ中，或者你们两个都愿意分享的情况下才能成功。


## 15.2 检测键盘输入

很多流行的单片机电脑，有一些列的GPIO口，点击一个按钮去实现一些功能，是一些基本的操作，这其实就用到了中断，这样可以避免CPU浪费时间，电池耗电去轮询这个输入状态。这个输入触发CPU去跑某个函数的方式是更好的选择。

这是一个树莓派的驱动，有时间去试一下。


## 15.3 下半部

假设你想在一个中断例程中做一堆事情。在不使中断在很长一段时间内不可用的情况下做到这一点的常用方法是将其与 tasklet 结合使用。这会将大部分工作推到调度程序中。

快速执行完那些关键操作，比如识别输入了0还是1, 然后启动一个任务。


# 16 加密

在互联网的黎明期，大家互相完全信任，但这并没有工作得很好。最初编写本指南时，那是一个更加纯真的时代，几乎没有人真正关心加密——至少在所有内核开发人员中。现在肯定不再是这种情况了。为了处理加密的东西，内核有自己的 API，支持加密、解密的常用方法和你最喜欢的哈希函数。


## 16.1 哈希函数

demo演示了 如何使用sha256


## 16.2 对称密钥加密

AES加密。



# 17 标准化接口：设备模型

到目前为止，我们已经看到各种各样的模块在做各种各样的事情，但是它们与内核其余部分的接口并不一致。

为了强加一些一致性，以便至少有一种标准化的方式来启动、暂停和恢复设备，添加了设备模型。下面是一个示例，您可以以此为模板添加自己的挂起、恢复或其他接口功能。


# 18 优化

## 18.1 Likely 和 unlikely 条件

有时，你可能希望你的代码尽可能的运行快，特别是如果你处理一个中断然后做一些事情，这会导致一些可以被注意到的延迟。

如果你的代码有布尔条件且如果你知道这个结果大部分情况都是true 或者 false，你可以用likely和unlikely宏。

比如：

```cpp
bvl = bvec_alloc(gfp_mask, nr_iovecs, &idx); 
if (unlikely(!bvl)) { 
    mempool_free(bio, bio_pool); 
    bio = NULL; 
    goto out; 
}
```

如果使用了这两个宏，编译器会改变机器码的输出，会把跳转指令放到情况比较少的分支。这可以防止刷新处理器的流水线。


# 19. 常见的陷阱

## 19.1 使用标准库文件

你不能使用标准库文件，你只能使用内核函数，你可以在/proc/kallsyms

## 19.2 屏蔽中断

你可以短时间的屏蔽中断，这是ok的，但你不要忘记打开，这样系统会卡死，你需要重启它。


# 20. 从这里开始，你应该去哪里？

对于非常感兴趣内核编程的人，推荐去[kernelnewbies.org](https://kernelnewbies.org/) 以及去看Documentation 的子文件夹，但这不太好理解，但这是进一步调查的开始。

最好的方式去学习内核，就是去读内核的源码。
