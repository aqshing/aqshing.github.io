---
title: core分析.md
categories: aqshing
Email: jdbc.cc <work@jdbc.cc>
abbrlink: '1265'
date: 2023-08-21 10:51:38
changed: 2023-08-21 21:24:38
tags:
---
# core分析

## 1. 前期准备
1. 关闭Linux内存地址随机化
Linux在启动可执行文件, 将代码加载到内存空间时, 会自动给程序基地址(即程序初始化地址)加上一个随机数的偏移量, 这样就会导致程序每次启动时的进程地址都不同, 即Linux内核会自动将进程的内存地址空间随机化，以增加攻击者对程序的攻击破解难度。
但这也会导致我们使用gdb调试程序时, 每次调试的地址都不一样, 导致我们调试的程序每次都不一样, 调试起来很麻烦。所以我们需要关闭Linux内存地址随机化。

参考链接: [Linux下关闭ASLR(地址空间随机化)的方法](https://blog.csdn.net/counsellor/article/details/81543197)

```bash
# 关闭Linux 内存地址随机化机制
echo 0 >/proc/sys/kernel/randomize_va_space
# 这种办法是临时关闭, 重启后失效, dmserver是开机时自启动的(systemctl status DmServicexpc),
# 所以我们修改完之后, 需要重启dmserver. 想要永久关闭, 请参考上面给的链接
# 也可以在~/.gdb.init配置文件中添加如下内容
# 关闭ASLR：
set disable-randomization on
# 开启ASLR：
#set disable-randomization off
# 查看ASLR状态：
#show disable-randomization
```
``` markdown
User
echo 0 >/proc/sys/kernel/randomize_va_space

ChatGPT
这条命令是用来控制 Linux 内核中的 ASLR（Address Space Layout Randomization，地址空间布局随机化）功能。ASLR 是一种安全机制，用于随机化程序的内存布局，从而增加攻击者预测和利用漏洞的难度。

/proc/sys/kernel/randomize_va_space 是一个系统文件，它可以用来控制 ASLR 的级别。ASLR 的级别有以下几个选项：

0: 关闭 ASLR。程序和库加载到固定的内存地址，不进行随机化。
1: 仅对可执行文件进行随机化。
2: 对可执行文件和动态链接库进行随机化（默认选项）。
3: 对所有内存段（包括堆、栈、共享内存等）进行随机化。
使用 echo 0 > /proc/sys/kernel/randomize_va_space 命令可以将 ASLR 关闭，这样程序和库将加载到固定的内存地址，而不进行随机化。这在某些情况下可能会降低一些安全性，因为攻击者可以更容易地预测程序的内存布局。然而，在某些调试和分析场景下，关闭 ASLR 可能会更方便，因为每次运行程序时内存布局都保持一致。

需要注意的是，修改 /proc/sys/kernel/randomize_va_space 需要超级用户权限。此外，更改系统的安全设置可能会带来安全风险，因此在生产环境中应该谨慎操作。
```

2. 修改kernel.core_pattern
```bash
# 我修改的配置 vi /etc/sysctl.conf
# kernel.core_pattern=/dm/dmcore/core-%t-%e-%p-%c.core
kernel.core_pattern=/dm/dmcore/%e-%i-%s-%p-%t.core
# 临时修改内核参数
❯ echo "%e-%i-%s-%p-%t.core" | sudo tee /proc/sys/kernel/core_pattern
❯ cat /proc/sys/kernel/core_pattern
/dm/dmcore/%e-%i-%s-%p-%t.core
```

[core_pattern文档说明参考链接](http://man7.org/linux/man-pages/man5/core.5.html)
```bash
ChatGPT
当您设置 core_pattern 时，您可以使用不同的占位符来定义核心转储文件的命名格式。每个占位符都会被实际的信息替代，以便在生成核心转储文件时包含有关生成核心转储的进程或线程的相关信息。下面是每个占位符的含义：

%%：一个单独的 % 字符。
%c：崩溃进程的核心文件大小软资源限制（自 Linux 2.6.24 起）。
%d：转储模式，与通过 prctl(2) 返回的值相同，即 PR_GET_DUMPABLE（自 Linux 3.7 起）。
%e：进程或线程的 comm 值，通常与可执行文件的文件名相同（去除路径前缀，并截断为最多 15 个字符），但可能已被修改为其他值。comm 值在 /proc/pid/comm 和 /proc/pid/task/tid/comm 中可以看到（自 Linux 3.0 起）。
%E：可执行文件的路径名，斜杠（/）被替换为感叹号（!）（自 Linux 3.0 起）。
%g：崩溃进程的实际 GID（组 ID）。
%h：主机名，与 uname(2) 返回的 nodename 相同。
%i：触发核心转储的线程的 TID（线程 ID），在该线程所在的 PID 命名空间中可见（自 Linux 3.18 起）。
%I：触发核心转储的线程的 TID（线程 ID），在初始 PID 命名空间中可见（自 Linux 3.18 起）。
%p：被转储进程的 PID（进程 ID），在该进程所在的 PID 命名空间中可见。
%P：被转储进程的 PID（进程 ID），在初始 PID 命名空间中可见（自 Linux 3.12 起）。
%s：导致转储的信号编号。
%t：转储时间，表示自 1970 年 01 月 01 日 00:00:00 +0000（UTC）以来的秒数。
%u：崩溃进程的实际 UID（用户 ID）。
通过组合和使用这些占位符，您可以自定义核心转储文件名的格式，使其包含您希望查看的有关生成核心转储的进程或线程的信息。这样做可以帮助您更好地识别和理解核心转储文件的来源和原因。
```

## 2.

## 自由补充和发挥
1. 首先，先用`file /dm/dmdbms/dm8/bin/dmserver`查看*dmserver*确定其是**not stripped**，没有删除掉符号表信息，这样就可以找到相关调试信息从而可以被gdb调试。
```bash
❯ file /dm/dmdbms/dm8/bin/dmserver
/dm/dmdbms/dm8/bin/dmserver: setuid setgid ELF 64-bit LSB executable, x86-64, version 1 (SYSV),
dynamically linked (uses shared libs), for GNU/Linux 2.6.32, BuildID[sha1]=7aa1c5e8283daa989c1cf812897612aa8331a14c, not stripped
```
2. 知识补充
![dmserver可执行文件和调试符号表的关系](http://cdn.jdbc.cc/pict/202308211212502.png)
```bash
dmdba@core2 /dm/dmcore
❯ file /dm/dmdbms/dm8/bin/dmserver
/dm/dmdbms/dm8/bin/dmserver: setuid setgid ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked (uses shared libs), for GNU/Linux 2.6.32, BuildID[sha1]=7aa1c5e8283daa989c1cf812897612aa8331a14c, not stripped

dmdba@core2 /dm/dmcore
❯ readelf -S /dm/dmdbms/dm8/bin/dmserver | grep debug
  [30] .gnu_debuglink    PROGBITS         0000000000000000  04007c9d

dmdba@core2 /dm/dmcore
❯ file ~/dm8sym/dmserver_s.debug
/home/dmdba/dm8sym/dmserver_s.debug: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked (uses shared libs), for GNU/Linux 2.6.18, BuildID[sha1]=f096a6b6b61c7819917ba6d50619cb606d96a641, not stripped

dmdba@core2 /dm/dmcore
❯ readelf -S ~/dm8sym/dmserver_s.debug | grep debug
  [29] .debug_aranges    PROGBITS         0000000000000000  00000271
  [30] .debug_pubnames   PROGBITS         0000000000000000  0000b491
  [31] .debug_info       PROGBITS         0000000000000000  00161441
  [32] .debug_abbrev     PROGBITS         0000000000000000  0407e486
  [33] .debug_line       PROGBITS         0000000000000000  041569b4
  [34] .debug_str        PROGBITS         0000000000000000  046d6eaf
  [35] .debug_loc        PROGBITS         0000000000000000  048db03c
  [36] .debug_pubtypes   PROGBITS         0000000000000000  07021e0e
  [37] .debug_ranges     PROGBITS         0000000000000000  079fb478

dmdba@core2 /dm/dmcore
❯ gcc -g -Wall -o main ~/code/main.c
# -g: 编译可执行文件时, 带上调试信息
dmdba@core2 /dm/dmcore
❯ readelf -S main | grep debug
  [26] .debug_aranges    PROGBITS         0000000000000000  0000309d
  [27] .debug_info       PROGBITS         0000000000000000  000030cd
  [28] .debug_abbrev     PROGBITS         0000000000000000  000034d6
  [29] .debug_line       PROGBITS         0000000000000000  00003604
  [30] .debug_str        PROGBITS         0000000000000000  00003730

```
从上图中, 可以看到一个带有调试信息的可执行文件要至少包含`.debug_aranges .debug_info .debug_abbrev .debug_str`这些节, gdb通过读取这些节来获取调试信息, 从而实现调试.
用`readelf -S /dm/dmdbms/dm8/bin/dmserver | grep debug`命令搜索*dmserver*中却没有调试信息, 我们却没有搜索到和调试相关的节, 但是我们搜索到了.gnu_debuglink节, 该节可以将可执行文件与调试符号表文件关联起来.
[深入理解debuginfo](https://blog.csdn.net/chinainvent/article/details/24129311)

```
User
.gnu_debuglink    PROGBITS         0000000000000000  04007c9d
ChatGPT
.gnu_debuglink 是一种 ELF（Executable and Linkable Format，可执行与可链接格式）节（section），通常用于保存调试信息文件的相关信息，以便在调试时将可执行文件与调试信息文件关联起来。
在给定的信息中，0000000000000000 表示该节的虚拟地址或偏移量为0，而 04007c9d 是该节在文件中的偏移量，指示 .gnu_debuglink 节在文件中的位置。
这个节通常在可执行文件中用于存储调试信息文件的路径和校验信息，以便调试器能够在需要时找到并加载调试信息，以便进行源代码级别的调试。


User
.gnu_debuglink这个节的作用是?
ChatGPT
.gnu_debuglink 节是 ELF（Executable and Linkable Format，可执行与可链接格式）可执行文件中的一个节，它用于存储调试信息文件的相关信息。它的主要作用是将可执行文件与调试信息文件关联起来，以便在调试时能够加载并使用调试信息。
具体来说，.gnu_debuglink 节通常包含以下信息：
调试信息文件的路径： 这个节存储了调试信息文件的路径，该文件通常是一个单独的文件，包含了与可执行文件相关的调试符号、源代码信息等。
CRC 校验值： 为了验证调试信息文件的完整性，.gnu_debuglink 节还会包含一个 CRC（Cyclic Redundancy Check，循环冗余校验）校验值，用于确保调试信息文件没有被篡改。
当你使用调试器来调试一个可执行文件时，调试器会检查是否存在 .gnu_debuglink 节以及其中的信息，如果存在，调试器将会尝试加载对应的调试信息文件，以便能够在调试过程中显示源代码、变量名等更有用的信息，从而进行更有效的调试。
需要注意的是，.gnu_debuglink 节不是必需的，一些情况下可执行文件可能没有这个节，但如果可用，它可以大大提高调试的效率和准确性。


User
[27] .debug_aranges    PROGBITS         0000000000000000  00002adc
[28] .debug_info       PROGBITS         0000000000000000  00002b0c
[29] .debug_abbrev     PROGBITS         0000000000000000  00002ed9
[30] .debug_line       PROGBITS         0000000000000000  00002ff6
[31] .debug_str        PROGBITS         0000000000000000  00003125
这四个节的作用是什么?
ChatGPT
这四个节都是 ELF（Executable and Linkable Format，可执行与可链接格式）文件中用于存储调试信息的节。它们分别是：
.debug_aranges： 这个节包含调试地址范围（Address Ranges）信息。它帮助调试器更有效地映射调试信息与实际内存地址之间的关系，以便在调试时能够正确地将调试信息应用于对应的内存位置。
.debug_info： 这个节包含实际的调试信息，比如类型信息、变量信息、函数信息等。.debug_info 节中的信息是调试器在调试过程中最常用的信息，它帮助调试器理解程序的结构和运行时状态。
.debug_abbrev： 这个节包含了调试信息的缩写表（Abbreviation Table）。调试信息往往包含很多重复的结构，为了节省空间，ELF 格式使用缩写表来描述这些结构，而实际数据则存储在 .debug_info 节中。.debug_abbrev 节存储了缩写表中的各个缩写项。
.debug_line： 这个节包含源代码与机器代码之间的映射关系，也就是源代码行号与对应机器代码指令的关系。这对于调试器能够在源代码级别上定位问题非常重要，因为它使调试器能够在源代码级别上设置断点、查看变量值等。
.debug_str： 这个节包含字符串表，其中存储了各种调试信息中使用的字符串，比如类型名、变量名等。这使得调试器能够将这些字符串与具体的调试信息关联起来。
总之，这些调试信息节在 ELF 文件中存储了与调试相关的重要信息，它们帮助调试器在调试过程中理解程序的结构、映射内存地址和源代码行号之间的关系，并提供了必要的信息来进行源代码级别的调试。
```

# 参考附录
压缩包里面的*.debug符号表应该是用下面这种方式生成的，一般情况下，gdb会自动加载调试符号。当我们换了调试环境，或者符号文件路径和 ELF 文件内的路径不匹配的时候，就会报符号找不到的错，这时候仍然需要手动处理符号文件。
[GDB 调试 - 正确地加载调试符号文件 - 莫扎特的代码 - 博客园 (cnblogs.com)](https://www.cnblogs.com/codingMozart/p/17117698.html)

[GNU Debug Link](https://blog.csdn.net/CaspianSea/article/details/43083977)
[bing搜索](https://cn.bing.com/search?q=+.gnu_debuglink)