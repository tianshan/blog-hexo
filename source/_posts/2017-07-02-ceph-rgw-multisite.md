---
title: ceph-rgw-multisite
date: 2017-07-02 22:53:33
tags: rgw
---

工作一年，基本一直在做rgw，本篇作为续写博客第一篇，从最近看的multisite写起，争取在架构和代码上给rgw写一个系列。

ceph从J版开始，重构了multisite方案，改变了原来配置复杂，同步难于管理的问题。新版简化了配置，zone之间通过内部的协程框架同步。本篇先简单介绍同步流程。

<!-- more -->

### 同步日志

datalog，记录了每一个op的修改，记录于log池，omap形式，每个entry记录了全局单调递增的id，作为marker

### 日志入口

日志由RGWDataChangesLog类管理，在Put Op的索引complete的时候，调用add_entry

### 同步形式

* 每个zone会向zonemap中的其他zong拉取datalog，本地会记录sync_status，每次从上次成功同步的marker开始拉取
* 去重日志，保留timestamp最新的修改成为squash_map
* 遍历squash_map，data_sync_module的sync_object
* 默认的数据sync_module会调用fetch_remote_obj，其中通过timestamp判断是否要同步目标zone的数据

### log trim

随着数据修改，datalog会不断变大，每隔一段时间，日志会询问所有zone，计算得最小的同步marker，把该marker之前的日志trim掉

### data sync module

社区扩展了同步的功能，将最终的同步接口抽象了出来。目前包括默认的数据同步，ElasticSearch同步元数据，Log类，用于打印日志测试。
每个zone可以配置tier_type表示同步类型，这个zone下的网关就会启动对应的同步功能。
这里主要是ES同步元数据后，可以实现元数据检索。


每块的代码和详细分析后面再介绍。
