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
