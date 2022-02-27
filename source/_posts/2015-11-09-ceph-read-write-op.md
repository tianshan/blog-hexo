---
title: Ceph读写流程分析二
date: 2015-11-09
tags: ceph
---

[上一节]({% post_url 2015-06-08-ceph-readwrite %})分析了整体的读写流程，这节主要理一下OP出队过程中，主OSD上ReplicatePG->do_osd_op的过程。


读OP：OSD_OP_READ

![OSD_OP_READ](/images/2015-11-09-do_osd_op_read.png){:width="40%"}

写OP：OSD_OP_WRITE

![OSD_OP_WRITE](/images/2015-11-09-do_osd_op_write.png)