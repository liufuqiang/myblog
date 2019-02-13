---
title: "聊聊常用操作系统的分类"
tags: ["操作系统", "os"]
date: 2019-02-13T11:17:23+08:00
---

我们在选择下载软件的时候，经常会遇到选择操作系统的地方，比如darwin，linux，freebsd等，那么今天介绍下当前市面上常用到的操作系统的分类情况吧。

![类似场景](/images/promtheus_download.png)

## linux

[Linux](https://zh.wikipedia.org/wiki/Linux) 是一种自由和开放源代码的类 UNIX 操作系统。该操作系统的内核由林纳斯·托瓦兹在 1991 年 10 月 5 日首次发布，在加上用户空间的应用程序之后，成为 Linux 操作系统。Linux 也是自由软件和开放源代码软件发展中最著名的例子。只要遵循 GNU 通用公共许可证（GPL），任何个人和机构都可以自由地使用 Linux 的所有底层源代码，也可以自由地修改和再发布。大多数 Linux 系统还包括像提供 GUI 的 X Window 之类的程序。除了一部分专家之外，大多数人都是直接使用 Linux 发行版，而不是自己选择每一样组件或自行设置。

### linux 发行版

Linux的发行版本可以大体分为两类，一类是商业公司维护的发行版本，一类是社区组织维护的发行版本，前者以著名的Redhat(RHEL)为代表，后者以Debian为代表。

![Linux发行版](http://s5.51cto.com/wyfs02/M02/9D/E6/wKiom1mINIzA_dJqAAA-WlpJkQk751.jpg-wh_651x-s_2046512469.jpg)

Redhat，应该称为Redhat系列，包括RHEL(Redhat Enterprise Linux，也就是所谓的Redhat Advance Server，收费版本)、Fedora Core(由原来的Redhat桌面版本发展而来，免费版本)、CentOS(RHEL的社区克隆版本，免费)。Redhat应该说是在国内使用人群最多 的Linux版本，甚至有人将Redhat等同于Linux，而有些老鸟更是只用这一个版本的Linux。所以这个版本的特点就是使用人群数量大，资料非 常多，言下之意就是如果你有什么不明白的地方，很容易找到人来问，而且网上的一般Linux教程都是以Redhat为例来讲解的。Redhat系列的包管 理方式采用的是基于RPM包的YUM包管理方式，包分发方式是编译好的二进制文件。稳定性方面RHEL和CentOS的稳定性非常好，适合于服务器使用， 但是Fedora Core的稳定性较差，最好只用于桌面应用。

Debian，或者称Debian系列，包括Debian和Ubuntu等。Debian是社区类Linux的典范，是迄今为止最遵循GNU规范 的Linux系统。Debian最早由Ian Murdock于1993年创建，分为三个版本分支(branch)： stable, testing 和 unstable。其中，unstable为最新的测试版本，其中包括最新的软件包，但是也有相对较多的bug，适合桌面用户。testing的版本都经 过unstable中的测试，相对较为稳定，也支持了不少新技术(比如SMP等)。而stable一般只用于服务器，上面的软件包大部分都比较过时，但是 稳定和安全性都非常的高。Debian最具特色的是apt-get / dpkg包管理方式，其实Redhat的YUM也是在模仿Debian的APT方式，但在二进制文件发行方式中，APT应该是最好的了。Debian的资 料也很丰富，有很多支持的社区，有问题求教也有地方可去:)


Ubuntu严格来说不能算一个独立的发行版本，Ubuntu是基于Debian的unstable版本加强而来，可以这么说，Ubuntu就是 一个拥有Debian所有的优点，以及自己所加强的优点的近乎完美的 Linux桌面系统。根据选择的桌面系统不同，有三个版本可供选择，基于Gnome的Ubuntu，基于KDE的Kubuntu以及基于Xfc的 Xubuntu。特点是界面非常友好，容易上手，对硬件的支持非常全面，是最适合做桌面系统的Linux发行版本。



## darwin

[Darwin](https://zh.wikipedia.org/wiki/Darwin_(%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F))  是由苹果公司于2000年所发布的一个开放源代码操作系统。Darwin是macOS和iOS操作环境的操作系统部分。苹果公司于2000年把Darwin发布给开放源代码社群。

Darwin是一种类Unix操作系统，包含开放源代码的XNU内核，其以微核心为基础的核心架构来实现Mach，而操作系统的服务和用户空间工具则以BSD为基础。类似其他类Unix操作系统，Darwin也有对称多处理器的优点，高性能的网络设施和支持多种集成的文件系统。


## freebsd netbsd  openbsd

[FreeBSD](https://zh.wikipedia.org/wiki/FreeBSD)是一个类Unix的操作系统，也是FreeBSD项目的发展成果。FreeBSD是第一个开放源代码的系统，他是由基于BSD Unix的源代码派生而来的。BSD Unix是加州大学伯克利分校在1975年至1993年开发的操作系统。FreeBSD被开发为自由软件，这意味着其源代码开放，人人都可以使用FreeBSD。任何人都可以获得并使用它来满足各种需求，也可以修改它，然后再重发布它。此功能专为个人和公司量身定制，可用于创建各种基于FreeBSD的商业和非商业产品。尽管FreeBSD直接从BSD派生，但是从法律的角度来看，FreeBSD并不是“UNIX”。因为现在“UNIX”商标是属于国际开放标准组织的。FreeBSD的第一个版本于1993年发布。

FreeBSD与Linux的用户群有相当一部分是重 合的，二者支持的硬件环境也比较一致，所采用的软件也比较类似，所以可以将FreeBSD视为一个Linux版本来比较。

FreeBSD拥有两个分支： stable和current。顾名思义，stable是稳定版，而 current则是添加了新技术的测试版。FreeBSD采用Ports包管理系统，与Gentoo类似，基于源代码分发，必须在本地机器编后后才能运 行，但是Ports系统没有Portage系统使用简便，使用起来稍微复杂一些。FreeBSD的最大特点就是稳定和高效，是作为服务器操作系统的最佳选 择，但对硬件的支持没有Linux完备，所以并不适合作为桌面系统。


[NetBSD](https://zh.wikipedia.org/wiki/NetBSD)是一份自由、安全的具有高度可定制性的类Unix操作系统，适于多种平台，从64位AMD Athlon服务器和桌面系统到手持设备和嵌入式设备。它设计简洁，代码规范，拥有众多先进特性，使得它在业界和学术界广受好评，用户可以通过完整的源代码获得支持。许多程序都可以很容易地通过NetBSD Packages Collection获得。

[OpenBSD](https://zh.wikipedia.org/wiki/OpenBSD)是一个类Unix计算机操作系统，是加州大学伯克利分校所开发的Unix派生系统伯克利软件套件（BSD）的一个后继者。它是在1995年尾由荷裔加拿大籍项目领导者西奥·德·若特（Theo de Raadt）从NetBSD分支而出。除了操作系统，OpenBSD项目已为众多子系统编写了可移植版本，其中最值得注意的是PF（Packet Filter）、OpenSSH和OpenNTPD，作为软件包，它们在其他操作系统中随处可见。

### 多个BSD系统的差异点

FreeBSD侧重于稳定，NetBSD侧重于多平台，OpenBSD侧重于安全。



## windows

Microsoft Windows（中译：视窗操作系统）是微软公司推出的一系列操作系统。它问世于1985年，起初是MS-DOS之下的桌面环境，其后续版本逐渐发展成为主要为个人计算机和服务器用户设计的操作系统，并最终获得了世界个人计算机操作系统的垄断地位。此操作系统可以在几种不同类型的平台上运行，如个人计算机（PC）、移动设备、服务器（Server）和嵌入式系统等等，其中在个人计算机的领域应用内最为普遍。在2004年国际数据信息公司一次有关未来发展趋势的会议上，副董事长Avneesh Saxena宣布Windows拥有终端操作系统大约70％的市场份额。

Windows操作系统当前最新的稳定版是于2015年7月29日发布的 Windows 10。Windows Server当前最新的稳定版是2018年10月2日发布的Windows Server 2019。


## 参考文献
- http://os.51cto.com/art/201708/547373.htm 
