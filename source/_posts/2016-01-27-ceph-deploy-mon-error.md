---
title: "Ceph编译部署启动失败"
date: 2016-01-27
---

Ceph编译安装经常会出现各种问题，这里记录下。

1.权限问题

默认ceph是以ceph用户启动的，可以查看`src/init-ceph`，导致在root用户下用ceph-deploy部署时，mon不能起来。    
还没部署前，可以直接修改ceph为root就可以了。已经部署失败后，因为service文件已经写好了，还要改一个地方。
`ls /etc/systemd/system/ceph-mon.target.wants/` 可以看到对应的文件为`/usr/lib/systemd/system/ceph-mon@.service`，所以修改该文件中setuser和setgroup后的用户即可。

2.selinux

需要关闭，否则可能会遇到`filestore(/var/lib/ceph/tmp/mnt.icGL7K) mkjournal error creating journal on /var/lib/ceph/tmp/mnt.icGL7K/journal: (13) Permission denied` 类似的错误

```sh
getenfoce       #查看当前设置
setenforce 0    #临时设置

永久设置，修改文件，然后重启
/etc/selinux/config
SELINUX=disabled
```

3.动态链接库找不到

默认是安装在`/usr/local/bin`，对应的库在`/usr/local/lib`。   
这种情况需要修改 `/etc/ld.so.conf`，添加 `include /usr/local/lib`，然后执行`ldconfig` 重新加载库。    
如果要临时设置，也可以`export LD_LIBRARY_PATH=/usr/local/lib`

4.python文件找不到

问题同上，默认安装在`/usr/local/`。在`.bashrc`中添加python搜索路径，`export PYTHONPATH=$PYTHONPATH:/usr/local/lib/python2.7/site-packages/`