---
title: "Ceph 源码部署"
date: 2015-03-02
tags: ceph
---

源码的编译见[Ceph Compile](/2015-02-23-ceph-compile/)

Ceph 0.87， 系统ubuntu
参考[官方配置文档](http://ceph.com/docs/master/install/manual-deployment/)
集群需要至少1个monitor和2个OSD（对应两个副本）

集群的配置会按照如下的结构，node1作为monitor，node2和node3作为OSD节点。
![ceph-arch](/images/2015-03-02-ceph-arch.png)

<!--more-->

## 配置monitor

1.登录到monitor节点

```
ssh {hostname}
```

2.Ceph的默认配置目录是 `/etc/ceph` 。创建配置文件，默认是 `ceph.conf` ，其中ceph表示集群名

3.集群ID
    
```
#生成集群唯一ID
uuidgen
#把ID添加到配置文件
fsid={UUID}
```

4.添加初始monitor(s)和IP地址到配置文件

```
mon initial menbers = {hostname}[,{hostname}]
mon host = {ip-address}[,{ip-address}]
```

Note: 如果是IPv6，需要设置 `ms bind ipv6` 为 `true`，参考[Network Configuration Reference](http://ceph.com/docs/master/rados/configuration/network-config-ref)。

5.创建集群密钥环

```
#为monitor生成密钥
ceph-authtool --create-keyring /tmp/ceph.mon.keyring --gen-key -n mon. --cap mon 'allow *'
#生成 `client.admin` 用户并添加该用户到密钥环
sudo ceph-authtool --create-keyring /etc/ceph/ceph.client.admin.keyring --gen-key -n client.admin --set-uid=0 --cap mon 'allow *' --cap osd 'allow *' --cap mds 'allow'
#添加 `client.admin` 密钥到 `ceph.mon.keyring`
sudo ceph-authtool /tmp/ceph.mon.keyring --import-keyring /etc/ceph/ceph.client.admin.keyring
```

6.使用hostname，IP地址和FSID生成monitor映射。保存为 `/tmp/monmap`

```
monmaptool --create --add {hostname} {ip-address} --fsid {uuid} /tmp/monmap
#例如：
monmaptool --create --add ubuntu 192.168.230.128 --fsid 6d3c75f6-458b-4b29-b65f-e3083e7240db /tmp/monmap
```

7.在monitor上创建一个默认的数据目录

```
sudo mkdir -p /var/lib/ceph/mon/{cluster-name}-{hostname}
#例如：
sudo mkdir -p /var/lib/ceph/mon/ceph-ubuntu
```

详细配置见[Monitor Config Reference - Data](http://ceph.com/docs/master/rados/configuration/mon-config-ref#data)

8.将monitor映射和密钥环添加到monitor守护进程

```
sudo ceph-mon --mkfs -i {hostname} --monmap /tmp/monmap --keyring /tmp/ceph.mon.keyring
#例如
sudo ceph-mon --mkfs -i ubuntu --monmap /tmp/monmap --keyring /tmp/ceph.mon.keyring
#实测，上一步创建文件夹可以不做，这一步会默认在/var/lib/ceph/mon/下创建对应文件夹
```

9.配置模板

```
[global]
    fsid = {cluster-id}
    mon initial members = {hostname}[, {hostname}]
    mon host = {ip-address}[, {ip-address}]
    public network = {network}[, {network}]
    cluster network = {network}[, {network}]
    auth cluster required = cephx
    auth service required = cephx
    auth client required = cephx
    osd journal size = {n}
    filestore xattr use omap = true
    osd pool default size = {n}  # Write an object n times.
    osd pool default min size = {n} # Allow writing n copy in a degraded state.
    osd pool default pg num = {n}
    osd pool default pgp num = {n}
    osd crush chooseleaf type = {n}
```

根据前面的配置，得到：

```
[global]
    fsid = 6d3c75f6-458b-4b29-b65f-e3083e7240db
    mon initial members = ubuntu
    mon host = 192.168.230.128
    public network = 192.168.230.0/24
    auth cluster required = cephx
    auth service required = cephx
    auth client required = cephx
    osd journal size = 1024
    filestore xattr use omap = true
    osd pool default size = 2
    osd pool default min size = 1
    osd pool default pg num = 333
    osd pool default pgp num = 333
    osd crush chooseleaf type = 1
```

10.新建 `done` 文件

```
#标记monitor已经创建，准备好启动了。
sudo touch /var/lib/ceph/mon/ceph-ubuntu/done
```

11.启动monitor

* 在Ubuntu，使用Upstart

```
sudo start ceph-mon id=ubuntu

#要允许守护进程在每次重启后启动必须创建空文件，如下：
sudo touch /var/lib/ceph/mon/{cluster-name}-{hostname}/upstart
#例如：
sudo touch /var/lib/ceph/mon/ceph-ubuntu/upstart
```

>启动monitor的时候如果遇到 start: Unknown job: ceph-mon, 是因为使用 `make install` 安装时不会安装 [upstart script](http://ceph.com/docs/master/rados/operations/operating/) ， 可以手动将 `src/upstart` 中的脚本复制到 `/etc/init/` ，但是该方法有个问题，应为编译安装的目录是/usr/local/bin，但upstart中配置文件的启动目录是/usr/bin，所以需要手动修改ceph-mon.conf中两处代码， `pre-start' 中的test路径，和exec的执行路径。不知道有没有更好的方法。

* 对于 Debian/CentOs/RHEL，使用sysvinit：

```
sudo /etc/init.d/ceph -c /etc/ceph/ceph.conf start mon.{hostname}
```

注意
```
#centos下，也需要将ceph手动添加到启动项
sudo cp ceph/src/init-ceph /etc/init.d/ceph
#需要在mon数据目录添加sysvinit，在原文没有，不加的话会出现 mon.hostname not found
sudo touch /var/lib/ceph/mon/ceph-ubuntu/sysvinit
```
    
12.验证Ceph创建的默认池

```
ceph osd lspools
#输出如下：
0 rbd，
```

>* 如果出现 python的import xx找不到，需要把ceph/src/pybind包含到Python查找路径下，例如，在.bashrc中, export PYTHONPATH=$PYTHONPATH:~/ceph/src/pybind
* 如果出现 `OSError: librados.so.2` ，需要安装librados包(sudo apt-get install librados-dev)，详见[librados-intro](http://ceph.com/docs/master/rados/api/librados-intro/)
* 如果出现 `missing keyring, cannot use cephx for authentication` ，需要修改keyring的属性，详见[auth-config-ref](http://ceph.com/docs/master/rados/configuration/auth-config-ref/#keysS)

12.验证monitor正在运行

```
ceph -s
#输出如下：
cluster 6d3c75f6-458b-4b29-b65f-e3083e7240db
 health HEALTH_ERR 64 pgs stuck inactive; 64 pgs stuck unclean; no osds
 monmap e1: 1 mons at {ubuntu=192.168.230.128:6789/0}, election epoch 2, quorum 0 ubuntu
 osdmap e1: 0 osds: 0 up, 0 in
 pgmap v2: 64 pgs, 1 pools, 0 bytes data, 0 objects
        0 kB used, 0 kB / 0 kB avail
        64 creating
```

## 添加OSD

monitor启动后，就可以添加OSDs了。只有当集群有足够OSDs来处理object的副本时，集群才能达到 `active+clean`状态（例如，osd pool default size=2 需要至少两个OSDs）。 在monitor引导启动后，集群有了默认的CRUSH映射，然而，映射中还没有任何Ceph OSD 的守护进程映射到Ceph节点。

### [简单配置 ](#short-form)

Ceph提供了 `ceph-disk` 工具，可以处理磁盘、分区或者目录。该工具通过自增的索引来创建OSD ID。并且该工具会把新的OSD自动添加到主机的CRUSH映射。执行 `ceph-disk -h` 来获得命令的详细信息。工具会自动执行下面[复杂配置](#long-form)的流程。

1.准备OSD

```
ssh {node-name}

sudo ceph-disk prepare --cluster {cluster-name} --cluster-uuid {uuid} --fs-type {ext4|xfs|btrfs} {data-path} [{journal-path}]
#例如
sudo ceph-disk prepare --cluster ceph --cluster-uuid a7f64266-0894-4f1e-a635-d0aeaca0e993 --fs-type ext4 /dev/hdd1
```

2.激活OSD

```
sudo ceph-disk activate {data-path} [--activate-key {path}]
#例如
sudo ceph-disk activate /dev/hdd1
```
Note: 如果Ceph节点上没有 `/var/lib/ceph/bootstrop-osd/{cluster}.keyring` 需要添加参数 `--activate-key` 。

### [复杂配置 ](#long-form)

不利用工具的情况下，可以通过如下配置实现创建OSD，添加OSD到CRUSH映射。通过下面的过程可以更好的了解整个过程。分别登录node2和node3执行以下步骤。

1.登录到OSD主机

```
ssh {node-name}
```

2.为OSD生成UUID

```
uuidgen
```

3.创建OSD，如果不设置UUID，在OSD启动时会自动设定。该命令会输出OSD的编号，在下面的步骤中会用到。

```
ceph osd create [{uuid}]
```

4.在新的OSD上创建默认目录

```
sudo mkdir -p /var/lib/ceph/osd/ceph-{osd-number}
```

5.如果OSD是硬盘而不是系统，需要挂载到刚创建的目录

```
sudo mkfs -t {fstype} /dev/{hdd}
sudo mount -o user_xattr /dev/{hdd} /var/lib/ceph/osd/ceph-{osd-number}
#user_xattr 表示启用扩展的用户属性

#如果格式化为btrfs文件系统，且出现 `mkfs.btrfs: no such file or directory` ，需要安装btrfs，
sudo apt-get install btrfs-tools 
#如果使用btrfs文件系统，则在mount时不需要 user_xattr 选项，该项btrfs默认支持该选项，否则会出错。
```

6.初始化OSD的数据目录

```
ssh {new-osd-host}
sudo ceph-osd -i {osd-num} --mkfs --mkkey --osd-uuid [{uuid}]
```

在运行 `ceph-osd --mkkey` 之前，该目录必须是空的。并且，ceph-osd 工具需要用参数 `--cluster` 指定自定义的集群名。

注意

* OSD的大小要略于配置文件中的 `osd journal size` ，配置单位为MB。
* 如果因为配置错误，需要删除osd的挂载目录，该目录是带有只读属性的，可以用chattr修改


7.注册OSD的认证密钥。路径中 `ceph-{osd-num}` 的 `ceph` 值为 `$cluster-$id` 。如果集群名不是 `ceph` , 使用你对应的集群名。

```
sudo ceph auth add osd.{osd-num} osd 'allow *' mon 'allow profile osd' -i /var/lib/ceph/osd/ceph-{osd-num}/keyring
```

8.添加Ceph节点到CRUSH映射中

```
ceph osd crush add-bucket {hostname} host
```

9.把Ceph节点放到根节点 `default` 下

```
ceph osd crush move node1 root=default
```

10.把OSD添加到CRUSH映射，这样就可以开始接受数据了。 同样也可以反编译CRUSH的映射，把OSD添加到设备列表，把主机作为一个bucket(在它还没有加入CRUSH映射前），在主机中把设备当做项目添加，指定一个权重后重新编译并设置。

```
ceph osd crush add {id-or-name} {weight} [{bucket-type}={bucket-name} ...]
#例如
ceph osd crush add osd.0 1.0 host=node1
```

11.在添加一个OSD到Ceph后，OSD就在配置文件中了。但是还没有运行，OSD处于 `down` 和 `in` 状态。必选启动新的OSD才能开始接收数据。

* Ubuntu中，使用Upstart

```
sudo start ceph-osd id={osd-num}
#例如：
sudo start ceph-osd id=0
```

注意，和monitor一样，需要复制 `src/upstart` 下的 ceph-osd 到 `/etc/init/` ，同时需要修改文件中 `/usr/bin` 为 `/usr/local/bin`， `/usr/libexec` 为 `/usr/local/libexec` 。

* Debian/CentOS/RHEL中，使用sysvint

```
sudo /etc/init.d/ceph start osd.{osd-num}
#例如：
sudo /etc/init.d/ceph start osd.0

#注意：sysvint下需要创建如下的空文件
sudo touch /var/lib/ceph/osd/{cluster-name}-{osd-num}/sysvinit
#例如：
sudo touch /var/lib/ceph/osd/ceph-0/sysvinit
```

一旦启动了OSD，它就处于 `up` 和 `in` 状态。

总结
---

一旦有了monitor和两个OSD启动并运行，通过如下的执行可以查看placement groups节点。

```
ceph -w
#查看OSD树：   
ceph osd tree
#将会看到如下信息：
    # id    weight  type name   up/down reweight
    -1  2   root default
    -2  2       host ubuntu
    0   1           osd.0       up      1   
    1   1           osd.1       up      1
```

要添加或移除额外的monitors，详见[Add/Remove Monitors](http://ceph.com/docs/master/rados/operations/add-or-rm-mons)。要添加或移除额外的Ceph OSD守护进程，详见[Add/Remove OSDs](http://ceph.com/docs/master/rados/operations/add-or-rm-osds)。

注意，monitor和OSD的通信需要在防火墙添加例外，centos 6是iptables，centos 7是firewall。
