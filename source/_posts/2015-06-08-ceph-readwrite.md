---
title: Ceph读写流程分析
date: 2015-06-08
tags: ceph
---

初步的Ceph读写流程分析。更详细的IO路径整理好了放上来。

<!--more-->

Ceph OSD层的数据层级

1.OSD 
    
主要实现 OSD,OSDService ，每个数据节点的守护进程

2.PG
    
主要实现 PG,ReplicatedPG,ReplicatedBackend，Object的逻辑组织

3.ObjectStore

主要实现 FileStore,KeyValueStore,MemStore，直接操作数据


OSD层和PG层的读写
---

OSD和ObjectStore层，都有一个线程池（tp）和消息队列（wq），每个线程会从不同消息队列中取出消息然后执行。

* 下图是OSD的消息入队路径，红色为主要路径

![OSD-in](/images/2015-06-08-osd-in.jpg)

* 消息入队后，线程池OSDService->op_tp（osd->osd_op_tp）从队列中拿出消息开始工作。详细流程见下图

![OSD-out](/images/2015-06-08-osd-out.jpg)

图中，红色方框都是主OSD的操作，蓝色方框是从OSD的操作。

首先会有主OSD消息，经过一定检查后，执行到右半部分，提交事务前，先将从OSD的op添加到Apply和Commit的等待队列，然后发消息给从PG的OSD，最后提交自己的操作到ObjectStore。

从OSD收到消息后，会注册Apply和Commit事件（在完成事务后执行），然后提交事务到ObjectStore。
