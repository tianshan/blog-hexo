---
title: Ceph的Journal机制
date: 2015-05-29
tags: ceph
---

Ceph底层FileStore模式下，采用了写日志，就是Journal。实现机制类似数据库的写日志。写数据时，会在journal上写日志，保证出现故障时可以从日志恢复。

<!--more-->

Journal源代码中主要涉及两个文件，os目录下的FileJournal和FileStore。FileStore中会有FileJournal的一个实例，调用都在这里发生。


写Journal有两种模式，parallel和writeahead。顾名思义，parallel就是日志和磁盘数据同时写，writeahead是先写日志，只要日志写成功了，就回返回。后台每隔一段时间后，会同步日志中的写操作，实现落盘。这种方法带来的好处就是，可以把很多小IO合并，形成顺序写盘，提高IOPS。

第二种方法中，[官方文档](http://ceph.com/docs/master/rados/configuration/journal-ref/)提到，

* 在速度上，通过预写日志方式，后端文件系统合并小IO，可以提升速度。但实际中，会造成性能有明显抖动，一段较高速率后会有一段低速率，如下图

![iops](/images/2015-05-29-rbd-iops.svg)

* 一致性：OSD的守护进程需要文件系统接口来保证原子复合操作。OSD守护进程会把操作的描述写到日志，然后应用到文件系统。这样保证了对象的原子操作。每隔几秒（`filestore max sync interval`和`filestore min sync interval`)，OSD守护进程会 **停止写操作** 然后同步日志到文件系统，使得守护进程能够删除日志重新利用空间。当失败时，OSD守护进程会从日志上一次同步点开始回滚操作。

注意：这里面提到会停止写操作，从日志队列的表现来看，就是filestore写队列比较满时，日志队列会为空，一般这时候，就是在同步操作。默认的最大同步间隔`filestore max sync interval`为5s。

同步的流程图如下

![iops](/images/2015-05-29-journal.png)

