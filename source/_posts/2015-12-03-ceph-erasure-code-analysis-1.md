---
title: "Ceph EC的代码分析(一)"
date: 2015-12-03
tags: ceph
---

传统的存储可靠性都是用多副本实现，但是很多大容量备份场景、多媒体内容存储等，多副本的存储成本已经不能忍受，ErasureCode是通用的解决方案，而且这些场景往往是一次写多次读，很适合EC的特点。

ErasureCode可以用1.5副本就实现丢失任意两块都可以恢复出原始数据，但EC也有很明显的缺点，修复数据代价太大，这个以后再讨论。

Ceph的EC目前支持append写，为丰富接口，对接nfs等，最近就在实现EC的overwrite，所以就从EC的角度，分析一下Ceph的PG层逻辑。

<!--more-->

### Placemeng Group

PG全称Placement Group， 用于对象的第一层逻辑聚合，对象到PG的映射就是oid的一次取模，这层映射是不变的，所以当OSD变动时，不会导致大量元数据的变动。

PG层代码，主要有

* `osd/PG` 定义了PG层的接口
* `osd/ReplicatedPG` 副本模式的接口实现

在引入EC后，复用了ReplicagePG部分逻辑，所以引入了PGBackend的新接口，用于实现Replicated和EC不同的逻辑。这样的设计，导致现在的PG层逻辑相当混乱，有些会调用PG接口，有写会调用Replicated接口，阅读代码会很困惑。从Loïc Dachary的[博客](http://dachary.org/?p=2320)可以看到Loïc在设计EC时和Sam对PG的讨论。目前Sam在对PG那块的代码进行重构，未来逻辑应该会清晰。

下面就EC的几个典型操作，分析下这层的逻辑。

### Ceph的EC逻辑

EC写，是client发送数据到主OSD，然后编码数据生成ObjectStore的Transaction，然后发送SubOp到对应的副OSD，都reply后，会回调完成。

EC读，逻辑简单一点，因为读数据需要读取多个副OSD的数据，所以EC只支持异步读。同样，client发送Op到主OSD，主OSD在生成SubReadOp，所有SubReadReply后，回调complete函数，解码并返回给client。

yahoo在前几个提了一个patch，实现了fast_read，其实就是避免了OSD短板，每次发送读请求给所有OSD，然后取先返回的k个OSD数据解码。读需要等所有数据返回后再解码，在长时间使用过程中，HDD会有性能下降，所以读Op速度取决于k个osd中最慢的那个。从[CDS_Jewel](http://tracker.ceph.com/projects/ceph/wiki/Tail_latency_improvements)可以看到，初步实验结果，fast read可以提升30%性能，但集群负载也会提高。


EC read的流程如下，相对比较简单

* 因为ec read需要多个osd的数据，所以只支持异步读。
* EC是条带写的，所以读之前，会把读的范围，通过`ECUtil::stripe_info_t.offset_len_to_stripe_bounds`转换成对应每个osd需要读的范围。

![ec read](/images/2015-12-03-ec-read.png){:width="75%"}


EC相关的分析链接：

1. [EC的读]({% post_url 2015-12-03-ceph-erasure-code-analysis-1 %})
2. [EC删除对象的流程]({% post_url 2016-01-13-ceph-ec-remove-object %})

