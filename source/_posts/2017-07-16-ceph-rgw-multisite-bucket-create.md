---
title: "ceph rgw multisite 桶创建"
date: 2017-07-16 23:54:05
tags: rgw
---

本篇是介绍ceph multisite第二篇，从多zone同步的 `桶创建` 看元数据的同步过程。zonegroup下多zone元数据是强同步，所有从zone的元数据修改会重定向到主zone，主zone的元数据修改会通知给所有从zone。而数据部分是多zone可以同时写入，所有zone之间会相互同步，实现最终数据一致。

<!-- more -->

### 测试

源码中，`test`目录有脚本可以快速搭建多个集群，方便本地测试multisite。  
在src目录运行如下命令，会自动创建zonegroup为zg1，zone分别为zg1-1，zg-2的集群。

```bash
# 启动集群
./test/rgw/test-rgw-multisite.sh <num-clusters>

# 运行每个集群单独的命令
./mrun c<序号> cmd
# 例如
./mrun c1 ceph -s

# 停止集群
./mstop.sh c<序号>
```

### 从zone（zg1-2）创建桶

s3协议通过PUT op创建桶，执行到 `RGWCreateBucket::execute()`，通过 `store->is_meta_master()` 读取period信息，判断当前zone是否为主zone，如果不是主zone，调用 `forward_request_to_master`，通过system用户将请求先转发到主zone，如果主zone创建失败，则直接返回错误码。如果主zone创建成功，从zone然后继续调用 `store->create_bucket`，


```bash
# 转发请求格式
ip:port/<bucket>/?rgwx-uid=zone.user&rgwx-zonegroup=a79c9c44-6422-46a7-b778-74ba9ee47c6d
```

### 主zone（zg1-1）创建

正常的创桶OP，`RGWCreateBucket`和 具体函数 `RGWStore::create_bucket`

### rule的选择

一个关键的问题，创建桶需要placement-rule，目前策略是，
* 创桶时没有指定rule，那两遍都使用默认的rule
* 如果指定了rule，那两边必须有同名rule，否则创建失败


### 主zone同步

主zone创建过程中`RGWStore::create_bucket`，在初始化`bucket_index`（调用`init_bucket_index`）之后，调用`put_linked_bucket_info`创建bucket元数据，然后调用`put_bucket_entrypoint_info`在用户的桶元数据中插入omap记录，两个接口都会调用`store->meta_mgr->put_entry`，所有元数据操作都会通过这个接口来持久化。

元数据操作`RGWMetadataManager::put_entry`中持久化数据，zone的log_meta打开时（即主zone）会记录meta_log。由于meta_log和元数据的对象是两个对象，为保证完成一致性采用了日志的两步提交。在元数据持久化前，先调用`pre_modify`预写入meta_log，然后调用`rgw_put_system_obj`写入修改的元数据，完成后调用`post_modify`完成日志提交，保证元数据和日志一起完成。

meta_oid 中的日志，会通过meta_notify定期通知所有的从zone，让从zone来拉取最新的修改，并同步到各自的本地。


![create bucket](/images/2017-07-06-create_bucket.png)

