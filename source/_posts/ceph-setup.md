---
title: Ceph测试集群搭建
date: 2015-05-19
tags: ceph
---

这里整理一篇从源码编译的测试集群搭建方法。
ceph-deploy的部署方式，可以[参考官网](http://ceph.com/docs/master/start/quick-start-preflight/)。

<!--more-->

1.编译
---
[Ceph-compile]({% post_url 2015-02-23-ceph-compile %}#centos)

2.部署
---
* monitor
[ceph-deploy]({% post_url 2015-03-02-ceph-deploy %}#配置monitor)

其中，主要步骤

    * 配置文件相关：2，3，4，9
    * 集群准备：6，7，8
    * 启动：11

测试集群的可以省略密钥的配置。

* osd
[ceph-deploy]({% post_url 2015-03-02-ceph-deploy %}#添加osd)

> 先把monitor的配置文件同步过来后，OSD可以按简单模式配置，启动参照复杂里的[11]


可能遇到的问题
---

1.无法访问monitor
关闭防火墙，centos 6 是iptable，centos 7是firewall 
systemctl stop firewalld.service
注意，这只是临时关闭，重启后会重新启动。

2.sudo找不到命令
是因为sudo的path被重置了，如下方法修改
sudo visudo
Defaults env_reset 改成 Defaults !env_reset
在~/.bashrc 添加alias sudo='sudo env PATH=$PATH'

3.sudo ceph -s找不到python依赖
也是因为路径重置了，一劳永逸的方法，把ceph/src/pybind的内容复制到Python系统路径中。

sudo cp ~/ceph/src/pybind/* /usr/lib/python2.7/site-packages

最后，good luck！

ceph-deploy快速部署
---
主要参考[官网](http://ceph.com/docs/master/start/quick-ceph-deploy/)
这里记录出现的问题

* RuntimeError: NoSectionError: No section: 'ceph'

执行yum remove ceph-release，据说是版本不兼容

* RuntimeError: remote connection got closed, ensure ``requiretty`` is disabled for

进入要部署的机器，执行如下命令
sudo visudo
把`Defaults requiretty` 修改为 `Defaults:ceph !requiretty`

如果改完还么起作用，说明免密码的没有配，执行如下

```bash
echo "ceph  ALL = (root) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/ceph  
sudo chmod 0440 /etc/sudoers.d/ceph 
```

linux相关命令
---

* 磁盘调度策略
```
echo deadline > /sys/block/sda/queue/scheduler 

#调整完会运行cat命令，显示如下，表示选中deadline。
cat /sys/block/sda/queue/scheduler
noop [deadline] cfq
#因为参数是维护在内存中的，所以不能直接用vim修改，否则保存时会提示 E667:同步失败
```

* 添加用户
```
sudo useradd ceph   #添加用户
sudo passwd ceph    #设置密码
#添加sudo权限
visudo
#在下面仿照root，添加
ceph ALL=(ALL) ALL
```

* 格式化硬盘
