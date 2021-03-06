---
title: "redis cluster 的Gossip协议介绍"
date: 2019-05-07T16:04:49+08:00
---

# 概述
本文简单围绕redis的cluster的实现原理进行下介绍，并整理了下知乎上描述交互的关于gossip的一篇文章。


# 了解什么是Gossip协议

## 背景

Gossip protocol 也叫 Epidemic Protocol （流行病协议），实际上它还有很多别名，比如：“流言算法”、“疫情传播算法”等。

这个协议的作用就像其名字表示的意思一样，非常容易理解，它的方式其实在我们日常生活中也很常见，比如电脑病毒的传播，森林大火，细胞扩散等等。

Gossip protocol 最早是在 1987 年发表在 ACM 上的论文 《Epidemic Algorithms for Replicated Database Maintenance》中被提出。主要用在分布式数据库系统中各个副本节点同步数据之用，这种场景的一个最大特点就是组成的网络的节点都是对等节点，是非结构化网络，这区别与之前介绍的用于结构化网络中的 DHT 算法 Kadmelia。

我们知道，很多知名的 P2P 网络或区块链项目，比如 IPFS，Ethereum 等，都使用了 Kadmelia 算法，而大名鼎鼎的 Bitcoin 则是使用了 Gossip 协议来传播交易和区块信息。

实际上，只要仔细分析一下场景就知道，Ethereum 使用 DHT 算法并不是很合理，因为它使用节点保存整个链数据，不像 IPFS 那样分片保存数据，因此 Ethereum 真正适合的协议应该像 Bitcoin 那样，是 Gossip 协议。


**这里先简单介绍一下 Gossip 协议的执行过程**：

Gossip 过程是由种子节点发起，当一个种子节点有状态需要更新到网络中的其他节点时，它会随机的选择周围几个节点散播消息，收到消息的节点也会重复该过程，直至最终网络中所有的节点都收到了消息。这个过程可能需要一定的时间，由于不能保证某个时刻所有节点都收到消息，但是理论上最终所有节点都会收到消息，因此它是一个*最终一致性协议*。



## Gossip 演示

现在，我们通过一个具体的实例来深入体会一下 Gossip 传播的完整过程

为了表述清楚，我们先做一些前提设定

1、Gossip 是周期性的散播消息，把周期限定为 1 秒

2、被感染节点随机选择 k 个邻接节点（fan-out）散播消息，这里把 fan-out 设置为 3，每次最多往 3 个节点散播。

3、每次散播消息都选择尚未发送过的节点进行散播

4、收到消息的节点不再往发送节点散播，比如 A -> B，那么 B 进行散播的时候，不再发给 A。

这里一共有 16 个节点，节点 1 为初始被感染节点，通过 Gossip 过程，最终所有节点都被感染：


![演示图](https://pic4.zhimg.com/v2-575e785e7d03ad317e5bce4e36debb03_b.gif)

## Gossip 的特点（优势）

1）扩展性

网络可以允许节点的任意增加和减少，新增加的节点的状态最终会与其他节点一致。

2）容错

网络中任何节点的宕机和重启都不会影响 Gossip 消息的传播，Gossip 协议具有天然的分布式系统容错特性。

3）去中心化

Gossip 协议不要求任何中心节点，所有节点都可以是对等的，任何一个节点无需知道整个网络状况，只要网络是连通的，任意一个节点就可以把消息散播到全网。

4）一致性收敛

Gossip 协议中的消息会以一传十、十传百一样的指数级速度在网络中快速传播，因此系统状态的不一致可以在很快的时间内收敛到一致。消息传播速度达到了 logN。

5）简单

Gossip 协议的过程极其简单，实现起来几乎没有太多复杂性。

Márk Jelasity 在它的 《Gossip》一书中对其进行了归纳：

![Gossip实现伪代码](https://pic4.zhimg.com/80/v2-c08ff37e40fd993475ee79919531bbe3_hd.jpg)


## Gossip 的缺陷

分布式网络中，没有一种完美的解决方案，Gossip 协议跟其他协议一样，也有一些不可避免的缺陷，主要是两个：

1）消息的延迟

由于 Gossip 协议中，节点只会随机向少数几个节点发送消息，消息最终是通过多个轮次的散播而到达全网的，因此使用 Gossip 协议会造成不可避免的消息延迟。不适合用在对实时性要求较高的场景下。

2）消息冗余

Gossip 协议规定，节点会定期随机选择周围节点发送消息，而收到消息的节点也会重复该步骤，因此就不可避免的存在消息重复发送给同一节点的情况，造成了消息的冗余，同时也增加了收到消息的节点的处理压力。而且，由于是定期发送，因此，即使收到了消息的节点还会反复收到重复消息，加重了消息的冗余。

## Gossip 类型

Gossip 有两种类型：

Anti-Entropy（反熵）：以固定的概率传播所有的数据
Rumor-Mongering（谣言传播）：仅传播新到达的数据
Anti-Entropy 是 SI model，节点只有两种状态，Suspective 和 Infective，叫做 simple epidemics。

Rumor-Mongering 是 SIR model，节点有三种状态，Suspective，Infective 和 Removed，叫做 complex epidemics。

其实，Anti-entropy 反熵是一个很奇怪的名词，之所以定义成这样，Jelasity 进行了解释，因为 entropy 是指混乱程度（disorder），而在这种模式下可以消除不同节点中数据的 disorder，因此 Anti-entropy 就是 anti-disorder。换句话说，它可以提高系统中节点之间的 similarity。

在 SI model 下，一个节点会把所有的数据都跟其他节点共享，以便消除节点之间数据的任何不一致，它可以保证最终、完全的一致。

由于在 SI model 下消息会不断反复的交换，因此消息数量是非常庞大的，无限制的（unbounded），这对一个系统来说是一个巨大的开销。

但是在 Rumor Mongering（SIR Model） 模型下，消息可以发送得更频繁，因为消息只包含最新 update，体积更小。而且，一个 Rumor 消息在某个时间点之后会被标记为 removed，并且不再被传播，因此，SIR model 下，系统有一定的概率会不一致。

而由于，SIR Model 下某个时间点之后消息不再传播，因此消息是有限的，系统开销小。



## Gossip 中的通信模式

在 Gossip 协议下，网络中两个节点之间有三种通信方式:

- Push: 节点 A 将数据 (key,value,version) 及对应的版本号推送给 B 节点，B 节点更新 A 中比自己新的数据
- Pull：A 仅将数据 key, version 推送给 B，B 将本地比 A 新的数据（Key, value, version）推送给 A，A 更新本地
- Push/Pull：与 Pull 类似，只是多了一步，A 再将本地比 B 新的数据推送给 B，B 则更新本地
如果把两个节点数据同步一次定义为一个周期，则在一个周期内，Push 需通信 1 次，Pull 需 2 次，Push/Pull 则需 3 次。虽然消息数增加了，但从效果上来讲，Push/Pull 最好，理论上一个周期内可以使两个节点完全一致。直观上，Push/Pull 的收敛速度也是最快的。

## 复杂度分析

对于一个节点数为 N 的网络来说，假设每个 Gossip 周期，新感染的节点都能再感染至少一个新节点，那么 Gossip 协议退化成一个二叉树查找，经过 LogN 个周期之后，感染全网，时间开销是 O(LogN)。由于每个周期，每个节点都会至少发出一次消息，因此，消息复杂度（消息数量 = N * N）是 O(N^2) 。注意，这是 Gossip 理论上最优的收敛速度，但是在实际情况中，最优的收敛速度是很难达到的。

假设某个节点在第 i 个周期被感染的概率为 pi，第 i+1 个周期被感染的概率为 pi+1 ，

1）则 Pull 的方式:

![pull](https://pic3.zhimg.com/80/v2-df4e038395c36b430a55a53dbc0b7b5e_hd.jpg)

2）Push 方式：

![push](https://pic3.zhimg.com/80/v2-b237352666764df3e3e0f432f4bdcc52_hd.jpg)



显然 Pull 的收敛速度大于 Push ，而每个节点在每个周期被感染的概率都是固定的 p (0<p<1)，因此 Gossip 算法是基于 p 的平方收敛，也称为概率收敛，这在众多的一致性算法中是非常独特的。

## 一个Golang版本的实现

[https://github.com/hashicorp/memberlist](https://github.com/hashicorp/memberlist)



# Redis cluster gossip的机制

社区版redis cluster是一个P2P无中心节点的集群架构，依靠gossip协议传播协同自动化修复集群的状态。本文将深入redis cluster gossip协议的细节，剖析redis cluster gossip协议机制如何运转。

## 协议解析

cluster gossip协议定义在在ClusterMsg这个结构中，源码如下：

```c
typedef struct {
    char sig[4];        /* Signature "RCmb" (Redis Cluster message bus). */
    uint32_t totlen;    /* Total length of this message */
    uint16_t ver;       /* Protocol version, currently set to 1. */
    uint16_t port;      /* TCP base port number. */
    uint16_t type;      /* Message type */     
    uint16_t count;     /* Only used for some kind of messages. */
    uint64_t currentEpoch;  /* The epoch accordingly to the sending node. */
    uint64_t configEpoch;   /* The config epoch if it's a master, or the last
                               epoch advertised by its master if it is a
                               slave. */
    uint64_t offset;    /* Master replication offset if node is a master or
                           processed replication offset if node is a slave. */
    char sender[CLUSTER_NAMELEN]; /* Name of the sender node */
    unsigned char myslots[CLUSTER_SLOTS/8];
    char slaveof[CLUSTER_NAMELEN];
    char myip[NET_IP_STR_LEN];    /* Sender IP, if not all zeroed. */
    char notused1[34];  /* 34 bytes reserved for future usage. */
    uint16_t cport;      /* Sender TCP cluster bus port */
    uint16_t flags;      /* Sender node flags */
    unsigned char state; /* Cluster state from the POV of the sender */
    unsigned char mflags[3]; /* Message flags: CLUSTERMSG_FLAG[012]_... */
    union clusterMsgData data;
} clusterMsg;
```

可以对此结构将消息分为三部分：

1、sender的基本信息

2、集群视图的基本信息

3、具体的消息，对应clsuterMsgData结构中的数据

## 运转机制
通过gossip协议，cluster可以提供集群间状态同步更新、选举自助failover等重要的集群功能。

### 握手联结

客户端给节点X发送cluster meet 节点Y的请求后，节点X之后就会尝试主从和节点Y建立连接。此时在节点X中保存节点Y的状态是：

CLUSTER_NODE_HANDSHAKE：表示节点Y正处于握手状态，只有收到来自节点Y的ping、pong、meet其中一种消息后该状态才会被清除

CLUSTER_NODE_MEET：表示还未给节点Y发送meet消息，一旦发送该状态清除，不管是否成功

 以下是meet过程：
 ```
（0）节点X通过getRandomHexChars这个函数给节点Y随机生成nodename
（1）节点X 在clusterCron运转时会从cluster->nodes列表中获取未建立tcp连接，如未发送过meet，发送CLUSTERMSG_TYPE_MEET，节点Y收到meet消息后：
（2）查看节点X还未建立握手成功，比较sender发送过来的消息，更新本地关于节点X的信息
（3）查看节点X在nodes不存在，添加X进nodes，随机给X取nodename。状态设置为CLUSTER_NODE_HANDSHAKE
（4）进入gossip处理这个gossip消息携带的集群其他节点的信息，给集群其他节点建立握手。
（5）给节点X发送CLUSTERMSG_TYPE_PONG，节点Y处理结束（注意此时节点Y的clusterReadHandler函数link->node为NULL）。
（6）节点X收到pong后，发现和节点Y正处在握手阶段，更新节点Y的地址和nodename，清除CLUSTER_NODE_HANDSHAKE状态。
（7）节点X在cron()函数中将给未建立连接的节点Y发送ping
（8）节点Y收到ping后给节点X发送pong
（9）节点X将保存的节点Y的状态CLUSTER_NODE_HANDSHAKE清除，更新一下nodename和地址，至此握手完成，两个节点都保存相同的nodename和信息。
```
![img](https://yqfile.alicdn.com/68692e492336e8052a5a0e87e20460c59959caca.png)
 
看完整个握手过程后，我们尝试思考两个问题：

1、如果发送meet失败后，节点X的状态CLUSTER_NODE_MEET状态又被清除了，cluster会如何处理呢？

这时候节点Y在下一个clusterCron()函数中会直接给节点Y发送ping，但是不会将节点X存入cluster->nodes，导致节点X认为已经建立连接，然而节点Y并没有承认。在后面节点传播中，如果有其他节点持有节点X的信息并给节点Y发送ping，也会触发节点Y主动再去给节点X发送meet建立连接。

2、如果节点Y已经有存储节点X，但还是收到了节点X的meet请求，如何处理？

#### nodename相同：
```
（1）节点Y发送pong给节点X
（2）如果正处于握手节点，会直接删除节点，这里会导致节点Y丢失了节点X的消息。相当于问题1。
（3）非握手阶段往下走正常的ping流程
```

#### nodename不同：
```
（1）节点Y重新创建一个随机nodename放入nodes中并设置为握手阶段，此时有两个nodename存在。
（2）节点Y发送pong给节点X
（3）节点Y如果已经创建过和节点X的连接，节点Y会在本地更新节点X的nodename，删除第一个nodename存储的node，更新握手状态，此时只剩下第二个正确的nodename。
（4）节点Y如果没创建过和节点X的链接，会在clustercron(）中再次给节点X发送ping请求，两个nodename会先后各发送一次。
（5）第一个nodename发送ping后，在收到节点X回复的pong中，更新节点X的nodename
（6）第二个nodename发送ping后，在收到节点X回复的pong中，发送节点X的nodename已经存在，第二个nodename处于握手状态，这时候直接删除了第二个nodename。
```

结论：只有nodename相同并且两个节点都在握手阶段，会导致其中一个节点丢掉另外一个节点。
 
### 健康检测及failover

详情见文章：https://yq.aliyun.com/articles/638627?utm_content=m_1000016044

### 状态更新及冲突解决

假如出现两个master的时候gossip协议是如何处理冲突的呢？

首先要理解两个重要的变量：

configEpoch： 每个分片有唯一的epoch值，主备epoch应该一致

currentEpoch：集群当前的epoch，=集群中最大分片的epoch

在ping包中会自带sender节点的slots信息和currentEpoch, configEpoch。

master节点收到来自slave节点后的处理流程：

（1）receiver比较sender的角色，

如果sender认为自己是master，但是在receiver被标记为slave，则receiver节点在集群视图中将sender标记为master。

如果sender认为自己是slave，但是在receiver被标记为master, 则在receiver的集群视图中将sender标记为slave, 加入到sender标记的master中，并且删除sender在reciver集群视图中的slots信息。

（2）比较sender自带的slot信息和receiver集群视图中的slots是否冲突，有冲突则进行下一步比较

（3）比较sender的configEpoch 是否 > receiver集群视图中的slots拥有者的configepoch，如是在clusterUpdateSlotsConfigWith函数中重新设置slots拥有者为sender，并且将旧slots拥有者设置为sender的slave，再比较本节点是有脏slot, 有则清除掉。

（4）比较sender自身的slots信息 < receiver集群视图中的slots拥有者的configepoch，发送update信息，通知sender更新，sender节点也会执行clusterUpdateSlotsConfigWith函数。

![img](https://yqfile.alicdn.com/8abfc607e1649060b2e14dabec47a6ecd57c791b.png)
 
 
如果两个节点的configEpoch, currentEpoch，角色都是master， 这时候如何处理呢？

receiver的currentEpoch自增并且赋值给configEpoch，也就是强制自增来解决冲突。这时候因为configEpoch大，又可以走回上文的流程。

所以可能存在双master同时存在的情况，但是最终会挑选出新的master。

# 参考文献

- [https://zhuanlan.zhihu.com/p/41228196](https://zhuanlan.zhihu.com/p/41228196)
- [https://yq.aliyun.com/articles/680237](https://yq.aliyun.com/articles/680237)