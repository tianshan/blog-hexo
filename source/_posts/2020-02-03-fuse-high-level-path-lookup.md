---
title: "Fuse high level path lookup 分析"
date: 2020-02-03 19:32:17
tags: fuse
---

Fuse 是一个可以在用户态实现文件系统功能的框架，包括内核模块和用户态库。
基于Fuse开发文件系统主要是实现fuse吐出的一系列文件系统操作函数，对外有两套API，high level和low level。
Low level是基于内部的句柄号来访问的, High level是在low level上实现的，基于Path来访问的API。
本文主要分析high level如何处理path的过程。

<!-- more -->

# Fuse简介

Fast 2017年的论文"To FUSE or Not to FUSE: Performance of User-Space File Systems"很详细的讲解了fuse的实现机制，并且有详细的性能对比，推荐大家阅读。
如图是Fuse的大致框架, 应用调用VFS层的内核文件系统接口，VFS转到fuse的内核模块，fuse内核模块发送的一个fuse的块设备/dev/fuse。Fuse的用户态会不断的查询块设备内的请求，出现请求后，根据请求类型会放到不同的处理队列，然后调用注册的处理函数。

![fuse-architecture](/images/2020-02-03-fuse-architecture.png)

# VFS访问模式

在有目录时，VFS层是逐层来访问文件的。例如create一个文件/path/to/dest，VFS会检查每一层目录的合法性，并最终传递的参数是`<parent_fd, name>`。
也就是当Fuse high level层拿到请求时，只有父目录的fd，以及最后一层目录的名字。而对于High level基于全路径的API模式，就需要High level内部重新组装出完整的path了。

# High level API实现

Fuse的内核态代码在kernel的fs目录下，用户态代码在git仓库 http://github.com/libfuse/libfuse, 其中high level实现主要在`lib/fuse.c`。

## API格式

前文提到了high level API是基于low level实现的，所以high level也会向low level注册函数指针，格式为`fuse_lib_xxx`，create接口的话就是`fuse_lib_create`。而High level对外暴露的注册接口格式为`fuse_fs_xxx`。

## 几个典型参数

* fuse_req_t req: 请求的数据结构，从low level那传递
* fuse_ino_t parent: 父目录的内部ino号
* char \*name: 最后一层的路径
* fuse_file_info \*fi: 输出参数，主要包括fd

## 缓存结构

前面提到了，最终High level只能拿到一个parent fd，那么要回溯整个路径，就需要缓存前面所有的父路径。
High level中很简单了实现了一个Hash桶，来存放所有的fd到name的双向映射。第一层是一个固定大小的数组，hash重复的以双向链表形式组织，新元素插表头。

### 主要函数
* node_table_init: 初始化hash表，固定初始大小为8192
* node_table_resize: 扩大一倍hash表内存
* node_table_reduce: 缩小一倍hash表，不会小于8192

### rehash

hash表增大或缩减都会造成所有元素需要重新计算hash。
这里采用里lazy的方式，首先在hash表使用了一半时开始rehash，在每次计算hash的时候，rehash一个桶，将按照一半size取余的hash迁移到按照当前size取余的桶。从0开始，rehash一半的时候，扩大内存。这样逐步rehash，保证了不会导致某个请求突然延迟增高。

## cache path的路径

前文提到，VFS层会逐层检查目录，到fuse这就是调用的getattr。检查目录会调用lookup函数，一次调用`do_lookup` -> `find_node`, 这里是按照name查找内部ino号，这里会用到name hash表。当name不存在时，调用`alloc_node`来创建`struct node`并插入到name hash表和id hash表。`struct node`中包括这一层的父目录ino，以及当前层的name和自己的ino。

## path的回溯

当High level层收到调用请求`<parent_fd, name>`时，就会根据hash表来还原出完整path，再传递给用户注册的实现函数。
调用路径`get_path_name`/`get_path_common` -> `try_get_path`，根据parent_fd查找到父目录node，拿到父目录name，在根据父目录node中记录的parent_fd回溯再上一层的node，直到根节点，就可以拷贝出完整的path。

# 总结

Fuse high level通过cache整个目录树来恢复完整路径，提供给用户的API实现，方便用户实现无状态的文件系统。
这里其实还有个另外的问题，cache的node什么时候释放。目前的实现中High level层是不会主动释放node的，node有引用计数，但最初创建的一个引用计数一直在。直到上层显示的调用内部接口`forget`才会释放。这里是因为fuse内核态也会cache一部分ino的映射关系，所以这里是做了一个联动，但是在访问了大量新文件后，node占用的内存会显著增加到好几G，可以通过drop_cache来释放，或者调整page cache的设置，来控制内存增长。

## 参考
1.Fast 2017, To FUSE or Not to FUSE: Performance of User-Space File Systems