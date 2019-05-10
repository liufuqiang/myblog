---
title: "Linux 性能优化随笔记录"
date: 2019-05-09T16:53:48+08:00
draft: true
---

# 概述

从倪朋飞老师在极客时间的《Linux性能优化实战》的专题里摘录一些知识点，算是一个学习笔记吧，大家对这个内容感兴趣的话，请移步到专栏购买支持下吧 [《Linux性能优化实战》](https://time.geekbang.org/column/article/70077)

# 工具大集合

- stress : Linux 系统压力测试工具，这里我们用作异常进程模拟平均负载升高的场景
- sysstat :包含了常用的 Linux 性能工具，用来监控和分析系统的性能。如 mpstat 和 pidstat
    - mpstat 是一个常用的多核 CPU 性能分析工具，用来实时查看每个 CPU 的性能指标，以及所有 CPU 的平均指标。
    - pidstat 是一个常用的进程性能分析工具，用来实时查看进程的 CPU、内存、I/O 以及上下文切换等性能指标。
- uptime : 包括系统运行时间，平均负载等信息
- watch : 可周期性执行某个程序，加 -d 可以高亮变化部分， -n 可以指定周期，单位是秒(s)
- vmstat : 虚拟内存统计报告 
- sysbench ： 是一个多线程的基准测试工具，一般用来评估不同系统参数下的数据库负载情况。
- perf : linux性能分析工具
- ab : 压测工具
- pstree : ps 的升级版，可以显示进程树
- execsnoop : 一个专为短时进程设计的工具。它通过 ftrace 实时监控进程的 exec() 行为，并输出短时进程的基本信息，包括进程 PID、父进程 PID、命令行参数以及执行的结果。
- hping3 : 发tcp数据包工具，如模拟syn flood攻击
- tcpdump : 抓包工具
- lscpu : 显示cpu的信息
- free : 查看内存情况



# CPU篇

### 平均负载

平均负载是指单位时间内，系统处于可运行状态和不可中断状态的平均进程数，也就是平均活跃进程数

### CPU结构

![cpu结构](https://static001.geekbang.org/resource/image/98/5f/98ac9df2593a193d6a7f1767cd68eb5f.png)
![内核空间-用户空间](https://static001.geekbang.org/resource/image/4d/a7/4d3f622f272c49132ecb9760310ce1a7.png)
![上下文切换](https://static001.geekbang.org/resource/image/39/6b/395666667d77e718da63261be478a96b.png)




- CPU上下文切换
    - 进程上下文切换
    - 线程上下文切换
    - 中断上下文切换

### 查看上下文切换

**系统级的维度**

```bash
$vmstat 5
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 1  0      0 654360 191584 4103180    0    0     5   108    0    0  2  1 96  1  0
```

需要关注几个重要内容：

- cs（context switch）是每秒上下文切换的次数。

- in（interrupt）则是每秒中断的次数。

- r（Running or Runnable）是就绪队列的长度，也就是正在运行和等待 CPU 的进程数。

- b（Blocked）则是处于不可中断睡眠状态的进程数。

**进程级维度**

```bash
# pidstat -w 5
Linux 4.4.0-33.bm.1-amd64 (n224-005-146) 	05/09/2019 	_x86_64_	(4 CPU)

05:22:08 PM   UID       PID   cswch/s nvcswch/s  Command
05:22:13 PM     0         1      1.00      0.00  systemd
05:22:13 PM     0         3      0.40      0.00  ksoftirqd/0
05:22:13 PM     0         7    110.18      0.00  rcu_sched
05:22:13 PM     0        10      0.20      0.00  watchdog/0
05:22:13 PM     0        11      0.20      0.00  watchdog/1
05:22:13 PM     0        12      0.60      0.00  migration/1
05:22:13 PM     0        16      0.20      0.00  watchdog/2
05:22:13 PM     0        18      1.20      0.00  ksoftirqd/2
05:22:13 PM     0        21      0.20      0.00  watchdog/3
05:22:13 PM     0        33      0.20      0.00  khugepaged
05:22:13 PM     0       378      0.40      0.00  kworker/0:1H
05:22:13 PM     0       385      1.00      0.00  kworker/2:1H
05:22:13 PM     0       407      0.60      0.00  jbd2/sda1-8
05:22:13 PM     0       421      0.40      0.00  kworker/3:1H
05:22:13 PM     0       456      1.60      0.00  systemd-journal
05:22:13 PM     0       646      1.00      0.00  jbd2/sdb-8
05:22:13 PM   110      1126      0.20      0.00  nscd
05:22:13 PM   105      1128      0.20      0.00  dbus-daemon
05:22:13 PM   109      1223    105.19      0.20  lldpd
05:22:13 PM     0      1262      0.20      0.00  svscan
05:22:13 PM  1000      1400      0.60      0.80  systemd
05:22:13 PM   108      1427      1.00      0.00  ntpd
```

重点关注2个对象：

- cswch 每秒自愿上下文切换（voluntary context switches）的次数。自愿上下文切换，是指进程无法获取所需资源，导致的上下文切换。比如说， I/O、内存等系统资源不足时，就会发生自愿上下文切换。

- nvcswch 每秒非自愿上下文切换（non voluntary context switches）的次数。非自愿上下文切换，则是指进程由于时间片已到等原因，被系统强制调度，进而发生的上下文切换。比如说，大量进程都在争抢 CPU 时，就容易发生非自愿上下文切换。


#### 如何查看软中断和硬中断

- /proc/interrupts
- /proc/softirq

#### linux HZ Tick Jiffies

- HZ:

    Linux核心每隔固定周期会发出timer interrupt (IRQ 0)，HZ是用来定义每一秒有几次timer interrupts。举例来说，HZ为1000，代表每秒有1000次timer interrupts。
    要检查系统上HZ的值是什么，就执行命令
    ```bash
     cat /boot/config-`uname -r` | grep '^CONFIG_HZ='
    ```

- Tick

Tick是HZ的倒数，意即timer interrupt每发生一次中断的时间。如HZ为250时，tick为4毫秒(millisecond)。 

- Jiffies

Jiffies为Linux核心变数(32位元变数，unsigned long)，它被用来纪录系统自开几以来，已经过多少的tick。每发生一次timer interrupt，Jiffies变数会被加一。

值得注意的是，Jiffies于系统开机时，并非初始化成零，而是被设为-300*HZ (arch/i386/kernel/time.c)，即代表系统于开机五分钟后，jiffies便会溢位。那溢位怎么办?事实上，Linux核心定义几 个macro(timer_after、time_after_eq、time_before与time_before_eq)，即便是溢位，也能藉由这 几个macro正确地取得jiffies的内容。 

#### top 里看到的进程的各种状态

```bash
#top
top - 17:49:20 up 125 days,  3:22,  1 user,  load average: 0.12, 0.27, 0.43
Tasks: 169 total,   2 running, 167 sleeping,   0 stopped,   0 zombie
%Cpu(s):  0.6 us,  0.3 sy,  0.0 ni, 98.8 id,  0.2 wa,  0.0 hi,  0.0 si,  0.1 st
KiB Mem:   7917296 total,  7286168 used,   631128 free,   192700 buffers
KiB Swap:        0 total,        0 used,        0 free.  4130452 cached Mem

    PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
   1223 _lldpd    20   0   49228    580      0 R   0.7  0.0 111:40.46 lldpd
2186814 tiger     20   0  309068  22716   2936 S   0.7  0.3 171:56.95 metricserver2
   1400 tiger     20   0   36212   3432   2720 S   0.3  0.0  58:38.65 systemd
   1770 tiger     20   0  246616  32100   4800 S   0.3  0.4 265:37.92 python
 653590 999       20   0  971100  36312      0 S   0.3  0.5 225:10.69 mongod
 786203 nobody    20   0   15716   3140    900 S   0.3  0.0   5:19.03 nginx
 786204 nobody    20   0   15716   2888    640 S   0.3  0.0   7:08.87 nginx
      1 root      20   0   37096   4388   2808 S   0.0  0.1  29:00.79 systemd
      2 root      20   0       0      0      0 S   0.0  0.0   0:02.08 kthreadd
      3 root      20   0       0      0      0 S   0.0  0.0   3:12.60 ksoftirqd/0
      5 root       0 -20       0      0      0 S   0.0  0.0   0:00.00 kworker/0:0H
      7 root      20   0       0      0      0 S   0.0  0.0  47:07.07 rcu_sched
      8 root      20   0       0      0      0 S   0.0  0.0   0:00.00 rcu_bh
      9 root      rt   0       0      0      0 S   0.0  0.0   1:32.35 migration/0
     10 root      rt   0       0      0      0 S   0.0  0.0   0:20.60 watchdog/0
     11 root      rt   0       0      0      0 S   0.0  0.0   0:18.62 watchdog/1
     12 root      rt   0       0      0      0 S   0.0  0.0   1:31.90 migration/1
     13 root      20   0       0      0      0 S   0.0  0.0   2:11.23 ksoftirqd/1
```

**状态列表**

- R 是 Running 或 Runnable 的缩写，表示进程在 CPU 的就绪队列中，正在运行或者正在等待运行。

- D 是 Disk Sleep 的缩写，也就是不可中断状态睡眠（Uninterruptible Sleep），一般表示进程正在跟硬件交互，并且交互过程不允许被其他进程或中断打断。

- Z 是 Zombie 的缩写。它表示僵尸进程，也就是进程实际上已经结束了，但是父进程还没有回收它的资源（比如进程的描述符、PID 等）。

- S 是 Interruptible Sleep 的缩写，也就是可中断状态睡眠，表示进程因为等待某个事件而被系统挂起。当进程等待的事件发生时，它会被唤醒并进入 R 状态。

- I 是 Idle 的缩写，也就是空闲状态，用在不可中断睡眠的内核线程上。前面说了，硬件交互导致的不可中断进程用 D 表示，但对某些内核线程来说，它们有可能实际上并没有任何负载，用 Idle 正是为了区分这种情况。要注意，D 状态的进程会导致平均负载升高， I 状态的进程却不会。

- T/t 是Stopped/Traced缩写，表示进程处于暂停或追踪状态。

- X Dead缩写，表示进程已经消亡，在top和ps里是看不到这个状态的。


#### 中断的2种阶段

- 上半部直接处理硬件请求，也就是我们常说的硬中断，特点是快速执行；

- 而下半部则是由内核触发，也就是我们常说的软中断，特点是延迟执行。


#### CPU cache,性能指标

![cpu cache](https://static001.geekbang.org/resource/image/aa/33/aa08816b60e453b52b5fae5e63549e33.png)

![cpu 指标](https://static001.geekbang.org/resource/image/1e/07/1e66612e0022cd6c17847f3ab6989007.png)

#### CPU 性能工具

![工具](https://static001.geekbang.org/resource/image/59/ec/596397e1d6335d2990f70427ad4b14ec.png)

![工具](https://static001.geekbang.org/resource/image/b0/ca/b0c67a7196f5ca4cc58f14f959a364ca.png)

粗线条查看问题
![工具](https://static001.geekbang.org/resource/image/7a/17/7a445960a4bc0a58a02e1bc75648aa17.png)

# 内存篇

### 内存映射

只有内核才可以直接访问物理内存。Linux 内核给每个进程都提供了一个独立的虚拟地址空间，并且这个地址空间是连续的。这样，进程就可以很方便地访问内存，更确切地说是访问虚拟内存。

虚拟地址空间的内部又被分为内核空间和用户空间两部分，不同字长（也就是单个 CPU 指令可以处理数据的最大长度）的处理器，地址空间的范围也不同。

![32位、64位内存地址映射](https://static001.geekbang.org/resource/image/ed/7b/ed8824c7a2e4020e2fdd2a104c70ab7b.png)

进程在用户态时，只能访问用户空间内存；只有进入内核态后，才可以访问内核空间内存。虽然每个进程的地址空间都包含了内核空间，但这些内核空间，其实关联的都是相同的物理内存。这样，进程切换到内核态后，就可以很方便地访问内核空间内存。

虚拟内存和物理内存映射

![虚拟地址和物理地址映射](https://static001.geekbang.org/resource/image/fc/b6/fcfbe2f8eb7c6090d82bf93ecdc1f0b6.png)

四级页表

![四级页表](https://static001.geekbang.org/resource/image/b5/25/b5c9179ac64eb5c7ca26448065728325.png)


虚拟内存空间分布

![32位](https://static001.geekbang.org/resource/image/71/5d/71a754523386cc75f4456a5eabc93c5d.png)


- 只读段，包括代码和常量等。

- 数据段，包括全局变量等。

- 堆，包括动态分配的内存，从低地址开始向上增长。

- 文件映射段，包括动态库、共享内存等，从高地址开始向下增长。

- 栈，包括局部变量和函数调用的上下文等。栈的大小是固定的，一般是 8 MB。查看linux栈的大小： ulimit -s 

以上堆和文件映射段的内存是动态分配的，如c标准库的malloc(), mmap() 


free 工具

```bash
$ free
             total       used       free     shared    buffers     cached
Mem:       7917296    6428940    1488356     345932     111576    3470220
-/+ buffers/cache:    2847144    5070152
Swap:            0          0          0

$ free -h
             total       used       free     shared    buffers     cached
Mem:          7.6G       6.1G       1.4G       337M       109M       3.3G
-/+ buffers/cache:       2.7G       4.9G
Swap:           0B         0B         0B
```

- 第一列，total 是总内存大小；

- 第二列，used 是已使用内存的大小，包含了共享内存；

- 第三列，free 是未使用内存的大小；

- 第四列，shared 是共享内存的大小；

- 第五列，buff/cache 是缓存和缓冲区的大小；

- 最后一列，available 是新进程可用内存的大小。

Buffers vs Cache

- Buffers 是内核缓冲区用到的内存，对应的是 /proc/meminfo 中的 Buffers 值。

Buffer 既可以用作“将要写入磁盘数据的缓存”，也可以用作“从磁盘读取数据的缓存”。


- Cache 是内核页缓存和 Slab 用到的内存，对应的是 /proc/meminfo 中的 Cached 与 SReclaimable 之和。

Cache 既可以用作“从文件读取数据的页缓存”，也可以用作“写文件的页缓存”。

**numactl 命令**

来查看处理器在 Node 的分布情况，以及每个 Node 的内存使用情况。

**numa策略调整**

/proc/sys/vm/zone_reclaim_mode

- 默认的 0 ，既可以从其他 Node 寻找空闲内存，也可以从本地回收内存。

- 1、2、4 都表示只回收本地内存，2 表示可以回写脏数据回收内存，4 表示可以用 Swap 方式回收内存。


/proc/zoneinfo 内存阈值信息

**swap相关**

- 修改积极程度 ， /proc/sys/vm/swapiness 
- 启用， swapon
- 关闭 ， swapoff 


#### 工具图谱

![](https://static001.geekbang.org/resource/image/e2/36/e28cf90f0b137574bca170984d1e6736.png)
![](https://static001.geekbang.org/resource/image/8f/ed/8f477035fc4348a1f80bde3117a7dfed.png)
![](https://static001.geekbang.org/resource/image/52/9b/52bb55fba133401889206d02c224769b.png)
![](https://static001.geekbang.org/resource/image/d7/fe/d79cd017f0c90b84a36e70a3c5dccffe.png)



# I/O 篇

**扇区、块、页(主要是HDD，SSD与之不同)**

- 扇区

磁盘的最下存储单位，一般是512byte,查看扇区的方式

```bash
#fdisk -l
Disk /dev/vda: 42.9 GB, 42949672960 bytes
16 heads, 63 sectors/track, 83220 cylinders
Units = cylinders of 1008 * 512 = 516096 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk identifier: 0x00038e27

   Device Boot      Start         End      Blocks   Id  System
/dev/vda1   *           3        1018      512000   83  Linux
Partition 1 does not end on cylinder boundary.
/dev/vda2            1018       83221    41430016   8e  Linux LVM
Partition 2 does not end on cylinder boundary.
```
可以看到512bytes是扇区的大小。

- 块

文件系统读写数据的最小单位；一般一个块由8个扇区组成，即4k bytes。

查看块大小的方式：

```bash
# stat /etc/passwd|grep 'IO Block'
  Size: 936       	Blocks: 8          IO Block: 4096   普通文件
```


- 页

**内存**的最小存储单位， 一般也是4k，与磁盘的块大小保持一致，以便取得做好的映射效率。

查看块大小的方式：

```bash
getconf PAGE_SIZE
```

![](https://pic002.cnblogs.com/images/2012/295881/2012052117213095.gif)
![](https://pic002.cnblogs.com/images/2012/295881/2012052117224297.gif)
![](https://pic002.cnblogs.com/images/2012/295881/2012052117250525.gif)


![](https://static001.geekbang.org/resource/image/32/47/328d942a38230a973f11bae67307be47.png)

存储结构

- 超级块， 存储整个文件系统的状态

- 索引节点区， 用来存储索引节点（i node)

- 数据块区， 用来存储文件数据

### 文件系统 and 虚拟文件系统

文件系统由 目录项、 索引节点、 逻辑块、 超级块 四大要素组成。

为了支持不同的文件系统，Linux内核在用户进程和文件系统之间引入了一个抽象层，这个称为虚拟文件系统（VFS).

用户进程 ---> VFS ------> FS


VFS 定义了一组所有文件系统都支持的数据结构和标准接口，这样对用户程序来说就屏蔽了底层不同文件系统的实现差异了。

![](https://static001.geekbang.org/resource/image/72/12/728b7b39252a1e23a7a223cdf4aa1612.png)

### I/O 模型

文件读写方式的各种差异，导致 I/O 的分类多种多样。最常见的有，

- 缓冲与非缓冲 I/O
- 直接与非直接 I/O
- 阻塞与非阻塞 I/O
- 同步与异步 I/O 

**根据是否利用标准库缓存来区分：**

- 缓冲 I/O，是指利用标准库缓存来加速文件的访问，而标准库内部再通过系统调度访问文件。

- 非缓冲 I/O，是指直接通过系统调用来访问文件，不再经过标准库缓存。


**根据是否利用操作系统的页缓存来区分：**


- 直接 I/O，是指跳过操作系统的页缓存，直接跟文件系统交互来访问文件。

- 非直接 I/O 正好相反，文件读写时，先要经过系统的页缓存，然后再由内核或额外的系统调用，真正写入磁盘。

_想要实现直接 I/O，需要你在系统调用中，指定 O_DIRECT 标志。如果没有设置过，默认的是非直接 I/O。_


**根据应用程序是否阻塞自身运行：**


- 所谓阻塞 I/O，是指应用程序执行 I/O 操作后，如果没有获得响应，就会阻塞当前线程，自然就不能执行其他任务。

- 所谓非阻塞 I/O，是指应用程序执行 I/O 操作后，不会阻塞当前的线程，可以继续执行其他的任务，随后再通过轮询或者事件通知的形式，获取调用的结果。

_访问管道或者网络套接字时，设置 O_NONBLOCK 标志，就表示用非阻塞方式访问；而如果不做任何设置，默认的就是阻塞访问。_


**根据是否等待响应结果:**

- 所谓同步 I/O，是指应用程序执行 I/O 操作后，要一直等到整个 I/O 完成后，才能获得 I/O 响应。

- 所谓异步 I/O，是指应用程序执行 I/O 操作后，不用等待完成和完成后的响应，而是继续执行就可以。等到这次 I/O 完成后，响应会用事件通知的方式，告诉应用程序。

举个例子，在操作文件时，如果你设置了 O_SYNC 或者 O_DSYNC 标志，就代表同步 I/O。如果设置了 O_DSYNC，就要等文件数据写入磁盘后，才能返回；而 O_SYNC，则是在 O_DSYNC 基础上，要求文件元数据也要写入磁盘后，才能返回。

再比如，在访问管道或者网络套接字时，设置了 O_ASYNC 选项后，相应的 I/O 就是异步 I/O。这样，内核会再通过 SIGIO 或者 SIGPOLL，来通知进程文件是否可读写。


### I/O 调度


- 第一种 NONE ，更确切来说，并不能算 I/O 调度算法。因为它完全不使用任何 I/O 调度器，对文件系统和应用程序的 I/O 其实不做任何处理，常用在虚拟机中（此时磁盘 I/O 调度完全由物理机负责）。

- 第二种 NOOP ，是最简单的一种 I/O 调度算法。它实际上是一个先入先出的队列，只做一些最基本的请求合并，常用于 SSD 磁盘。

- 第三种 CFQ（Completely Fair Scheduler），也被称为完全公平调度器，是现在很多发行版的默认 I/O 调度器，它为每个进程维护了一个 I/O 调度队列，并按照时间片来均匀分布每个进程的 I/O 请求。

-  DeadLine 调度算法，分别为读、写请求创建了不同的 I/O 队列，可以提高机械磁盘的吞吐量，并确保达到最终期限（deadline）的请求被优先处理。DeadLine 调度算法，多用在 I/O 压力比较重的场景，比如数据库等


### I/O 栈

![](https://static001.geekbang.org/resource/image/14/b1/14bc3d26efe093d3eada173f869146b1.png)

iostat

![](https://static001.geekbang.org/resource/image/cf/8d/cff31e715af51c9cb8085ce1bb48318d.png)

![](https://static001.geekbang.org/resource/image/b6/20/b6d67150e471e1340a6f3c3dc3ba0120.png)

![](https://static001.geekbang.org/resource/image/6f/98/6f26fa18a73458764fcda00212006698.png)

![](https://static001.geekbang.org/resource/image/c4/e9/c48b6664c6d334695ed881d5047446e9.png)

![](https://static001.geekbang.org/resource/image/18/8a/1802a35475ee2755fb45aec55ed2d98a.png)

