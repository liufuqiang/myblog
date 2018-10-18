---
title: "Linux的几种I/O模型的介绍"
date: 2018-10-18T18:02:45+08:00
tags: ["I/O","epoll","select","aio","高性能"]
---



### 概述
在高性能编程和优化领域经常会提及到同步、异步，阻塞、非阻塞的概念，如会涉及到为什么PHP的性能上不去，为什么Nginx和Golang比较高性能等等，其实这里边讲的是关于网络编程里的各种I/O模型的知识，如果能够较好的理解多个I/O模型，对之后理解高性能的优化会有很大帮助。我们先基于一些和I/O相关的概念入手，逐步讨论下各个I/O模型的一些知识,如会介绍经到什么是用户空间和内核空间、为什么进程切换成本会很高、文件描述符是个什么东东等一些和话题有关的问题。

### 先要了解的一些概念

#### 用户空间和内核空间
现代的操作系统采用虚拟存储器的技术(这里想深入了解可以搜索保护模式、实模式相关知识），如32位操作系统的寻址空间（虚拟存储空间）为4G（2的32次方）。操作系统的核心是kernel（内核）程序，它独立于普通的应用程序（用户程序），内核可以访问受保护的内存空间，也有访问底层硬件设备的所有权限。
为了保证用户进程不能直接操作内核，操作系统将虚拟空间划分为两部分，一部分为内核空间，一部分为用户空间。
针对linux操作系统而言，将最高的1G字节（从虚拟地址0xC0000000到0xFFFFFFFF）供内核使用，称为内核空间，而将较低的3G字节（从虚拟地址0x00000000到0xBFFFFFFF），供各个进程使用，称为用户空间。

如下图示意:

![image](http://p0.qhimg.com/t0130b31b5c98713b51.png)

在64位地址模式里，pgd、pud、pmd、pte各占了9位，加上12位的页内index，共用了48位。**即可管理的地址空间**为2^48=256T。支持的**物理内存**最大为64T，见e820.c中MAX_ARCH_PFN的定义：
```c
# define MAX_ARCH_PFN MAXMEM>>PAGE_SHIFT
```
其中MAXMEM为2^46，PAGE_SHIFT为12。而在32位地址时最大支持的**物理内存**为64G（开启[PAE](https://baike.baidu.com/item/PAE%E6%A8%A1%E5%BC%8F)选项）

#### 进程切换,成本不低
为了控制进程的执行，内核必须有能力挂起正在CPU上运行的进程，并恢复以前挂起的某个进程的执行。这种行为被称为进程切换。因此可以说，任何进程都是在操作系统内核的支持下运行的，是与内核紧密相关的。
内核把当前运行在CPU的一个进程的切换到另一个进程上运行，有以下操作要做：
1. 保存CPU上下文信息，如括PC（程序计数器）和其他寄存器（PSW，SP等）。
2. 更新PCB信息。
3. 把进程的PCB Push到相应的队列，如就绪、阻塞等队列。
3. 选择另一个进程执行，并更新其PCB。
4. 更新内存管理的数据结构。
5. 恢复CPU上下文信息

#### 进程的阻塞,让出CPU
正在运行中的进程，由于等待的某些事件未发生，如请求系统资源失败、等待某种操作的完成、新数据尚未到达或无新工作做等，则由系统自动执行阻塞原语(PV原语，P是阻塞原语)，使自己由运行状态变为阻塞状态。可见，进程的阻塞是进程自身的一种主动行为，也因此只有处于运行态的进程，才可能将其转为阻塞状态。当进程进入阻塞状态，是不占用CPU资源的。

> **阻塞和唤醒原语（PV原语)**:
PV原语通过操作信号量来处理进程间的同步与互斥的问题。 \
**P原语**：P是荷兰语Proberen（测试）的首字母。为阻塞原语，负责把当前进程由运行状态转换为阻塞状态，直到另外一个进程唤醒它。操作为：申请一个空闲资源（把信号量减1），若成功，则退出；若失败，则该进程被阻塞。\
**V原语**：V是荷兰语Verhogen（增加）的首字母。为唤醒原语，负责把一个被阻塞的进程唤醒，它有一个参数表，存放着等待被唤醒的进程信息。操作为：释放一个被占用的资源（把信号量加1），如果发现有被阻塞的进程，则选择一个唤醒之。

#### 文件描述符,FD
文件描述符（File descriptor）是计算机科学中的一个术语，是一个用于表述指向文件的引用的抽象化概念。

文件描述符在形式上是一个非负整数。实际上，它是一个索引值，指向内核为每一个进程所维护的该进程打开文件的记录表。当程序打开一个现有文件或者创建一个新文件时，内核向进程返回一个文件描述符。在程序设计中，一些涉及底层的程序编写往往会围绕着文件描述符展开。但是文件描述符这一概念往往只适用于UNIX、Linux这样的操作系统。

Linux内核维护的 3 个数据结构：

1. 进程级 文件描述符表 ( file descriptor table )
2. 系统级 打开文件表 ( open file table )
3. 文件系统 i-node表 ( i-node table )

三者的关系如图：
![image](https://learn-linux.readthedocs.io/zh_CN/latest/_images/26cf89b03f383aaac00e1da084d6c909.png)

> 文件描述符复制： 如图中A进程的fd 1和fd 20都指向打开文件表里的23，可能是调用了dup()系列函数产生的。\
子进程继承父进程文件描述符： 如A进程的fd 2和B进程fd 2都指向打开文件的73，有可能B是A进程fork出来的进程。

#### 缓存I/O (buffered I/O)

缓存I/O(即buffered I/O,是相对于unbuffered I/O而言的) 又被称作标准I/O，大多数文件系统的默认IO操作都是缓存IO。在Linux 的缓存IO机制中，操作系统会将IO 的数据缓存在文件系统的页缓存（page cache）中，也就是说，数据会先被拷贝到操作系统内核的缓冲区中，然后才会从操作系统内核的缓冲区拷贝到应用程序的地址空间。

缓存I/O，非缓存I/O和物理存储的关系是：
![image](http://p0.qhimg.com/t01a5c5b42dc3447e4a.png)

> **缓存I/O为什么效率高**：\
首先要确定一个观点，由于库函数调用相比系统调用少了用户态和内核态的切换，所以库函数调用的效率会更好一些。
buffered I/O中都是库函数,而unbuffered I/O中为系统调用,所以，buffered I/O会好一些。\
buffered I/O就是通过尽可能的少使用系统调用来提高效率的.它的基本方法是,在用户进程空间维护一块缓冲区,第一次读(库函数)的时候用read(系统调用)多从内核读出一些数据,下次在要读(库函数)数据的时候,先从该缓冲区读,而不用进行再次read(系统调用)了.同样,写的时候,先将数据写入(库函数)一个缓冲区,多次以后,在集中进行一次write(系统调用),写入内核空间


### Linux I/O模型

一般我们把I/O模型分为以下几种类型：

1. Synchronous I/O （同步I/O)
2. Asynchronous I/O (异步I/O)
3. Blocking I/O     (阻塞I/O)
4. Non-blocking I/O (非阻塞I/O)
5. I/O multiplexing (I/O 多路复用)
6. Signal driven I/O （信号驱动 I/O)


一般讲到I/O模型，都是指的用户空间的I/O，而且都是偏重于网络编程中的I/O (socket I/O)，它需要经历2个阶段操作：

1. 等待数据准备就绪（waiting for the data to be ready）
2. 将数据从内核拷贝到进程中（copying the data from kernel to the process）

下面我们来进行更多介绍，开始之前先来说下网络I/O的一些概念。

**网络I/O** 的本质是socket的读取，socket在linux系统被抽象为流，I/O可以理解为对流的操作。上边介绍过，对于一次I/O访问（以read举例），数据会先被拷贝到操作系统内核的缓冲区中，然后才会从操作系统内核的缓冲区拷贝到应用程序(进程）的地址空间。


#### 阻塞I/O (Blocking I/O)

在Linux中所有的socket文件描述符默认都是blocking的，一个典型的网络socket文件描述符的read 操作的流程如下图所示。
![image](http://p0.qhimg.com/t01f7dab525c723ac7f.png)

当用户进程调用 read这个系统调用，kernel 就开始了 I/O 的第一个阶段：准备数据（对于网络IO来说，很多时候数据在一开始还没有到达。比如，还没有收到一个完整的UDP包。这个时候 kernel就要等待足够的数据到来），这个过程需要等待，也就是说数据被拷贝到操作系统内核的缓冲区中是需要一个过程的。

而在用户进程这边，整个进程一直处于阻塞状态。当 kernel中数据准备好之后，系统就会将数据从 kernel中拷贝到用户空间进程的缓冲区中，然后 kernel返回结果，用户进程才解除blocking 的状态，重新运行起来。

**blocking I/O 的特点是用户进程在 I/O 操作的两个阶段都被 block 了**。

#### 非阻塞I/O (Non-blocking I/O)

Linux 系统中可以在调用open函数打开 socket文件时或者调用fcntl函数来设置socket文件描述符为非阻塞状态，当对一个 non-blocking 的 socket文件描述符执行读操作时，它的执行流程如下图所示：
![image](http://p0.qhimg.com/t013532127fea00c0a6.png)

在 non-blocking I/O的情境下，当用户进程调用 read系统调用函数时，如果 kernel中的数据没有准备好，那么系统并不会 block用户进程，而是立即返回一个 error。

从用户进程角度看 ，当它发起一个read 操作后，并不需要等待，而是马上就得到了一个结果。用户进程判断结果是一个 error 时，它就知道数据还没有准备好，于是它可以再次发送read操作(期间也可以干点其他工作)。
一旦 kernel中的数据准备好之后，并且再次收到了用户进程发送的 read 系统调用时，它就马上将准备好的数据拷贝到用户空间的缓冲区中。（就是说内核不会主动通知用户进程，需要用户进程主动来调用才行）

**non-blocking I/O的特点就是用户进程需要不断地询问kernel 数据是否准备好了。**

#### I/O多路复用 (I/O Multiplexing)

**I/O 多路复用**就是我们经常说的 select、poll和epoll，有些地方也称这种 I/O 方式为 event driven I/O。I/O 多路复用的好处就是一个进程可以处理多个socket I/O，它的工作原理就是select/poll/epoll 函数会不断的查询所监测的socket 文件描述符中是否有socket准备好读写了，如果有，那么系统就会通知用户进程来处理。

I/O多路复用的特点是通过一种机制让一个进程能同时等待多个文件描述符，而这些文件描述符（套接字描述符）其中的任意一个进入可读的就绪状态，select/poll/epoll函数就可以立即返回，流程如下图：
![image](http://p0.qhimg.com/t013201a0a0dea1b189.png)

这个图和 blocking I/O的图其实并没有太大的不同，事实上，开销还会更大一点。因为这里需要使用两个 system call (select 和 read)，而 blocking I/O 只调用了一个system call (read)。但是，用select的优势在于它可以同时处理多个connection。

所以，如果处理的连接数不是很高的话，使用select/epoll 的 web server 不一定比使用 multi-threading + blocking I/O 的 web server性能更好，可能延迟还更大。**select/epoll的优势并不是对于单个连接能处理得更快，而是在于能处理更多的连接。**

在 I/O 多路复用模型中，实际中，对于每一个socket，一般都设置成为non-blocking，但是，如上图所示，整个用户的process 其实是一直被 block的。只不过用户进程是被 select 这个函数block，而不是被 socket I/O 给 block。

以下是epoll 模型的流程：
![image](https://tech.youzan.com/content/images/2017/04/io---1.png)

**我们在来重点看下select、poll和epoll的对比和各自优缺点。**

select，poll，epoll本质上都是同步I/O，因为他们都需要在读写事件就绪后自己负责进行读写，也就是说这个读写过程是阻塞的，而异步I/O则无需自己负责进行读写，异步I/O的实现会负责把数据从内核拷贝到用户空间。

##### select

原理:

select 的核心功能是调用linux文件系统的poll方法(f_op->poll())，不停的查询，如果没有想要的数据，主动执行一次调度（防止一直占用cpu），直到有一个连接获取到想要的消息为止。可以看出select的执行方式基本就是不停的调用f_op->poll(),直到有需要的消息为止。

优点:

1.	select的可移植性更好，在某些Unix系统上不支持poll和epoll。
2. select对于超时值提供了更好的精度：微秒，而poll是毫秒。

缺点: 

1. 每次调用select，都需要把fd集合从用户态拷贝到内核态，这个开销在fd很多时会很大；
2. 同时每次调用select都需要在内核遍历传递进来的所有fd，这个开销在fd很多时也很大；
3. select支持的文件描述符数量太小了，默认是1024。

##### poll 

原理:

poll本质上和select没有区别，它将用户传入的fd数组拷贝到内核空间，然后查询每个fd对应的设备状态，如果设备就绪则在设备等待队列中加入一项并继续遍历，如果遍历完所有fd后没有发现就绪设备，则挂起当前进程，直到设备就绪或者主动超时，被唤醒后它又要再次遍历fd。这个过程经历了多次无谓的遍历。poll还有一个特点是“水平触发”，如果报告了fd后，没有被处理，那么下次poll时会再次报告该fd。

优点:

1. poll() 不要求开发者计算最大文件描述符+1的大小。
2. 如select相比，poll()在应付大数目的文件描述符的时候速度更快
3. 它没有最大连接数的限制，原因是它是基于链表来存储的。

缺点:

1. 大量的fd的数组被整体复制于用户态和内核地址空间之间，而不管这样的复制是不是有意义；
2. 与select一样，poll返回后，需要轮询poll fd来获取就绪的描述符。

##### epoll 

原理:

epoll同样只告知那些就绪的文件描述符，而且当我们调用epoll_wait()获得就绪文件描述符时，返回的不是实际的描述符，而是一个代表就绪描述符数量的值，你只需要去epoll指定的一个数组中依次取得相应数量的文件描述符即可，这里也使用了内存映射技术（mmap)，这样便彻底省掉了这些文件描述符在系统调用时复制的开销。 

优点:

1. 支持一个进程打开大数目的socket描述符：相比select，epoll则没有对FD的限制，它所支持的FD上限是最大可以打开文件的数目，这个数字一般远大于2048,举个例子,在1GB内存的机器上大约是10万左右，具体数目可以cat /proc/sys/fs/file-max察看,一般来说这个数目和系统内存关系很大。
2. I/O效率不随FD数目增加而线性下降：epoll不存在这个问题，它只会对"活跃"的socket进行操作,这是因为在内核实现中epoll是根据每个fd上面的callback函数实现的。那么，只有"活跃"的socket才会主动的去调用callback函数，其他idle状态socket则不会，在这点上，epoll实现了一个"伪"AIO，因为这时候推动力在os内核。
3. 使用mmap加速内核与用户空间的消息传递：这点实际上涉及到epoll的具体实现了。无论是select,poll还是epoll都需要内核把FD消息通知给用户空间，如何避免不必要的内存拷贝就很重要，在这点上，epoll是通过内核于用户空间mmap同一块内存实现的。

缺点:

如果所有的socket基本上都是活跃的,比如一个高速LAN环境，epoll并不比select/poll有什么效率，相反，如果过多使用epoll_ctl,效率相比还有稍微的下降。但是一旦使用idle connections模拟WAN环境,epoll的效率就远在select/poll之上了。

以下是Select和epoll的流程对比图：

select模式：<br>
![image](http://img.chuansong.me/mmbiz_png/KyXfCrME6ULnoC94BK755Zed7WWuiaH5gjiclzgAFTbwynic5fRqxiczKGhnOwEyHb5ib4SLzA7CpT8XGFdSu7JbOfA/0?wx_fmt=png)

epoll模式：<br>
![image](http://img.chuansong.me/mmbiz_png/KyXfCrME6ULnoC94BK755Zed7WWuiaH5gyldylVvbVzicuASdz3HmwJC5DNBiabG5YLPHibAsbWwNUuSSUxIG9icwzA/0?wx_fmt=png)

> **关于文件系统的poll方法** \
poll/select/epoll的实现都是基于文件提供的poll方法(f_op->poll)，
该方法利用poll_table提供的_qproc方法向文件内部事件掩码_key对应的的一个或多个等待队列(wait_queue_head_t)上添加包含唤醒函数(wait_queue_t.func)的节点(wait_queue_t)，并检查文件当前就绪的状态返回给poll的调用者(依赖于文件的实现)。
当文件的状态发生改变时(例如网络数据包到达)，文件就会遍历事件对应的等待队列并调用回调函数(wait_queue_t.func)唤醒等待线程。


#### 信号驱动I/O (Signal Driven I/O)

Signal Driven I/O 的工作原理就是用户进程首先和 kernel 之间建立信号的通知机制，即用户进程告诉 kernel，如果 kernel 中数据准备好了，就通过 SIGIO 信号通知我。然后用户空间的进程就会调用 read 系统调用将准备好的数据从 kernel 拷贝到用户空间。

但是这种 I/O 模型存在一个非常重大的缺陷问题：**SIGIO 这种信号对于每个进程来说只有一个！**如果使该信号对进程中的两个描述符（这两个文件描述符都等待着 I/O 操作）都起作用，那么进程在接到此信号后就无法判别是哪一个文件描述符准备好了。所以 Signal Driven I/O 模型在现实中用的非常少。模型如图：

![image](https://tech.youzan.com/content/images/2017/04/------1.png)

#### 异步I/O (Asynchronous I/O)

POSIX规范定义了一组异步操作I/O的接口，不用关心fd 是阻塞还是非阻塞，异步I/O是由内核接管应用层对fd的I/O操作。异步I/O向应用层通知I/O操作完成的事件，这与前面介绍的I/O 复用模型、SIGIO模型通知事件就绪的方式明显不同。以aio_read 实现异步读取IO数据为例，如下图所示，在等待I/O操作完成期间，不会阻塞应用程序。

![image](https://tech.youzan.com/content/images/2017/04/--io-1.png)

用户进程通过 aio_read()函数进行读取操作时，可以立刻返回到进程中，接着执行其他的操作。从 kernel 的角度来看，当它收到 asynchronous read操作时，它会立刻返回，并不会阻塞用户进程。然后，kernel会等待数据准备完成，接着将数据拷贝到用户空间进程的缓冲区中。当这一切都完成之后，kernel 会给用户发送一个 signal通知用户空间的进程，告诉它 read操作完成了。

**Asynchronous I/O 操作最大的特点就是整个 I/O 操作流程中，用户进程始终没有被 block。**
**AIO是个标准，不同OS的实现不一样**
> epoll+nonblock从宏观角度可以叫做全异步，但从微观的角度来看还是同步的IO。只是在数据到达后得到系统通知，然后同步执行recv取回数据，没有iowait。\
真正的异步IO即AIO应该像Windows IOCP一样，传入文件句柄，缓存区，尺寸等参数和一个函数指针，当操作系统真正完成了IO操作，再执行对应的函数。详情可以看[这篇文章](http://rango.swoole.com/archives/282)

#### 同步I/O (Synchronous I/O)

其实我们介绍的阻塞I/O,非阻塞I/O,多路复用I/O 都属于同步I/O， 可以看下POSIX对同步I/O和异步I/O的定义:

1. A synchronous I/O operation causes the requesting process to be blocked until that I/O operation completes;
2. An asynchronous I/O operation does not cause the requesting process to be blocked;

两者的区别就在于 同步I/O做IO操作的时候会将用户空间的进程阻塞。比如，就非阻塞I/O来说，它在执行read系统调用的时候，如果内核空间的数据没有就绪，这时候不会阻塞用户进程，但是当数据就绪后，read会将数据从内核态copy到用户空间，这段时间里用户进程是被block的。

### 参考资料
- https://www.jianshu.com/p/486b0965c296
- https://blog.csdn.net/u014530704/article/details/77090346
- https://learn-linux.readthedocs.io/zh_CN/latest/system-programming/file-io/file-descriptor.html
- http://blog.51cto.com/4983206/1142074
- https://woshijpf.github.io/linux/2017/07/10/Linux-IO%E6%A8%A1%E5%9E%8B.html 
- https://tech.youzan.com/yi-bu-wang-luo-mo-xing/
- http://blog.51cto.com/luminous/1832114
- https://blog.csdn.net/lishenglong666/article/details/45536611
- http://chuansong.me/n/1392417151737
- http://rango.swoole.com/archives/282

