---
title: "Ceph RGW multipart 上传简析"
date: 2016-07-11
tags: ceph
---

最近研究了跟了Ceph的RGW S3协议的multipart的一个[bug15886](http://tracker.ceph.com/issues/15886)，顺着bug分析了下多块上传的流程。问题主要是多块上传中，如果有相同upload id的重传，就有可能丢数据，目前H版和J版都已修复，并在s3-tests的master分支添加了对应的测试用例`test_multipart_resend_first_finishes_last`。


Multipart是客户端发起的一个分块上传的协议。首先是一个POST消息，请求upload id，然后就是每一块的PUT上传，PUT过程类似普通块的上传。
如下图，是multipart的put流程图。出错逻辑主要是函数`add_written_obj`，他的作用是，记录正在写的对象，如果出错，那么会在销毁processor时清除该对象。问题在于原来函数调用位置在`handle_obj_data`，也就是写对象之前，导致对象重复时，实际没有写入，但把原来正确的对象清除了。修正后在`throttle_data`中调用，改在对象写成功后，就不会有问题了。

测试的用例是，先写A，然后以重传形式写B，然后complet B，再完成A。这时实际对象内容应该是A。在B写入时，发现upload id相同的分片存在，就会重新生成临时upload id，保证不重复，所以B也就是一个临时对象，最后complete的A为实际对象内容。

![Ceph RGW Multipart 流程图](/images/2016-07-11-ceph-rgw-multipart.png)