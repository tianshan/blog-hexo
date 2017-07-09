---
title: Calamari部署
date: 2015-07-25
tags: ceph calamari
---

Calamari是Ceph官方出品的监控平台，监控内容很丰富，可以做一些基本的运维操作，但是还不能做Ceph部署，需要结和ceph-deploy。
Calamari比较复杂，文档也比较少，记录下部署中遇到的坑。。。

<!--more-->

简单介绍下Calamri架构
---

* Calamari 监控平台，使用Apache做服务器
    * Calamari-server： 服务端
    * Calamari-client： 客户端，包括了web端的UI，可以自己定制
    * graphite: 收集数据的存储与展现，提供了接口，可以获取指定间隔的统计量。可以通过服务器地址单独访问 {calamari}/graphite/dashboard。包含在Calamari中
* Salt 服务器基础架构管理平台，具备配置管理、远程执行、监控等功能。
    * Salt-master Salt的服务端
    * Salt-minion Salt的节点端
* diamond 各个节点的数据收集器，收集节点信息发送到服务端。Calamari定制了一个版本。

部署
---

### Ubuntu下的Calamari部署

可以[参考官方](http://calamari.readthedocs.org/en/latest/operations/server_install.html)的介绍。没有尝试过。。

### 主要是Centos的部署

* 从安装包部署 <https://bbs.ceph.org.cn/topic/135>

* 从源码部署 <http://ovirt-china.org/mediawiki/index.php/%E5%AE%89%E8%A3%85%E9%83%A8%E7%BD%B2Ceph_Calamari>

刚部署完，diamond老是起不来，看log，有个包load不到，`PySNMP`，需要pip装下。

> 特别注意，在下面版本下，需要`Salt-2014.1`版本，出现的症状就是，ceph-server能连上，但是Calamari识别不了ceph集群。
>
```
ceph 0.94.2
calamari-1.3
diamond-3.4.67
```
> 被这个问题坑了2天，后来发现社区前几天刚报了个bug <http://tracker.ceph.com/issues/12264> 