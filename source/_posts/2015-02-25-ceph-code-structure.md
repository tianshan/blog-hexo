---
title: "Ceph 代码分析"
date: 2015-02-25
tags: ceph
---

Ceph代码的目录结构。

<!--more-->

* 网络通信
    * `msg` 目录包括了网络传输的代码
    * `message` 目录里定义了传输的消息格式
* 元数据服务器
    * `mds` 目录包括 metadata server 代码
* 数据服务器
    * `os` 目录里包含了 object store 的代码
    * `osd` 目录包括了 object storage device 的代码
* 客户端
    * `osdc` 目录里包括跨网络访问 osd 的代码
    * `librados` 包括了对象存储的客户端操作的代码
    * `librbd, rgw, client`  客户端代码，其代码都是基于 librados 之上
* 监控
    * `mon` 目录里包括了 Ceph Monitor的代码
* 存储映射算法
    * `crush` 目录里包括了 cursh 算法的代码
