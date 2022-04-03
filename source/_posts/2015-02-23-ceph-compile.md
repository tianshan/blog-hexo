---
title: "Ceph 编译"
date: 2015-02-23
tags: ceph
---

初学ceph，记录下编译过程。

<!--more-->

Ceph发展优势：是OpenStack热门的开源存储。
Ceph核心思想
* “算算就好，无需查表”，通过数据ID的简单计算，就可以得到数据的存储位置。
* 充分利用存储节点的计算能力，在服务器端进行数据处理。

[编译](#编译)
---
使用版本Ceph-0.87，系统Ubuntu 14.04，从源码编译安装。

1. `./autogen.sh`

2. `./configure`

* 需要很多依赖包

    sudo apt-get install uuid-dev libblkid-dev libudev-dev libkeyutils-dev libfuse-dev libedit-dev libatomic-ops-dev libsnappy-dev libleveldb-dev libaio-dev xfslibs-dev libboost-dev libboost-thread-dev libcrypto++-dev libcrypto++-doc libcrypto++-utils libgoogle-perftools-dev

* 其中 no tcmalloc found 的手动安装方法。tcmalloc是Google Preftools内的一个组件，可以通过如下方法安装。

    + step 1. linux 64位先安装libunwind，32位不需要

    wget http://download.savannah.gnu.org/releases/libunwind/libunwind-1.1.tar.gz
    tar zxvf libunwind-1.1.tar.gz
    cd libunwind-1.1/
    CFLAGS=-fPIC ./configure --enable-shared
    make CFLAGS=-fPIC
    make CFLAGS=-fPIC install

    + step 2. 安装Google Performance Tools

    wget https://gperftools.googlecode.com/files/gperftools-2.4.tar.gz
    tar zxvf gperftools-2.4.tar.gz
    cd gpehttp://blog.coolceph.com/?p=85rftools-2.4/
    ./configure
    make -j8
    make install
    sudo sh -c 'echo "/usr/local/lib" > /etc/ld.so.conf.d/usr_local_lib.conf'
    /sbin/ldconfig

3. `make`

机器配置不好的话，编译需要时间比较长。可以使用"make -j"增加并发度，8表示同时执行的make方法数。

4. `sudo make install`

    可执行文件会安装在 `/usr/lcoal/bin` ，这里有个坑，通过ceph-deploy安装的可执行文件会在 `/usr/bin` 会有不一样。


[centos](#centos)
------

```
git clone https://github.com/ceph/ceph.git

注意切换到一个release版本

cd ceph

./install-deps.sh

./autogen.sh

./configure

make -j4

sudo make install (可选)
```

>注：需要安装epel源
sudo yum install epel-release && sudo yum update -y

CPU 4核 内存 4G 参数 -j4，编译大概需要20分钟。
