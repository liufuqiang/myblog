---
title: "关于突破tcp连接限制的思考"
date: 2018-09-05T10:43:49+08:00
tags: ["c10k","高并发","tcp"]
---

## 概述
昨天在对接入层系统目前的问题进行review时，讨论到目前随着接入层系统的业务变多，目前已经有 20w/s 的qps了，会遇到服务器上报本地端口不够用的问题。但这个问题其实是一个比较复杂的优化点，涉及很多环节，那今天我来把这块大致的进行阐述下。

## 要了解的知识1: ulimit
ulimit是系统用来限制用户对系统资源的访问控制的，目的是尽最大可能去保护系统资源的合理使用，不至于被透支而导致系统崩溃。
ulimit支持以下各种类型的限制：

- 所创建的内核文件的大小
- 进程数据块的大小
- Shell 进程创建文件的大小
- 内存锁住的大小
- 常驻内存集的大小
- 打开文件描述符的数量
- 分配堆栈的最大大小
- CPU 时间
- 单个用户的最大线程数
- Shell 进程所能使用的最大虚拟内存。
同时，它支持硬资源（hard)和软资源(soft)的限制。
说重点， 一个TCP连接其实就是一个Socket文件（在linux里一切都是文件），而缺省情况下ulimit 里的nofile是1024，抛去每个进程必然打开的标准输入，标准输出，标准错误，服务器监听 socket，进程间通讯的unix域socket等文件，那么剩下的可用于客户端socket连接的文件数就只有大概1024-10=1014个左右，那就意味着缺省的操作系统配置下基于Linux的通讯程序最多允许同时1014个TCP并发连接。

所以，对于想支持更高数量的TCP并发连接的通讯处理程序，就必须修改Linux对当前用户的进程同时打开的文件数量的软限制(soft limit)和硬限制(hardlimit)。
```bash
echo '* soft nofile 1048576' >> /etc/security/limits.conf
echo '* hard nofile 1048576' >> /etc/security/limits.conf
```

## 要了解的知识2：sysctl
sysctl命令被用于在内核运行时动态地修改内核的运行参数，可用的内核参数在目录/proc/sys中。它包含一些TCP/ip堆栈和虚拟内存系统的高级选项， 这可以让有经验的管理员提高引人注目的系统性能。用sysctl可以读取设置超过五百个系统变量。文件位于：/etc/sysctl.conf
作为服务端，为了能够支持较高的连接数，我们需要优化下几个系统内核参数：

```bash
sysctl -w fs.file-max=10485760 #系统允许的文件描述符数量10m , 注意区分ulimit 里的nofile是用户级别的
sysctl -w net.ipv4.tcp_rmem=1024 #每个tcp连接的读取缓冲区1k，一个连接1k
sysctl -w net.ipv4.tcp_wmem=1024 #每个tcp连接的写入缓冲区1k
#修改默认的本地端口范围
sysctl -w net.ipv4.ip_local_port_range='1024 65535'
sysctl -w net.ipv4.tcp_tw_recycle=1  #快速回收time_wait的连接
sysctl -w net.ipv4.tcp_tw_reuse=1
sysctl -w net.ipv4.tcp_timestamps=1
```
对于服务端角色而言，能够抗多少的连接数，基本上是取决于机器的内存和网络带宽的。这里我们列出一些和网络相关的内核参数以便参考。 实际工作上如果需要对参数修改，务必要自己亲自多轮测试才行，不要盲从。[引用资料](https://pathbox.github.io/2018/02/06/65535-port-and-concurrent-socket/)

```bash
net.core.netdev_max_backlog = 400000
#该参数决定了，网络设备接收数据包的速率比内核处理这些包的速率快时，允许送到队列的数据包的最大数目。

net.core.optmem_max = 10000000
#该参数指定了每个套接字所允许的最大缓冲区的大小

net.core.rmem_default = 10000000
#指定了接收套接字缓冲区大小的缺省值（以字节为单位）。

net.core.rmem_max = 10000000
#指定了接收套接字缓冲区大小的最大值（以字节为单位）。

net.core.somaxconn = 100000
#Linux kernel参数，表示socket监听的backlog(监听队列)上限

net.core.wmem_default = 11059200
#定义默认的发送窗口大小；对于更大的 BDP 来说，这个大小也应该更大。

net.core.wmem_max = 11059200
#定义发送窗口的最大大小；对于更大的 BDP 来说，这个大小也应该更大。

net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1
#严谨模式 1 (推荐)
#松散模式 0

net.ipv4.tcp_congestion_control = bic
#默认推荐设置是 htcp

net.ipv4.tcp_window_scaling = 0
#关闭tcp_window_scaling
#启用 RFC 1323 定义的 window scaling；要支持超过 64KB 的窗口，必须启用该值。

net.ipv4.tcp_ecn = 0
#把TCP的直接拥塞通告(tcp_ecn)关掉

net.ipv4.tcp_sack = 1
#关闭tcp_sack
#启用有选择的应答（Selective Acknowledgment），
#这可以通过有选择地应答乱序接收到的报文来提高性能（这样可以让发送者只发送丢失的报文段）；
#（对于广域网通信来说）这个选项应该启用，但是这会增加对 CPU 的占用。

net.ipv4.tcp_max_tw_buckets = 10000
#表示系统同时保持TIME_WAIT套接字的最大数量

net.ipv4.tcp_max_syn_backlog = 8192
#表示SYN队列长度，默认1024，改成8192，可以容纳更多等待连接的网络连接数。

net.ipv4.tcp_syncookies = 1
#表示开启SYN Cookies。当出现SYN等待队列溢出时，启用cookies来处理，可防范少量SYN攻击，默认为0，表示关闭；

net.ipv4.tcp_timestamps = 1
#开启TCP时间戳
#以一种比重发超时更精确的方法（请参阅 RFC 1323）来启用对 RTT 的计算；为了实现更好的性能应该启用这个选项。

net.ipv4.tcp_tw_reuse = 1
#表示开启重用。允许将TIME-WAIT sockets重新用于新的TCP连接，默认为0，表示关闭；

net.ipv4.tcp_tw_recycle = 1
#表示开启TCP连接中TIME-WAIT sockets的快速回收，默认为0，表示关闭。

net.ipv4.tcp_fin_timeout = 10
#表示如果套接字由本端要求关闭，这个参数决定了它保持在FIN-WAIT-2状态的时间。

net.ipv4.tcp_keepalive_time = 1800
#表示当keepalive起用的时候，TCP发送keepalive消息的频度。缺省是2小时，改为30分钟。

net.ipv4.tcp_keepalive_probes = 3
#如果对方不予应答，探测包的发送次数

net.ipv4.tcp_keepalive_intvl = 15
#keepalive探测包的发送间隔

net.ipv4.tcp_mem
#确定 TCP 栈应该如何反映内存使用；每个值的单位都是内存页（通常是 4KB）。
#第一个值是内存使用的下限。
#第二个值是内存压力模式开始对缓冲区使用应用压力的上限。
#第三个值是内存上限。在这个层次上可以将报文丢弃，从而减少对内存的使用。对于较大的 BDP 可以增大这些值（但是要记住，其单位是内存页，而不是字节）。

net.ipv4.tcp_rmem
#与 tcp_wmem 类似，不过它表示的是为自动调优所使用的接收缓冲区的值。

net.ipv4.tcp_wmem = 30000000 30000000 30000000
#为自动调优定义每个 socket 使用的内存。
#第一个值是为 socket 的发送缓冲区分配的最少字节数。
#第二个值是默认值（该值会被 wmem_default 覆盖），缓冲区在系统负载不重的情况下可以增长到这个值。
#第三个值是发送缓冲区空间的最大字节数（该值会被 wmem_max 覆盖）。

net.ipv4.ip_local_port_range = 1024 65000
#表示用于向外连接的端口范围。缺省情况下很小：32768到61000，改为1024到65000。

net.ipv4.netfilter.ip_conntrack_max=204800
#设置系统对最大跟踪的TCP连接数的限制

net.ipv4.tcp_slow_start_after_idle = 0
#关闭tcp的连接传输的慢启动，即先休止一段时间，再初始化拥塞窗口。

net.ipv4.route.gc_timeout = 100
#路由缓存刷新频率，当一个路由失败后多长时间跳到另一个路由，默认是300。

net.ipv4.tcp_syn_retries = 1
#在内核放弃建立连接之前发送SYN包的数量。

net.ipv4.icmp_echo_ignore_broadcasts = 1
# 避免放大攻击

net.ipv4.icmp_ignore_bogus_error_responses = 1
# 开启恶意icmp错误消息保护

net.inet.udp.checksum=1
#防止不正确的udp包的攻击

net.ipv4.conf.default.accept_source_route = 0
#是否接受含有源路由信息的ip包。参数值为布尔值，1表示接受，0表示不接受。
#在充当网关的linux主机上缺省值为1，在一般的linux主机上缺省值为0。
#从安全性角度出发，建议你关闭该功能
```

## 要了解的知识3： TCP的五元组
所谓一个TCP连接，是包含了client和server双端的，那么如何来区分双端的一个唯一连接标识呢（即 session），这样就引入了一个组合方式的五元组来作为唯一标识， 五元组包括： 源IP地址，源端口，目的IP地址，目的端口，和传输层协议 （因为TCP本身就是讲的TCP协议，很多地方其实用四元组来表达）。

例如：192.168.1.1 12345 TCP 1.2.3.4 80 就构成了一个五元组。其意义是，一个IP地址为192.168.1.1的终端通过端口12345，利用TCP协议，和IP地址为1.2.3.4，端口为80的终端进行连接。

## 接入层的问题和优化方案
简单说下接入层服务是个什么，接入层是我们开发的一个HTTPS的接入平台，它提供高性能、高吞吐、具备GSLB(Global Server Load Balance，即全局负载均衡)、CA证书远程管理以及Waf（web应用防火墙）等很多高可用的feature。
接入层负责承担用户侧的请求，对用户侧来说它是Server端。 接入层把请求通过Proxy方式转给下游的具体服务（http协议，因为https问题在接入层解决掉了），所以对下游服务来说它是Client端。

好了介绍完接入层是什么之后，回归到我们本文的主题上，我们随着接管业务增加后，接入层和下游的连接数量就会增长，初期我们可以做的是把上文提到的如nofile和sysctl里的net内核参数等进行合理调整，但这个基本上还是针对优化Server端的接入能力，对作为Client端来说还是无法突破6w本地端口数的限制。

ok，说到重点地方了， 根据我们提到的TCP的五元组原则，我们如果要突破限制，唯一能做的就是把机器的ip从一变多了。这个可能需要一些网络知识，对一个机器配置多个ip（当然我们内部都是内网ip，没有太多的成本开销），然后配置上route规则，让客户端和服务端直接的路由可以用轮训方式连接，这样一来，根据业务量的规模，去增加ip就行了。但实际操作中，会有很多技术上的细节和困难存在，等我实操和整理好后再来补充吧。







