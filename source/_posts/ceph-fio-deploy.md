---
title: ceph性能测试
date: 2015-05-10
tags: ceph
---

研究一段时间后，开始测试Ceph性能。
ceph与fio的结合见[Ceph Performance Analysis: fio and RBD](https://telekomcloud.github.io/ceph/2014/02/26/ceph-performance-analysis_fio_rbd.html)，官方给出了一个样例。

<!--more-->

fio的安装
---
centos下，从源码编译安装

```
wget http://brick.kernel.dk/snaps/fio-2.2.8.tar.gz
tar -zxvf fio-2.2.8.tar.gz
#安装依赖包,librbd用于连接ceph测试
sudo yum install -y libaio-devel zlib-devel librbd1-devel

#其中，librbd1-devel提供fio rbd测试接口，安装需要配置ceph的源
sudo vim /etc/yum.repos.d/ceph.repo
#粘贴如下内容，其中hammer是当前release版本，el7表示centos7，下载源用的欧洲的，注意把gpgcheck置0，因为经常连不上
[ceph-noarch]
name=Ceph noarch packages
baseurl=http://eu.ceph.com/rpm-hammer/el7/noarch
enabled=1
gpgcheck=0
type=rpm-md
gpgkey=https://ceph.com/git/?p=ceph.git;a=blob_plain;f=keys/release.asc
#添加完后执行，sudo yum update

cd fio-2.2.8
#安装librbd后可以看到`Rados Block Device engine`选项为yes
./configure
make
sudo make install
```


可能用到的一些命令
---

查看挂载  

```
df -hT
```

查看io状态,每隔2s查看sda的扩展统计，共6次  

```
iostat -x sda 2 6
```

```bash
#参数含义
avgrq-sz    #发送到设备的请求的平均大小,单位是扇区
avgqu-sz    #发送到设备的请求的平均队列长度
```

查看硬盘SATA信息   

```
dmesg | grep ata
```

扫描物理卷

```
sudo lvmdiskscan
```

显示所有逻辑卷

```
sudo pvs
```

删除物理卷

```
sudo pvremove /dev/sdb
```

删除逻辑卷分区，先删除逻辑卷，然后删除逻辑卷组，然后删除分区

```bash
sudo lvremove /dev/centos1/home
sudo vgremove centos1
sudo pvremove /dev/sdb2
#此时，pvs已经看不到分区的逻辑信息了
```

fio使用的参考链接
---

fio[单机测试](http://blog.sina.com.cn/s/blog_6b1ccd6501012pvl.html)

fio[参数说明](http://blog.csdn.net/yuesichiu/article/details/8722417)

iostat[命令详解](http://blog.csdn.net/zhangjay/article/details/6656771)

