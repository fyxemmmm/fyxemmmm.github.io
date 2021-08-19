---
title: kafka原理分析
date: 2021-08-17 21:57:35
author: yuxuan
img: 
top: false
hide: false
cover: false
coverImg: 
toc: false
mathjax: false
summary: kafka痛点分析 & page cache 原理
categories: 消息队列
tags:
  - kafka
---

**Kafka痛点分析&核心目标**

当Kafka支撑的实时作业数量很多，单机承载的Topic和Partition数量很大。这种场景下很容易出现的问题是：同一台机器上不同Partition间竞争PageCache资源，相互影响，导致整个Broker的处理延迟上升、吞吐下降。

![Kafka处理读写流程的示意图](https://image.fyxemmmm.cn/blog/images/kafka.png)


**对于Produce请求**：Server端的I/O线程统一将请求中的数据写入到操作系统的PageCache后立即返回，当消息条数到达一定阈值后，Kafka应用本身或操作系统内核会触发强制刷盘操作（如左侧流程图所示）。

**对于Consume请求**：主要利用了操作系统的ZeroCopy机制，当Kafka Broker接收到读数据请求时，会向操作系统发送sendfile系统调用，操作系统接收后，首先试图从PageCache中获取数据（如中间流程图所示）；如果数据不存在，会触发缺页异常中断将数据从磁盘读入到临时缓冲区中（如右侧流程图所示），随后通过DMA操作直接将数据拷贝到网卡缓冲区中等待后续的TCP传输。

综上所述，Kafka对于单一读写请求均拥有很好的吞吐和延迟。处理写请求时，数据写入PageCache后立即返回，数据通过异步方式批量刷入磁盘，既保证了多数写请求都能有较低的延迟，同时批量顺序刷盘对磁盘更加友好。处理读请求时，实时消费的作业可以直接从PageCache读取到数据，请求延迟较小，同时ZeroCopy机制能够减少数据传输过程中用户态与内核态的切换，大幅提升了数据传输的效率。

但当同一个Broker上同时存在多个Consumer时，就可能会由于多个Consumer竞争PageCache资源导致它们同时产生延迟。下面我们以两个Consumer为例详细说明：

![](https://image.fyxemmmm.cn/blog/images/kafka2.png)

如上图所示，Producer将数据发送到Broker，PageCache会缓存这部分数据。当所有Consumer的消费能力充足时，所有的数据都会从PageCache读取，全部Consumer实例的延迟都较低。此时如果其中一个Consumer出现消费延迟（图中的Consumer Process2），根据读请求处理流程可知，此时会触发磁盘读取，在从磁盘读取数据的同时会预读部分数据到PageCache中。当PageCache空间不足时，会按照LRU策略开始淘汰数据，此时延迟消费的Consumer读取到的数据会替换PageCache中实时的缓存数据。后续当实时消费请求到达时，由于PageCache中的数据已被替换掉，会产生预期外的磁盘读取。这样会导致两个后果：

1. **消费能力充足的Consumer消费时会失去PageCache的性能红利。**
2. **多个Consumer相互影响，预期外的磁盘读增多，HDD负载升高。**



**数据分析**

如果Kafka集群TP99流量在170MB/s，TP95流量在100MB/s，TP50流量为50-60MB/s；单机的PageCache平均分配为80GB，取TP99的流量作为参考，在此流量以及PageCache分配情况下，PageCache最大可缓存数据时间跨度为80*1024/170/60 = 8min，可见当前Kafka服务整体对延迟消费作业的容忍性极低。该情况下，一旦部分作业消费延迟，实时消费作业就可能会受到影响。



**痛点分析总结**

总结上述的原理分析以及数据统计，目前Kafka存在如下问题：

1. 实时消费与延迟消费的作业在PageCache层次产生竞争，导致实时消费产生非预期磁盘读。
2. 传统HDD随着读并发升高性能急剧下降。
3. 存在20%的延迟消费作业。

按目前的PageCache空间分配以及集群流量分析，Kafka无法对实时消费作业提供稳定的服务质量保障，该痛点亟待解决。



**预期目标**

根据上述痛点分析，我们的预期目标为保证实时消费作业不会由于PageCache竞争而被延迟消费作业影响，保证Kafka对实时消费作业提供稳定的服务质量保障。

**解决方案**

**可以选择使用SSD**

根据上述原因分析可知，解决目前痛点可从以下两个方向来考虑：

1. 消除实时消费与延迟消费间的PageCache竞争，如：让延迟消费作业读取的数据不回写PageCache，或增大PageCache的分配量等。
2. 在HDD与内存之间加入新的设备，该设备拥有比HDD更好的读写带宽与IOPS。

