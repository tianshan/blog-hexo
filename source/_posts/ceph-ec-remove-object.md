---
title: Ceph EC代码分析(二)
date: 2016-01-13
tags: ceph
---

Ceph EC常用的编码RS(k, m)，k块数据块，编码为m块校验块，可以容忍任意m块丢失。所以，为保证数据一致性，EC的写需要至少k块完成，才算写成功。
Ceph为保证这样的一致性，引入了rollback机制，任意操作都是可回滚的，保证在出错时，能够恢复成上一个完整的版本。

本篇就对象删除的流程简单介绍下rollback机制，相对比较好理解。

<!--more-->

rollback常见于数据库中，是对之前操作的撤销。Ceph中同样实现rollback机制，在EC对象的修改出错后，rollback对应的操作。

在PGLog中，每条pg_log_entry_t中有这样一个对象`ObjectModDesc`，顾名思义记录了对象修改。
代码在`src/osd/osd_types.h`中，支持append, setattrs, rmobject, create, update_snaps几种操作的回滚。而对于副本对象或者不支持的一些操作，会直接`mark_unrollbackable`。

每种操作记录如下

* append：记录old_size，利用truncate回滚  
* setattrs：记录old_attrs，直接覆盖回滚  
* rmobject：记录deletion_version，删除的对象会先重命名为带版本的对象，回滚时再重命名回去，下面会详细讨论。  
* create：直接删除就是回滚了。  
* update_snaps：记录old_snaps，这个暂时不讨论。  

`ObjectModDesc`中还有一个`Visitor`类，访问者模式，定义了回滚不同操作执行的接口。回滚类的实现，就定义在`PGBackend::RollbackVisitor`，对每种操作，都会调用PGBackend的接口，来生成对应的回滚事务。下面主要讨论删除操作。

首先，删除对应OP类型为CEPH_OSD_OP_DELETE，在`ReplicatedPG::do_osd_ops`中，然后调用`ReplicatedPG::_delete_oid`，其中通过`pool.info.require_rollback()`来判断是否需要回滚，也就是否是EC pool，回滚模式下，会调用stash，把对象重命名为当前版本的对象。

这里要提一下什么是带版本的对象，原本保存对象id的结构是hobject_t(代码在src/common/hojbect.h)，在引入EC后，同一个对象的不同OSD存储的数据是不一样的，但还是PG层复用的ReplicatedPG(这里不得不吐槽下混乱的PG层逻辑)，所以需要一个新的结构shard_id来区分不同的OSD，通过为了支持回滚操作，引入了对象版本的概念，所以增加了一个新的结构ghobject_t，来保存shard_id和version参数，然后还保存了一个hobject_t参数。然后在对象索引时，用的是hobject_t，在每个osd具体存储数据时，用的ghojbect_t。

下面看下被删除对象的两个去处。

### 需要回滚的时候
在集群状态发生变化的时候，pg会进入peering状态，然后发现pglog有不同时会merge日志，具体的函数为`PGLog::_merge_object_divergent_entries`。其中，在发现自己比权威日志多出一部分时，就会把多出的日志添加到`PG::PGLogEntryHandler`对象的`to_rollback`列表中。然后在下次记录日志调用`PG::append_log`时（比如下一次写操作），在`handler.apply(this, &t)`函数中生成回滚事务。

### 对象真正的删除

![object-remove](/images/2015-01-13-ceph-ec-remove-object.png)

上图描述了一个对象执行删除操作之后，真正删除时的流程。
首先，对象真正的删除，是在删除对象的这条pglog要被剔除的时候。pglog保存的条数有两个参数配置，第一个`osd_min_pg_log_entries`，pglog最少保存的条目数，默认3000；第二个`osd_max_pg_log_entries`，pglog最多保存的条目数，默认10000。两个的区别是，在pg health情况下，pglog超过min条数，就会判断是否要删除，在pg降级或者处于恢复状态时，会从max开始判断。判断pglog的剔除，是在`ReplicatedPG::calc_trim_to`，小于`min_last_complete_ondisk`(表示还没有落盘的最小版本)的pglog都是可以trim的。

在每次提交事务前，都会调用calc_trim_to函数，判断是否要剔除，更新pg_trim_to，表示直到该版本的pglog都要执行trim。
第二步在记录新log时，从上一次trim过的pglog开始往后扫描，直到pg_trim_to对应的版本，添加到`PG::PGLogEntryHandler`中的`to_trim`列表。
第三部，调用真正的删除接口，`PGbackend::trim_stashed_object`生成remove事务，包括在正常Op事务中，发送给ObjectStore，至此对象才会真正删除。
以上是主OSD的流程。在主OSD给副OSD发送SubOp时，会把最新的pg_trim_to发送过去，副OSD收到后，会执行以上的2，3步。


EC相关的分析链接：

1. [EC的读]({% post_url 2015-12-03-ceph-erasure-code-analysis-1 %})
2. [EC删除对象的流程]({% post_url 2016-01-13-ceph-ec-remove-object %})
