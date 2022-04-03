---
title: "Civetweb大并发超时分析"
date: 2019-06-12 15:38:45
tags: rgw
---

# 背景

Ceph RGW压测大并发时，由于底层性能不足，会造成服务端请求超时。如果在中间有ngxin做负载均衡时，压测工具Cosbench会显示有请求失败，而直连rgw则不会显示失败，为了分析这个问题，研究了下rgw请求超时过程中发生了什么。

而关于Cosbench请求是否失败的问题，其实比较简单，由于中间有nginx时，nginx在判断server超时时，会给客户端返回50x错误，直连rgw时，请求超时返回的是客户端错误，而cosbench对客户端错误只是打印了一行warn日志，不会报错，所以导致两种测试方式结果有差异，本质上都是服务端来不及处理客户端请求了。

<!-- more -->

# TCP状态

测试中，通过ss命令查看，某个中间状态下，服务端有如下状态值。
```
SYN-RECV 17
ESTAB 313
CLOSE-WAIT 136
TIME-WAIT 10
```

SYN-RECV 状态是建立TCP连接的3次握手中，client发出SYN包后，服务端接收后进入的状态；
ESTAB 是完成TCP 3次握手后的状态，到这都还不需要应用的参与；
CLOSE-WAIT 是关闭TCP连接中，被动关闭一方收到FIN包后的状态，在等待应用层回复FIN；
TIME-WAIT 是关闭TCP连接中，主动关闭连接一方在回完最后一个ACK包后等待两个MSL的状态；

SYN-RECV 状态是在所有队列满了后，连接会进入的状态。建立连接时，服务端会回复SYN+ACK包，但是不会完成连接建立，所以client会处于ESTABLISTED状态，并发送后续数据包。
服务端由于没有完成连接建立，后续所有包都会被忽略，client在发包没有收到ACK时，TCP层从间隔200ms开始依次翻倍不断重试。服务在忽略收包的同时，会间断的发送SYN+ACK，猜测是为了保持连接？当应用层能够处理时，会最终完成连接建立，并处理请求。
处于SYN-RECV状态的连接，如果client提前结束，会处于FIN-WAIT-1状态，因为sever端同样也不会处理FIN包。

# Civetweb 线程模型

## 1.主线程，master_thread
负责accept请求，生成socket
主要函数：

* master_thread_run
  * accept_new_connection
  * produce_socket

## 2.工作线程，worker_thread
负责消费请求，读取客户端数据，并回调注册的后端rgw处理，以及关闭socket。
新版本rgw配置为500，也就是有一个500线程池负责处理请求。
主要函数：
* worker_thread_run
  * consume_socket
  * process_new_connection
    * getreq
    * handle_request 
  * close_connection

# Civetweb 处理请求过程

Civetweb在处理请求时，包括自己和内核中由几个队列控制了请求的处理，底层超时会逐步导致各个队列满。

主线程中，accept_new_connection 调用`accept`接受请求，并copy socket到内部队列，队列长度为`MGSQLEN = 20`，如果队列已满，那么就会阻塞并等待信号量。在工作线程调用consume_socket后会发送信号量。

阻塞后，会导致很多请求建立了TCP连接，处于ESTAB状态，但是没有accept，内核中有独立的队列来保存这一类的请求，队列大小为`/proc/sys/net/core/somaxconn`和`listen(socket, backlog)`的backlog中取小的那个，默认为128。
超过128时，服务端TCP层将不会完成3次握手，这样的链接就会处于SYN-RECV状态。而SYN-RECV同样有内核队列保存，队列大小为`/proc/sys/net/ipv4/tcp_max_syn_backlog`，这个值会根据内存大小自动调整。

工作线程在process_new_connection中，收到空包后断开，进入close_connection，调用`shutdown`发送FIN包，处理关闭TCP链接的四次握手。

# 总结

根据上述过程，产生这些状态的过程比较清晰了
1）底层IO慢，导致rgw的处理线程会卡在等待IO返回上，worker_thead卡在process_new_connection中的handle_request；
2）服务端卡主后，客户端会提前关闭连接，向服务端发送FIN包，但是服务端由于还在handle_request中，需要等返回后才能到getreq接收到关闭请求来处理TCP连接的关闭，所以这时候服务端就处于CLOSE_WAIT状态，客户端会处于FIN-WAIT-2状态；
3）当服务端不断累积有线程卡主，那么会把rgw的线程池耗完，这时候civetweb的accept队列会达到20并把主线程卡主；
4）主线程卡主后，会累积established状态且没有accept的链接，直到队列达到128；
5）等待accept队列满了后，后续请求将不再建立连接，就会处于SYN-RECV状态；


参考：http://veithen.io/2014/01/01/how-tcp-backlog-works-in-linux.html
