---
title: top源码学习.md
Author: aqshing
Email: jdbc.cc <aqdebug@gmail.com>
abbrlink: aa42
date: 2022-10-31 19:28:14
changed: 2023-05-07 19:25:49
tags:
---

---
title: top源码学习
date: 2022-10-31 19:28:14
tags:
---
# load average源码学习

## 楔子
``` text
Q1：客户的MySQL数据库服务器负载很高（load average值大约为59.21, 48.39, 47.90），业务SQL响应特别慢，
要求：
a. 请问可能有哪些原因会导致负载如此之高（请罗列不少于3点）。
b. 您有什么样的分析套路、方法、流程、知识库可用于分析此类情况（请您按照问题排查思路描述，注意分析方法及逻辑）。
- 输出类似问题的处理指南手册，包括脚本，命令，可以在现场直接复用的。
```
题目中没有给出服务器的相关信息，只给了3个数字，上来就让分析找原因直接就把我给弄不会了。正好我也不太熟悉这块的内容，于是便找到了top的源码学习了一波，终于勉强弄明白了些，下面就我的学习作一些分享。

---

## top源码分析
首先下载top源码，地址如下：
```bash
git clone https://gitlab.com/procps-ng/procps.git
```
top的源码一共7000多行，粗略的看了一下，发现top只是遍历了/proc目录下的一些文件，并对其做格式化输出。也就是说核心的代码逻辑都是在/proc由Linux内核去实现的。
为了弄清楚top都具体读了哪些文件，用 **`strace -T -tt -e trace=file top `** 来具体查看一下。

首先, top是加载了一些动态库文件。
![top动态库](http://cdn.jdbc.cc//pict/20221102193312.png)

然后发现top打开了 */proc/sys/kernel/pid_max* */sys/devices/system/node/node0/meminfo* 、*/proc/uptime*等关于OS统计信息的文件。
![top1](http://cdn.jdbc.cc/pict/top1.png "top1")

最后，top遍历/proc目录下的{pid}文件夹，打开/proc/{pid}/stat和/proc/{pid}/statm来读取相关信息。

在这期间, 我们发现top是读取的 /proc/loadavg 这个文件得到的 load average 的那3个值。

恰巧，我在研究top源码的时候，发现`uptime`命令也能读取那三个值。
![](http://cdn.jdbc.cc/pict/202212281052318.png)
uptime也是包含在procps这个项目中的，源码在uptime.c中，只有124行，我们可以拿gdb来跟踪一下，看看uptime是怎么拿到loadavg的那三个值得。
首先，我们要先编译一下procps这个项目。它是使用传统的`./configure`方式进行编译的。
我们首先要安装 *autoconf automake libtool autopoint* 等命令，然后运行`./autogen.sh`生成configure文件。
![](http://cdn.jdbc.cc/pict/202212281107751.png)
因为gdb调试依赖debug符号表，所以我们要指定gcc的编译参数包含`-g -O0`(-g生成debug符号表, -O0表示不优化), 然后再调用`make`命令去编译源码即可.
![](http://cdn.jdbc.cc/pict/202212281123290.png)
`make`完成之后,进入src目录,发现在该目录下生成了一个bash脚本文件,通过`./uptime`即可运行,看到该命令的输出.我们通过`bash -x uptime`来看一下它具体运行了哪个ELF可执行文件.
![](http://cdn.jdbc.cc/pict/202212281139739.png)
我们发现它运行的是`.libs/uptime`,但是还需要导出动态链接库的环境变量`LD_LIBRARY_PATH=/workspaces/note/clone/procps/library/.libs`,由此我们就可以用gdb进行调试了.
![](http://cdn.jdbc.cc/pict/202212281408150.png)
用gdb单步调试,发现它进入了`procps_uptime_sprint()`这个函数,继续跟踪进去看看:
![](http://cdn.jdbc.cc/pict/202212281635743.png)
很容易就可以看到,该函数在136行通过`procps_loadavg(&av1, &av5, &av15)`这个函数获取到了load average的3个值.
![](http://cdn.jdbc.cc/pict/202212281639368.png)
可以看到`uptime`也是通过fopen打开 **/proc/loadavg** 这个文件, 然后通过`fscanf(fp, "%lf %lf %lf", &avg_1, &avg_5, &avg_15)`这条命令去读取loadavg的那3个值.
![](http://cdn.jdbc.cc/pict/202212281702549.png)
由此,我们可以确定用户态应用程序获取loadavg的方法就是通过读取 **/proc/loadavg** 获得的.但是我们想要知道的是该值是怎么计算出来的以及它背后的计算原理, 由于 **/proc/loadavg** 该文件中的值是由Linux内核直接写入的, 所以我们接下来就深入Linux内核源码, 看看Linux是怎么计算该值并写入的.

---

## /proc目录简介

> **<font color=yellow>/proc/{pid}/statm</font>**

/proc/{pid}/statm 显示进程所占用内存大小的统计信息。包含七个值，度量单位是 page(page大小可通过 getconf PAGESIZE 得到)。
```bash
# 示例
❯ cat /proc/1/statm
42242 2679 1494 240 0 5185 0
```
> 含义如下:
> 1. 进程占用的总的内存
> 2. 进程当前时刻占用的物理内存
> 3. 同其它进程共享的内存
> 4. 进程的代码段
> 5. 共享库(从2.6版本起，这个值为0)
> 6. 进程的堆栈
> 7. dirty pages(从2.6版本起，这个值为0)

**<font color=yellow>/proc/{pid}/status</font>** 包含进程的状态信息。其很多内容与 /proc/{pid}/stat 和 /proc/{pid}/statm 相同，但是却是以一种更清晰地方式展现出来。

---

## 内核源码探究
本次分析的Linux Kernel的版本并没有选最新的linux-6.1.*的版本，选的是linux-5.4.219，对应OS版本约为 Ubuntu 18和CentOS 8, 是目前市面上比较主流,比较常用的版本，更具实际意义，地址如下：
```bash
wget https://mirrors.edge.kernel.org/pub/linux/kernel/v5.x/linux-5.4.219.tar.gz
```

### Linux内核目录简介
* arch：包含和CPU相关的代码，每种架构一个目录，如x64、arm64、risc-v等。存放的是Linux内核对各平台芯片、进程调度、内存管理、中断等的支持等。
* block：块设备驱动程序 I/O 调度。
* crypto：常用加密和散列算法（如AES、SHA 等），还有一些压缩和 CRC 校验算法。
* documentation：内核各部分的通用解释和注释。
* drivers ：设备驱动程序，每个不同的驱动占用一个子目录，如char、block、net、mtd、i2c 等。
* fs：所支持的各种文件系统，如EXT、FAT、NTFS、JFFS2 等，**我们想要看的/proc目录源码也包含在其中**。
* include：内核 API 头文件，与系统相关的头文件放置在include/linux 子目录下。
* init：内核初始化代码。著名的start_kernel() 就位于init/main.c 文件中。
* ipc：进程间通信的代码。
* kernel ：内核最核心的部分，包括进程调度、定时器等，而和平台相关的一部分代码放在arch/*/kernel 目录下。
* lib：库文件代码。
* mm：内存管理代码，和平台相关的一部分代码放在arch/*/mm 目录下。
* net：网络相关代码，实现各种常见的网络协议。
* scripts：用于配置内核的脚本文件。
* security：主要是一个SELinux 的模块。
* sound：ALSA、OSS 音频设备的驱动核心代码和常用设备驱动。
* usr：实现用于打包和压缩的cpio 等。

### loadavg分析
通过搜索`loadavg`关键字,我们很容易找到Linux Kernel向外输出的相关函数`loadavg_proc_show()`, 我们以该函数为起点,展开对 **loadavg** 的追踪和分析.
![](http://cdn.jdbc.cc/pict/202212281838673.png)

#### 输出分析

```bash
[root@gip ~] ❯ cat /proc/loadavg
2.42 2.35 1.11 7/689 5994
# 2.42  : 1min 平均load
# 2.35  : 5min 平均load
# 1.11  : 15min 平均load
# 7/689 : 可运行的进程数 / 进程总数
# 5994  : 最近运行的进程ID号

[root@gip ~] ❯ grep -c '^processor' /proc/cpuinfo
8 # 查看CPU逻辑核数
```

```c
// %lu.%02lu %lu.%02lu %lu.%02lu %ld/%d   %d
//   2.42      2.35      1.11      7/689  5994
seq_printf(m, "%lu.%02lu %lu.%02lu %lu.%02lu %ld/%d %d\n",
    LOAD_INT(avnrun[0]), LOAD_FRAC(avnrun[0]),
    LOAD_INT(avnrun[1]), LOAD_FRAC(avnrun[1]),
    LOAD_INT(avnrun[2]), LOAD_FRAC(avnrun[2]),
    nr_running(), nr_threads,
    idr_get_cursor(&task_active_pid_ns(current)->idr) - 1);


#define FSHIFT    11    /* nr of bits of precision */
#define LOAD_INT(x) ((x) >> FSHIFT)
// 1<<11 = 2048
#define FIXED_1		(1<<FSHIFT)	/* 1.0 as fixed-point */
#define LOAD_FRAC(x) LOAD_INT(((x) & (FIXED_1-1)) * 100)
```
将上面的输出和下面的源码对应来看: 根据输出格式，LOAD_INT对应计算的是load的整数部分，LOAD_FRAC计算的是load的小数部分, 也就是说Linux在真正做运算的时候,把本应该是浮点数的运算转化为了定点数运算,也就是整数运算.
这样做的目的有很多:
1. Liunx主要是在CPU上进行调度, CPU的ALU只支持整数运算, 不支持浮点数运算.
2. 浮点数计算需要依赖FPU协处理器, 一些低端的CPU(如早期的RISC-V)或DSP芯片可能就不支持浮点数运算, 而Linux是一个跨平台的OS,应该减少对除CPU以外的硬件的依赖.
3. 即使CPU内部集成了FPU协处理器, 在现行的IEEE 754的浮点数标准的情况下, 也依然有精度缺失等问题, 或者是频繁的与FPU交互，导致内核效率低下等问题，而且754的算法也是将浮点数转化为整数计算，和Linux的定点数算法有异曲同工之妙.
```c
/*
 * nr_running and nr_context_switches:
 *
 * externally visible scheduler statistics: current number of runnable
 * threads, total number of context switches performed since bootup.
 */
unsigned long nr_running(void)
{
	unsigned long i, sum = 0;

	for_each_online_cpu(i)
		sum += cpu_rq(i)->nr_running;

	return sum;
}
#define for_each_cpu(cpu, mask)			\
	for ((cpu) = 0; (cpu) < 1; (cpu)++, (void)mask)
```







## 引用 & 参考
https://www.linuxprobe.com/linux-proc-pid.html
