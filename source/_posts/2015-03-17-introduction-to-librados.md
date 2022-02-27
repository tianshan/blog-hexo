---
title: Ceph的Librados介绍
date: 2015-03-17
tags: ceph
---

Librados是Ceph提供访问存储集群中RADOS（reliable autonomic distributed object store）的接口。
本文主要翻译自[官方文档](http://ceph.com/docs/master/rados/api/librados-intro/)，代码部分只保留了C++版本，其余见原文。

<!--more-->

通过librados可以和存储集群的两类守护进程交互：

* Monitor，负责维护了集群映射的主备份
* OSD， 负责在存储节点存储对象数据

![librados-1](/images/2015-03-17-librados-1.png)

1.安装librados

见原文

2.配置集群句柄

Ceph客户端通过librados和OSD直接交互，存储或者检索数据。要和OSD交互，客户端应用必须调用librados并链接到Monitor。一旦连接后，librados从Monitor中检索集群映射。当客户端应用想要读写数据时，它会创建I/O上下文并绑定到一个pool（存储数据的逻辑分区）。pool有个关联的规则集（应用到特定pool的数据放置策略），定义了在存储集群中如何放置数据。通过I/O上下文，客户端提供对象名给librados，它根据对象名和集群映射（集群的拓扑）计算出数据所在的PG和OSD。然后客户端应用可以读写数据了。客户端应用并不需要直接知道集群的拓扑。

![librados-2](/images/2015-03-17-librados-2.png)

Ceph存储集群句柄处理封装的客户端配置，包括：

* 用户ID用于`rados_create()`，或者用户名用于`rados_create2()`
* _cephx_ 授权密钥
* monitor的ID和IP地址
* 日志级别
* 调试级别

然后，应用使用集群的首要步骤是1）创建集群句柄；2）使用句柄来连接集群。应用要连接集群，必须提供monitor地址，用户名和授权密钥（cephx默认是打开的）。

> 要和不同存储集群连接或者同一个集群的不同用户，需要不同的集群句柄。

RADOS提供很多方式来设置要求的值。对于monitor和加密钥匙设置，一个很容易处理方式是确保Ceph配置文件中包含`keyring`路径以及至少一个monitor地址（例如，`mon host`）。例如：

```
    [global]
    mon host = 192.168.1.1
    keyring = /etc/ceph/ceph.client.admin.keyring
```

一旦创建了句柄，就可以读取Ceph配置文件来配置句柄。你同样可以传参数到你的应用然后用函数（例如 `rados_conf_parse_argv()`）来解析命令行参数，或者解析Ceph环境变量（例如，`rados_conf_parse_env()`）。一些封装可能没有实现方便的方法，所以你可能需要实现这些功能。下图提供了初识化连接的高层流。

![librados-3](/images/2015-03-17-librados-3.png)

一旦连接后，你的应用可以通过集群句柄来调用函数影响整个集群。例如，你可以：
* 获取集群统计
* 使用Pool操作（exists，create，list，delete）
* 获取和设置配置

Ceph的一个有力的特点是绑定到不同pools的能力。每一个pool会有不同数量的PG，对象副本和副本策略。例如，pool可以设置为"hot"来使用SSDs，用于存储频繁访问的对象，或者设置为"cold"，来用纠删码。

大量librados绑定之间的区别在于C和面向对象的C++，Java，Python之间。面向对象的绑定使用对象来代表集群句柄，IO上下文，迭代器，异常等。

C++ 示例

Ceph工程在`ceph/examples/librados`中提供C++示例。对于C++，使用用户`admin`的简单集群句柄需要初始化一个`librados::Rados`集群句柄对象。

``` linenos
#include <iostream>
#include <string>
#include <rados/librados.hpp>

int main(int argc, const char **argv)
{

    int ret = 0;

    /* Declare the cluster handle and required variables. */
    librados::Rados cluster;
    char cluster_name[] = "ceph";
    char user_name[] = "client.admin";
    uint64_t flags;

    /* Initialize the cluster handle with the "ceph" cluster name and "client.admin" user */
    {
        ret = cluster.init2(user_name, cluster_name, flags);
        if (ret < 0) {
                std::cerr << "Couldn't initialize the cluster handle! error " << ret << std::endl;
                ret = EXIT_FAILURE;
                return 1;
        } else {
                std::cout << "Created a cluster handle." << std::endl;
        }
    }

    /* Read a Ceph configuration file to configure the cluster handle. */
    {
        ret = cluster.conf_read_file("/etc/ceph/ceph.conf");
        if (ret < 0) {
                std::cerr << "Couldn't read the Ceph configuration file! error " << ret << std::endl;
                ret = EXIT_FAILURE;
                return 1;
        } else {
                std::cout << "Read the Ceph configuration file." << std::endl;
        }
    }

    /* Read command line arguments */
    {
        ret = cluster.conf_parse_argv(argc, argv);
        if (ret < 0) {
                std::cerr << "Couldn't parse command line options! error " << ret << std::endl;
                ret = EXIT_FAILURE;
                return 1;
        } else {
                std::cout << "Parsed command line options." << std::endl;
        }
    }

    /* Connect to the cluster */
    {
        ret = cluster.connect();
        if (ret < 0) {
                std::cerr << "Couldn't connect to cluster! error " << ret << std::endl;
                ret = EXIT_FAILURE;
                return 1;
        } else {
                std::cout << "Connected to the cluster." << std::endl;
        }
    }
    return 0;
}
```

编译源代码，并用librados链接。例如：

```
g++ -g -c ceph-client.cc -o ceph-client.o
g++ -g ceph-client.o -lrados -o ceph-client
```

3.创建I/O上下文

一旦应用有了集群句柄并连接到了存储集群，你可以创建I/O上下文并开始读写数据。I/O上下文绑定连接到特定的pool。用户必须有特定的CAPS允许来访问特定的pool。例如，有度权限但是美柚写权限的用户就只能读数据。I/O上下文的功能包括：

* 读写数据和扩展属性
* 列出、迭代对象和扩展属性
* 快照pools，列出快照等等

![librados-4](/images/2015-03-17-librados-4.png)

RADOS可以同步或异步的方式交互。一旦你的应用有了I/O上下文，读写操作只需要你知道对象/扩展文件属性的名字。CRUSH算法封装在librados，使用集群映射来确认合适的OSD。OSD守护进程处理副本复制，正如在[Smart Daemons Enable Hyperscale]({{site.baseurl}}/2015/03/08/ceph-architecture.html#smart-daemons-enable-hyperscale)。librados库同时负责映射对象到PG，正如在[Calculating PG IDs]({{site.baseurl}}/2015/03/08/ceph-architecture.html#calculating-pg-ids)。

下面的示例使用默认的`data`pool。然而，你可以使用API来列出所有的pool，确保是存在的，或者创建、删除pools。对于写操作，是】示例列出了如何使用同步模型。对于读操作，示例列举了如何使用异步模型。

> 重要：用API删除pool的警告。如果删除一个pool，pool和上面的全部数据都会丢失。

C++示例

``` linenos
#include <iostream>
#include <string>
#include <rados/librados.hpp>

int main(int argc, const char **argv)
{
    /* Continued from previous C++ example, where cluster handle and
     * connection are established. First declare an I/O Context.
     */

    librados::IoCtx io_ctx;
    const char *pool_name = "data";

    {
        ret = cluster.ioctx_create(pool_name, io_ctx);
        if (ret < 0) {
                std::cerr << "Couldn't set up ioctx! error " << ret << std::endl;
                exit(EXIT_FAILURE);
        } else {
                std::cout << "Created an ioctx for the pool." << std::endl;
        }
    }

    /* Write an object synchronously. */
    {
        librados::bufferlist bl;
        bl.append("Hello World!");
        ret = io_ctx.write_full("hw", bl);
        if (ret < 0) {
                std::cerr << "Couldn't write object! error " << ret << std::endl;
                exit(EXIT_FAILURE);
        } else {
                std::cout << "Wrote new object 'hw' " << std::endl;
        }
    }

    /*
     * Add an xattr to the object.
     */
    {
        librados::bufferlist lang_bl;
        lang_bl.append("en_US");
        ret = io_ctx.setxattr("hw", "lang", lang_bl);
        if (ret < 0) {
                std::cerr << "failed to set xattr version entry! error "
                << ret << std::endl;
                exit(EXIT_FAILURE);
        } else {
                std::cout << "Set the xattr 'lang' on our object!" << std::endl;
        }
    }

    /*
     * Read the object back asynchronously.
     */
    {
        librados::bufferlist read_buf;
        int read_len = 4194304;

        //Create I/O Completion.
        librados::AioCompletion *read_completion = librados::Rados::aio_create_completion();

        //Send read request.
        ret = io_ctx.aio_read("hw", read_completion, &read_buf, read_len, 0);
        if (ret < 0) {
                std::cerr << "Couldn't start read object! error " << ret << std::endl;
                exit(EXIT_FAILURE);
        }

        // Wait for the request to complete, and check that it succeeded.
        read_completion->wait_for_complete();
        ret = read_completion->get_return_value();
        if (ret < 0) {
                std::cerr << "Couldn't read object! error " << ret << std::endl;
                exit(EXIT_FAILURE);
        } else {
                std::cout << "Read object hw asynchronously with contents.\n"
                << read_buf.c_str() << std::endl;
        }
    }

    /*
     * Read the xattr.
     */
    {
        librados::bufferlist lang_res;
        ret = io_ctx.getxattr("hw", "lang", lang_res);
        if (ret < 0) {
                std::cerr << "failed to get xattr version entry! error "
                << ret << std::endl;
                exit(EXIT_FAILURE);
        } else {
                std::cout << "Got the xattr 'lang' from object hw!"
                << lang_res.c_str() << std::endl;
        }
    }

    /*
     * Remove the xattr.
     */
    {
        ret = io_ctx.rmxattr("hw", "lang");
        if (ret < 0) {
                std::cerr << "Failed to remove xattr! error "
                << ret << std::endl;
                exit(EXIT_FAILURE);
        } else {
                std::cout << "Removed the xattr 'lang' from our object!" << std::endl;
        }
    }

    /*
     * Remove the object.
     */
    {
        ret = io_ctx.remove("hw");
        if (ret < 0) {
                std::cerr << "Couldn't remove object! error " << ret << std::endl;
                exit(EXIT_FAILURE);
        } else {
                std::cout << "Removed object 'hw'." << std::endl;
        }
    }
}
```

4.关闭会话

一旦你的应用使用完了I/O上下文和集群句柄，应用应该关闭连接并关闭句柄。对于异步I/O，应用同样需要等待直到异步操作完成。

C++示例

``` linenos
io_ctx.close();
cluster.shutdown();
```
