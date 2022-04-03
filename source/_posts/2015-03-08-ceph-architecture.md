---
title: "Ceph架构"
date: 2015-03-08
tags: ceph
---

翻译自官网，原文：<http://ceph.com/docs/master/architecture/>

Ceph独特的提供了统一了object， block，和file storage的系统。Ceph高度可靠，容易管理，以及免费。Ceph的优势可以转移公司的IT基础建设和你的能力到管理海量的数据中。Ceph提供额外的扩展性--成千的客户端访问PB和EB大小的数据。一个Ceph节点利用商业硬件和智能守护进程，一个Ceph存储集群可容纳大量的节点，这些节点互相通信来动态复制以及重分配数据。

![stack](/images/2015-03-08-stack.png)

<!--more-->

Ceph的存储集群
---

Ceph基于RADOS(Reliable Autonomic Distributed Object Store)提供了无限可扩展的存储集群，详细可以阅读[论文](http://ceph.com/papers/weil-rados-pdsw07.pdf)

一个Ceph存储集群包含两类守护进程

* [_Ceph Monitor_](http://ceph.com/docs/master/glossary/#term-ceph-monitor)
* [_Ceph OSD Daemon_](http://ceph.com/docs/master/glossary/#term-ceph-osd-daemon)

![daemons](/images/2015-03-08-daemons.png)

Ceph monitor维护了集群映射的主拷贝。Ceph的monitors集群通过监控守护进程的失效确保了集群的高可靠性。存储集群客户端通过montior检索集群映射的拷贝。
Ceph OSD 守护进程检查自身的状态以及其他OSDs的状态并报告给monitors。

存储集群客户端和每一个OSD守护进程使用CRUSH算法来高效的计算数据位置信息，而不是依赖一个中心查询表。Ceph's的高层特性包括基于 `librados` 提供给存储集群的一个本地接口，以及构建在 `librados` 上层的一系列服务接口。

### 存储数据

Ceph存储集群从客户端接收数据并存储为对象，不管对方是Ceph Block Device, Ceph Object Storage, Ceph Fileststem 或者是通过 `librados` 定制的实现。每一个对象对应文件系统中的一个文件存储在对象存储设备中(Object Storage Device)。Ceph OSD守护进程在存储磁盘上处理读写操作。

![storage-1](/images/2015-03-08-storage-1.png)

Ceph OSD守护进程在一个扁平的命名空间(没有层级目录)存储所有数据为对象。一个对象包括id，二进制数据，以及包含一组j键值对形式的元数据，元数据的语义完全依赖客户端。例如，CephFS利用元数据存储文件属性，文件所有者，创建时间，最后修改的日期等。

![storage-2](/images/2015-03-08-storage-2.png)

Note: 对象的ID在整个集群中都是唯一的，并不只是本地文件系统。

### 扩展性和高可靠性

在传统架构中，客户端需要和中心组件交互（例如网关，broker，API，facade等），中心组件往往是进入一个复杂子系统的唯一入口。这样的设定引入了单点故障，就限制了性能和可靠性。

Ceph去除了中心网关来使客户端能够和OSD守护进程直接交互。OSD守护进程在其他节点上创建对象副本来确保数据安全和高可靠性。Ceph同时使用了一个monitor集群来确保可靠性。为了实现去中心化，Ceph使用了CRUSH算法。

#### CRUSH介绍

Ceph客户端和OSD守护进程都使用CRUSH算法来高效的计算出对象位置信息，而不是依赖于一个中心的查询表。CRUSH使用了智能的数据副本来确保弹性，这点更适合超大规模的存储。接下来的章节提供了CRUSH如何运作的额外细节。CRUSH的详细讨论见[论文](http://ceph.com/papers/weil-crush-sc06.pdf).

#### 集群映射

Ceph需要客户端和OSD守护进程知道集群的拓扑，包括了5个映射集合，这就是 “集群映射”。

**1.The Monitor Map:** 包括集群的 `fsid` ，位置，名字地址，和每一个monitor端口。同时指示了映射的创建时间，和最后的修改时间。查看monitor的映射，通过命令 `ceph mon dump` 。

**2.The OSD Map:** 包括集群 `fdis` ，映射创建时间和修改时间，池的列表，副本大小，PG个数，OSDs的列表以及他们的状态（例如 `up` ， `in` ）。查看OSD映射，执行 `ceph osd dump` 。

**3.The PG Map:** 包括PG的版本，它的时间戳，上一个OSD周期，每个pg的full ratios和细节，比如PG ID，Up Set，PG的状态（例如 active+clean），和每个池子的数据使用率。

**4.The CRUSH Map:** 包含一个存储设备的列表，失效域的层级（例如，设备，主机，机架，行，房间等），以及存储数据时遍历层级的规则。查看一个CRUSH的映射，执行 `ceph osd getcrushmap -o {filename}` ；然后通过执行 `crushtool -d {comp-crushmap-filename} -o {decomp-crushmap-filename}` 反编译。可以在文本编辑器或者 `cat` 命令查看反编译映射。

**5.The MDS Map:** 包含当前MDS映射周期，映射的创建时间，最后的修改时间。 同时包含了存储员数据的池子，元数据服务器的列表，以及哪些元数据服务器是 `up` 和 `in` 状态的。 查看MDS映射，执行 `ceph mds map` 。

每一个映射包含了一个操作状态修改的迭代历史。Ceph Monitor维护了一个集群映射的主拷贝，包括集群成员，状态，修改，以及整体存储集群的健康状况。

#### 高可靠性Monitors

在客户端能够读写数据前，需要先联系Monitor来获得集群映射的最近拷贝。Ceph的存储集群在有一个monitor就可以工作；然而，这会带来了单点故障（例如，如果monitor宕机，客户端就不能读写数据了）

为了增加可靠性和容错性，Ceph支持monitors集群。在monitors集群中，延迟和其他错误会导致一个或者更多的monitors的状态落后于集群。因此，Ceph必须在大量monitor实例中达成一致，不管集群的状态。Ceph总是使用一个主要的monitor（例如，1,2:3,4:5,4:6，等）和[pacos](http://en.wikipedia.org/wiki/Paxos_(computer_science))算法就集群当前状态在monitors之间达成一致。

monitors配置的详细说明见 [Monitor Config Reference](http://ceph.com/docs/master/rados/configuration/mon-config-ref)

#### 高可靠性验证

为了区分用户以及抵御中间人攻击，Ceph提供 `Cephx` 验证系统俩授权用户和守护进程。

> 注意，Cephx协议并不在传输过程或者其他对数据加密（例如，SSL/TLS）。

Ceph使用共享密钥来授权，意味着客户端和monitor集群都会有客户端的密钥。如此的验证协议就是双方都能给对方证明拥有密钥的拷贝而不用真的展示。这创建了共同的授权，集群确定用户拥有密钥，并且用户确定集群拥有密钥的拷贝。

Ceph密钥的可伸缩性使得Ceph对象存储不需要一个中心接口，这意味着Ceph客户端需要和OSDs直接交互。为了保护数据，Ceph提供 `cephx` 验证系统，用来给用户操作客户端授权。 `cephx` 协议的操作方法类似 [Kerberos](http://en.wikipedia.org/wiki/Kerberos_(protocol))。

用户或者执行者调用Ceph客户端来和monitor通信。不同于Kerberos，每一个monitor都可以授权用户和分发密钥，所以不存在单点故障或者瓶颈。monitor返回一个类似于Kerberos ticket的授权数据结构，包含一个回话钥匙用来获得Ceph服务。这个会话钥匙本身使用用户的永久密钥加密，所以只有对应的用户才能向Ceph monitor(s)请求服务。客户端然后使用会话钥匙来向monitor请求服务，monitor会提供ticket给客户端，授权客户端访问OSDs处理数据的权限。Ceph monitors和OSDs共享一个密钥，所以客户端可以使用monitor提供的ticket访问集群中的任意OSD或者元数据服务器。类似于Kerberos， `cephx` 的tickets会过期，所以攻击者不能偷偷的使用过期的ticket或者会话钥匙。这种形式的授权可以防止攻击者通过访问通信媒介用其他用户的id伪造假消息或者改变其他用户正常的消息，只要用户的密钥不在过期前泄露。

要使用 `cephx` ，管理员必须先设定用户。在下面的途中， `client.admin` 从命令行调用 `ceph auth get-or-create-key` 来生成用户名和密钥。Ceph的 `auth` 子系统生成用户名和密钥，存储一份到monitor(s)并发回给 `client.admin` 用户。这意味着客户端和monitor共享一个密钥。

> 注意， `client.admin` 必须用安全的方法把用户ID和密钥发送给用户。

![auth-1](/images/2015-03-08-auth-1.png)

要获得monitor授权，客户端必须发送用户名给monitor，monitor生成会话钥匙并使用用户名对应的密钥加密。然后，monitor发送加密后的ticket给client。客户端然后用共享密钥解密传输数据来获得会话密钥。会话密钥标识了当前会话的用户。客户端然后请求一个用会话密钥签名代表用户的ticket。monitor生成ticket，用用户的密钥加密然后回传给客户端。客户端解密ticket然后使用它签名访问集群的OSDs和元数据服务器。

![auth-2](/images/2015-03-08-auth-2.png)

`cephx` 协议授权通信在客户端机器和Ceph服务器之间。每一条消息的发送在客户端和服务器之间，然后初始化授权，用ticket签名后，monitors、OSDs和元数据服务器就可以验证共享密钥了。

![auth-3](/images/2015-03-08-auth-3.png)

授权提供的Ceph客户端和服务器主机之间的协议。授权不会超出Ceph客户端之外。如果用户从远程主机访问Ceph客户端，Ceph的授权不会应用到用户主机到客户端主机之间。

详细的配置，见[Ceph Config Guide](http://ceph.com/docs/master/rados/configuration/auth-config-ref)。用户管理的详细配置，见[User Mangement](http://ceph.com/docs/master/rados/operations/user-management)。

#### 支撑超大规模的智能守护进程

在许多集群架构中，集群成员的主要目的是提供中心化的接口，就是节点可以访问。这类中心化的接口通过双重调度向客户端提供服务--这就会在PB到EB规模时造成了巨大的瓶颈。

Ceph去除了这样的瓶颈：Ceph的OSD守护进程和客户端是集群感知的。类似于Ceph客户端，每一个OSD守护进程知道集群中其他ISD守护进程。这使得OSD守护进程可以和其他OSD守护进程和monitors直接交互。除此以外，这使得客户端能够直接和OSD守护进程交互。

Ceph客户端，monitor，OSD守护进程能够相互交互，意味着OSD守护进程可以利用Ceph节点的CPU和RAM很容易的执行那些使中心服务器负担的任务。计算能力的利用会带来如下几点益处：

1.OSDs直接服务客户端：因为网络设备可支持的连接数是有限的，一个中心化的系统在大规模系统中有较低的物理上限。通过使客户端和OSD守护进程直连，Ceph同时增加了性能和整个系统的容量，并移除了单点故障。当需要时，客户端可以和OSD守护进程保持一个会话，而不是中心服务器。

2.OSD成员和状态：OSD守护进程加入一个集群并报告状态。在最底层，OSD守护进程状态为 `up` 或者 `down` 反映是否在运行并且能够服务客户端。如果OSD守护进程状态是 `down` 和 `in` ，这个状态表示OSD守护进程已经失效。如果OSD守护进程不在运行（例如宕机了），OSD不能通知Monitor它 `down` 了。monitor可以周期性的ping OSD守护进程来确定他是运行状态。然而，Ceph同样授权OSD守护进程来决定相邻的OSD守护进程是否是 `down` 来更新集群映射并报告给monitor(s)。这意味着monitors可以保持为轻量级的处理方法。额外的细节见[Monitoring OSDs](http://ceph.com/docs/master/rados/operations/monitoring-osd-pg/#monitoring-osds)和[Heartbeats](http://ceph.com/docs/master/rados/configuration/mon-osd-interaction)。

3.数据清洗：为了维护数据干净和一致性，OSD守护进程可以在pg内清洗对象。OSD守护进程可以在pg内和其他OSD的对象副本比较元数据。清洗方法（通常一天一次）捕获bugs或者文件系统错误。OSD守护进程同样通过逐位比较对象中数据执行更深层的清洗。深层清洗（通常一周一次）会发现轻清洗不能发现的硬盘坏扇区。清洗的详细配置见[Data Scrubbing](http://ceph.com/docs/master/rados/configuration/osd-config-ref#scrubbing)

4.副本：类似于客户端，OSD守护进程使用CRUSH算法，但是OSD用它来计算对象的副本该存在什么地方（重平衡也会用到）。在典型的写场景，客户端使用CRUSH算法来计算存放对象的位置，映射对象到pool和pg，然后查看CRUSH映射来发现pg的主OSD。

客户端把对象写到对应pg的主OSD。然后，主OSD在自己的CRUSH映射拷贝中发现第二和第三OSDs来放置副本，然后复制对象到第二第三OSDs（额外副本数）中对应的pg，在确认数据成功存储后（注，这里的存储成功指成功存储到内存缓冲区）回应给客户端。

![data-1](/images/2015-03-08-data-1.png)

OSD守护进程实现数据复制后，减轻了客户端的职责，同时确保了数据可用性和数据安全。

### 动态集群管理

在[扩展性和高可用性](#扩展性和高可靠性)章节，我们解释了Ceph如何使用CRUSH，集群感知和智能的守护进程保证了扩展性和高可用性。Ceph设计的关键是自治，自恢复，以及智能的OSD守护进程。我们来更深层次的看下CRUSH是如何工作来使现代云存储设施放置数据，重平衡集群以及从失效中动态恢复。

#### 关于Pools

Ceph的存储系统支持 `Pools` 的概念，这是存储数据的逻辑分区。

Ceph客户端从monitor检索[Cluster Map](#集群映射)，并写对象到pools。pool的大小或者副本的个数，CRUSH规则集以及pgs的数量会决定Ceph如何放置数据。

![management-1](/images/2015-03-08-management-1.png)

Pools至少设置需要如下参数：

* 对象的所有权或者访问权

* PG的数量

* 要使用的CRUSH规则集

详细见[Set Pool Values](http://ceph.com/docs/master/rados/operations/pools#set-pool-values)

#### 映射PGs到OSDs

每个pool有一定数量的pg。CRUSH直接映射PGs到OSDs。当客户端存储数据时，CRUSH映射每一个对象到pg3。

映射对象到pg在OSD守护进程和客户端之间创建了一个间接层。Ceph存储集群必须能够在存数据时动态的扩大（或者收缩）以及重平衡。如果客户端知道某个OSD守护进程所拥有的对象，这就在客户端和OSD守护进程创建了一个紧密的耦合。相反，CRUSH算法映射每一个对象到pg然后映射每一个pg到一个或者多个OSD守护进程。这一层费直接联接允许Ceph在新OSD守护进程上线或者失效OSD守护进程恢复时动态重平衡。下面的图描述了CRUSH如何映射对象到pgs，以及pgs到OSDs。

![map-1](/images/2015-03-08-map-1.png)

有了集群映射和CRUSH算法，客户端可以在读写数据时精确计算出需要使用的OSD。

#### 计算PG的ID

当一个客户端绑定到monitor，他会检索最新的[集群映射](#集群映射)。有了集群映射后，客户端就知道了集群中所有的monitors，OSDs，和元数据服务器。然而，它不知道人换个关于对象位置的信息。

> 对象的位置是通过计算得到的。

客户端唯一需要的输入是对象ID和pool。很简单：Ceph存储数据在有命名的pool中（例如，"liverpool"）。当客户端想要存储有命名的对象（例如: "john", "
paul", "george", "ringo" 等），它会用对象名的哈希值，pool中的PGs数，pool名字俩计算pg。客户端使用如下步骤计算：

1.客户端输入pool ID和对象ID（例如，pool="liverpool", object-id="john"）。
2.Ceph拿到对象ID然后计算哈希值。
3.用哈希值取余PGs的数量（例如，58）然后得到PG ID。
4.Ceph用pool获得pool ID（例如，"liverpool"=4）。
5.Ceph连接pool ID到PG ID前面（例如，4.58）

计算对象位置远快于在会话中查询对象位置。CRUSH算法允许客户端来计算对象要存的位置，并且允许客户端连接主OSD来存储或者检索对象。

#### Peering and sets

在前面的章节中，我们注意到OSD守护进程互相检查心跳然后报告给monitor。OSD守护进程做的另外一个是 'peering' ，该方法使所有存储Placement Group(PG)的OSDs对PG中所有对象（以及他们的元数据）的状态达成一致。事实上，OSD守护进程[报告对等失败](http://ceph.com/docs/master/rados/configuration/mon-osd-interaction#osds-report-peering-failure)给monitors。对等问题通常是自己解决；然而，如果这个问题仍然存在，可以参考[Troubleshooting Peering Failure](http://ceph.com/docs/master/rados/troubleshooting/troubleshooting-pg#placement-group-down-peering-failure)章节。

> 注意：状态达成一致并不意味着PGs有最新的内容。

存储集群被设计为至少存储对象最新的两份拷贝（也就是size=2），这是数据安全的最小需求。为了高可用性，存储集群应该存储更多的副本（例如， `size=3` 和 `min size=2`），所以当集群处于维护数据安全的退化状态时任能够运行。

回过来看[支撑超大规模的智能守护进程](#支撑超大规模的智能守护进程)中的图表，我们并不具体的命名每个OSD守护进程（例如，osd.0,osd.1等)，而更多的是成做主要的，第二的，以及第四的。按照惯例，主要的是活动集中的第一个，负责在作为主要的地方为每个pg协调对等进程，以及作为唯一OSD接收客户端初始的写对象到主要placememt group的请求。

当一系列的OSDs负责一个pg时，我们把他们称为活动集(Acting Set)。一个活动集会设计到当前负责pg的OSD守护进程，或者在一些周期负责特定pg的OSD守护进程。

活动集中部分的OSD守护进程可能并不总是 `up` 。当活动集中某个OSD处于 `up` ，他就属于 `up` 集。 `up` 集一个重要的区分，因为当OSD失效时，Ceph可以重新映射PGs到其他OSD守护进程。

> 注意：PG的活动集中包含 `osd.25`, `osd.32`, `osd.61`，第一个OSD， `osd.25` 就是主要的。如果哪个OSD失效了，第二的OSD， `osd.32` ，就变成了主要的， `osd.25` 就会从Up集中移除。

#### 重平衡

当添加一个OSD守护进程到存储集群时，集群映射就会用新的OSD更新。回看[计算PG IDs](#计算PG的ID)，这会改变集群映射。因此，它改变了对象放置，因为它改变了计算的输入。下面的图描述了重平衡过程（尽管非常粗糙，因为它在大集群中基本影响很小），一些但并不是所有PGs从已有的OSDs（OSD 1和OSD 2）迁移到新的OSD（OSD 3）。就算在重平衡中，Ceph也是稳定的。许多pg仍然是原来配置，每一个OSD获得一些新的容量，所以在重平衡结束后新OSD不会出现负载峰值。

![rebalance-1](/images/2015-03-08-rebalance-1.png)

#### 数据一致性

作为维护数据一致性和干净的一部分，Ceph同样可以在pg内清洗对象。也就是说，OSDs可以在一个pg内和其他OSDs的pg存储的副本比较对象元数据。清洗（通常是一天一次）捕获OSD的bugs和文件系统错误。OSD守护进程同样通过逐位比较对象中数据执行更深层的清洗。深层清洗（通常一周一次）会发现轻清洗不能发现的硬盘坏扇区。

清洗的详细配置见[Data Scrubbing](http://ceph.com/docs/master/rados/configuration/osd-config-ref#scrubbing)。

### 纠删码

使用纠删码的pool存储每个对象为 `K+M` 块。分为K个数据块和M个编码块。该pool被配置为K+M大小，所以每个块都存在活动集中的一个OSD。块的排列存储为对象的属性。

例如，使用纠删码的pool被创建有5个OSDs（K+M=5），支持两块丢失（M=2）。

#### 读写编码块

当包含 `ABCDEFGHI` 的对象 __NYAN__ 写到pool时，纠删码函数简单的分割内容到3个块：第一块包含 `ABC` ，第二块包含 `DEF` 和最后一块 `GHI` 。如果内容不是K的倍数，会添加内容直到满足。该函数同时创建了两个编码块：第四块 `YXY` 和第五块 `GQC` 。每一块都存储到活动集的OSD中。这些块会存到名字（ __NYAN__）相同的对象中，但是在不同的OSDs上。除了名字，块的创建顺序也必须保留，并存储在对象的属性中（ `shard_t` )。块1 包含 `ABC` ，存储在OSD5上，而块4包含 `YXY` 存储在OSD3上。

![erasure-coding-1](/images/2015-03-08-erasure-coding-1.png)

当对象NYAN从纠删编码的pool中读数据时，解码函数读取三个块，块1 包含 `ABC` ，块3包含 `GHI` ，块4包含 `YXY` 。然后它重建了原始对象 `ABCDEFGHI` 。解码函数得知块2和块5丢失了（成为被纠删了）。块5不能读取是因为OSD4失效了。在三块读取后，解码函数就可以被调用了：OSD2最慢所以该块就没有计算在内。

![erasure-coding-2](/images/2015-03-08-erasure-coding-2.png)

#### 完全中断写

在擦出编码pool中，up集中的主OSD接收所有的写操作。它负责编码有小腹在的的数据为 K+M 块，然后发送到其他OSDs。它同时负责维护pg日志的权威版本。

在下面的图中，一个纠删码pg被创建为 K=2，M=1 共3个OSDs，2个放K，1个放M。pg的活动集由 __OSD 1__， __OSD 2__ 和 __OSD 3__ 组成。一个对象被编码存储到OSDs上：块D1v1（数据块 1，版本 1）在 __OSD 1__ 上，D2v1在 __OSD 2__ 上，C1v1（编码块 1，版本 1）在 __OSD 3__ 上。每一个OSD的pg的日志是完全一样的(1,2 为周期1，版本1)。

![erasure-coding-3](/images/2015-03-08-erasure-coding-3.png)

OSD 1是主要的，并且接收客户端的完全写入（WEITE FULL），这意味着有效负载是用来替换对象而不是覆写一部分。对象的第二版本（v2）是用来覆盖版本1（v1）。OSD 1把有效数据编码为3块：D1v2（数据块1，版本2）会在OSD1，D2v2在OSD2，C1v2（编码块1版本2）在OSD3.每一块会发送到目标OSD，包括用来存储块并且处理些操作以及维护placementgroup权威版本的主要OSD。当OSD收到写块消息时，它同时创建了pg的新记录来表示这个修改。例如，OSD 3存好C1v2后，就添加了记录 1,2（周期1，版本2）到日志。因为OSDs是异步工作的，所以会出现一些块还没有写入（例如D2v2)，而其他的已经确认写入磁盘了（例如C1v1和D1v1)

![erasure-coding-4](/images/2015-03-08-erasure-coding-4.png)

如果所有操作顺利，活动集中每一个OSD都会确认写入块，并日志中的 `last_complete` 指针会从 1,1 移到 1,2。

![erasure-coding-5](/images/2015-03-08-erasure-coding-5.png)

最终，对象的用于存储块的文件的前一个版本可以删除了：OSD1上的D1v1，OSD2上的C1v1，OSD3上的C1v1。

![erasure-coding-6](/images/2015-03-08-erasure-coding-6.png)

但是以外发生了。如果D2v2还没写入时OSD1失效了，对象的版本2只写了一部分：OSD3有1个块，但是不够恢复。它就会丢失两个块：D1v2和D2v2，纠删码参数K=2,M=1需要至少2个块来重建第三块。OSD4成为了新的主要的，发现日志记录 `last_complete` （也就是，这条记录之前的所有对象是可以在之前的活动集的OSDs获得的）是1,1，然后这会变成新的权威日志的head。

![erasure-coding-7](/images/2015-03-08-erasure-coding-7.png)

OSD3上发现的日志记录 1,2 和OSD4上新的权威日志有分歧：然后他就被抛弃了，包含块C1v2块的文件就被删除了。然后D1v1块会在数据清理时通过解码函数重建并存储新的主OSD 4。

额外的细节见[Erasure Code Notes](https://github.com/ceph/ceph/blob/40059e12af88267d0da67d8fd8d9cd81244d8f93/doc/dev/osd_internals/erasure_coding/developer_notes.rst)

### 缓存层

缓存层可以给客户端提供后端存储层的数据子集更好的I/O表现。缓存层包含相对快速，昂贵存储设备（例如，ssd）上创建的pool，配置用来当做缓存层，以及一个后端pool，不管是纠删编码的或者是相对缓慢、便宜的设备配置为商业存储层。Ceph对象处理处理放置对象的位置，层级代理决定向缓存还是后端存储写入对象。所以，缓存层和后端存储层对客户端来说都是透明的。

![cache-1](/images/2015-03-08-cache-1.png)
额外细节见[Cache Tiering](http://ceph.com/docs/master/rados/operations/cache-tiering)

### 扩展Ceph

可以通过创建共享对象类 `Ceph class` 来扩展Ceph。Ceph直接加载存储在 `osd class dir` 目录中的 `.so` 类（例如， 默认是$libdir/rados-classes）。当你实现一个类后，可以创建能够调用存储集群的本地方法的新的对象方法，或者其他其他类库或者你自己创建的其他类方法。

在写的时候，Ceph类可以调用本地或者类方法，可以对未归纳数据（inbound data）执行任何系列操作，然后生成Ceph的原子级应用的结果写入事务。

在读的时候，Ceph类可以调用本地或者类方法，可以对出站数据（outbound data）执行系列操作然后返回数据到客户端。

> __Ceph类样例__
> 内容管理系统的呈现特定大小和纵横比图片的Ceph类，可以读取归类的位图图像，裁剪到特定纵横比，改变大小，并嵌入不可见的版权或者水印俩帮助保护知识版权；然后保存结果位图到对象存储。

可以通过 `src/objclass/objclass.h` ， `src/fooclass.cc` 和 `src/barclass` 查看样例实现。

### 总结

Ceph存储集群就像充满活力的有机体。然而，许多存储应用并没有完全应用典型CPU和RAM，但是Ceph做到了。从心跳，到对等，到重平衡集群或者从失效中恢复，Ceph减轻了客户端的责任（以及解除了中心化的网关，所以Ceph架构并不需要），并使用了OSDs的计算能力执行任务。参考[Hardware Recommendations](http://ceph.com/docs/master/install/hardware-recommendations)和[Network Config Reference](http://ceph.com/docs/master/rados/configuration/network-config-ref)来理解Ceph如何利用计算资源的概念。

Ceph 协议
---

Ceph客户端使用本地协议来和存储集群交互。Ceph把这些功能打包到了 `librados` 库，所以可以创建你自己的客户端。下面的图描述了基本的架构。

![protocol-1](/images/2015-03-08-protocol-1.png)

### 本地协议和 LIBRADOS

现代应用需要一个有异步通信能力的简单对象存储接口。Ceph存储集群就提供了这样的能力。接口提供了在整个集群中直接，并行访问对象的能力。

* Pool操作
* 快照和写入时复制的克隆
* 读写对象-创建或删除-整个对象或比特范围-追加或截断
* 创建/设置/获取/删除 扩展属性(XATTRs)
* 创建/设置/获取/删除 键值对
* 复合操作或双重确认语义
* 对象类

### 对象观察/通知

客户端可以对一个对象注册持久关注并保持和主OSD会话的打开。客户端可以发送一个通知消息并通过有效数据传输给所有观察者，在观察者收到通知后会收到回传通知。这使得客户端能够使用任何对象的异步通信通道。

![object-1](/images/2015-03-08-object-1.png)

### 数据分段

存储设备有吞吐量限制，这影响了性能和扩展性。所以，存储系统通常支持[分段](http://en.wikipedia.org/wiki/Data_striping)-向多个存储设备存储信息的连续片段-来增加吞吐量和性能。数据分片最常见的形式是[RAID](http://en.wikipedia.org/wiki/RAID)。和Ceph的分段最类似的是[RAID 0](http://en.wikipedia.org/wiki/RAID_0#RAID_0)，或者 'striped volume' 。Ceph的分段提供了RAID 0分段的吞吐量，n-RAID镜像的可靠性和更快的恢复性。

Ceph提供三类客户端：Ceph块设备，Ceph文件系统，Ceph对象存储。Ceph客户端把数据从它提供给用户的描述模式（一个块设备图，RESTful对象，CephFS文件系统目录）转换为用来存储到Ceph存储系统的对象。

>  __提示：__ Ceph存储系统存储忽的对象并不是分段的。Ceph对象存储，Ceph块设备和Ceph文件系统在存储集群上多个对象上分段数据。客户端通过 `librados` 直接写到存储集群时必须执行分段（和并行I/O）来获得这些益处。

Ceph分段对简单的格式是分段数为1的对象。客户端写分段单元直到对象达到最大的容量，然后创建新的对象在存储额外的分段数据。最简单的分段形式可能是足够多的小块设备，S3、Swift对象和CephFS文件。然而，这种最简单的形式美柚最大化的利用Ceph跨pg分发数据的能力，因此并没有很好的改进性能。下面的图描述了分段最简单的形式。

![striping-1](/images/2015-03-08-striping-1.png)

如果需要使用大的印象，大的S3或者Swift对象（例如视频），或者大的CephFS目录，你可以看到通过在一个对象集中多个对象的分段客户端数据带来的客观的读写性能改进。当客户端并行写分段单元到对象会得到客观的写性能。自从对象映射到不同的pg以及进一步映射到不同的OSDs，每一次写都以最大的写速度并行发生。单个磁盘的写会被磁头的移动（例如，每次6ms的寻道）和单个设备的带宽（例如，100MB/s)所限制。通过在多个对象间（映射到了不同的pg和OSDs）传播写操作，Ceph可以减少每个硬盘的寻道次数并合并多个硬盘来获得更快的写（或读）速度。

> 注意：分段和对象副本是独立的。CRUSH通过OSDs实现副本，分段会自动复制副本。

在下面的图中，客户端数据通过一个包含4个对象的对象集分段（图中的 `object set 1` ），其中第一个分段单元是 `strip unit 0` 位于 `object 0` ，第4块分段单元 `stripe unit 3` 位于 `object 3` 。在写完这4个分段后，客户端决定对象集是否满了。如果对象集没有满，客户端开始再次写分段到第一个对象（下途中的 `object 0` )。如果对象集满了，客户端创建一个新的对象集（ 下图中的 `object 2` )，然后开始在新对象集中的第一个对象（下图中的 `object 4` ）写第一个分段（ `stripe unit 16` ）。

![striping-2](/images/2015-03-08-striping-2.png)

3个变量决定了Ceph如何分段数据：

* Object Size: 存储集群中的对象有一个最大配置大小（例如，2MB，4MB等）。对象的大小要足够大能够容纳许多分段单元，而且应该是分段单元大小的倍数。
* Stripe Width: 分段会有配置的单元大小（例如，64kb）。客户端会把要写入对象的数据划分为同样大小的分段单元，除了最后一个分段单元。分段的宽度，需要是对象大小的分数，这样一个对象才能包含许多分段单元。
* Stripe Count: 客户端连续的写分段单元到一系列的对象，这一系列对象个数由stripe count决定。这一系列的对象称为对象集。在客户端写完对象集的最后一个对象后，它会返回对象集中的第一个对象。

> 重要：在把集群投入生产前测试你的分段配置性能。在分段好数据并写入对象后，就不能改变分段参数了。

一旦客户端划分数据到段并映射分段单元到对象后，Ceph的CRUSH算法会在对象以文件形式存储到磁盘前，把对象映射到pg，以及映射pg到OSD守护进程。

> 注意：因为客户端写到单个pool中，所有的数据分段到的对象映射到同一个pool中的pg。所以他们使用同一个CRUSH映射和相同的访问控制。

Ceph 客户端
---

Ceph客户端包括一系列服务接口，包括：

* 块设备：[Ceph块设备](http://ceph.com/docs/master/glossary/#term-ceph-block-device)（又名RBD）服务提供可变大小，精简配置的块设备，拥有快照和克隆功能。Ceph分段集群中的块设备来获得高性能。Ceph支持内核对象（KO）和直接使用 `librdb` 的QEMU虚拟机-避免内核对象出现在虚拟系统上层。
* 对象存储：[Ceph对象存储](http://ceph.com/docs/master/glossary/#term-ceph-object-storage)（又名，RGW）服务提供RESTful接口和Amazon S3和OpenStack Swift兼容。
* 文件系统：[Ceph 文件系统](http://ceph.com/docs/master/glossary/#term-ceph-filesystemhttp://ceph.com/docs/master/glossary/#term-ceph-filesystem)（CephFS）服务提供POSIX兼容的文件系统，可以使用 mount 作为文件系统或者在用户空间（FUSE）。

Ceph可以运行额外的OSDs，MDSs和Monitors实例来扩展和获得高可用性。下图描述了上层架构。

![client-1](/images/2015-03-08-client-1.png)

### Ceph对象存储

Ceph的对象存储守护进程， `radisgw` ，是一个FastCGI服务，提供RESTful HTTP API来存储对象和元数据。它有自己的数据格式，位于Ceph存储集群的顶层，并维护自己的用户数据库，授权和访问控制。RADOS网关使用统一的命名空间，这意味着，你可以使用OpenStack Swift兼容的API或者Amazon S3兼容的API。例如，你可以在一个应用用S3兼容的API写数据然后在另一个应用用Swift兼容的API来读数据。

> S3/Swift对象和存储集群对象比较
Ceph的对象存储使用术语 _对象_ 来描述存储的数据。S3和Swift对象和Ceph写到存储集群的对象并不一样。Ceph对象存储的对象是映射到Ceph存储集群的对象。S3和Swift对象并不需要和存储集群存储的对象以1:1的方式符合。S3或者Swift的对象映射到多个Ceph对象也是可以的。

详见[Ceph对象存储](http://ceph.com/docs/master/radosgw/)

### Ceph块设备

Ceph块设备在存储系统多个对象间分段块设备镜像，每个对象映射到pg并分发，pg在整个集群中独立的 `ceph-osd` 守护进程之间传播。

> 分段技术可以使RBD块设备比单个服务器表现更好！

精简配置且可快照的块设备对虚拟化和云计算来说是吸引人的选项。在虚拟机场景，人们通常在Qemu/KVM部署Ceph块设备到有rbd的网络存储驱动，主机机器使用 `librbd` 来提供块设备服务给客户。许多云计算服务商使用 `libvirt` 来融合虚拟管理系统。相比其他解决方案，你可以使用精简配置的Ceph块设备和Qemu， `libvirt` 来提供OpenStack和CloudStack服务。

虽然我们当前的 `librbd` 不支持其他的虚拟管理系统，你同样可以使用Ceph块设备内核对象来提供块给设备。其他虚拟技术例如Xen可以访问Ceph的块设备内核对象。这些可以通过命令行工具 `rbd` 。

### Ceph文件系统

在基于对象的Ceph存储集群的上层，Ceph文件系统（CephFS）提供POSIX兼容的文件系统来作为服务。CephFS文件映射Ceph存储集群存储的到对象。Ceph客户端把Ceph文件系统作为内核对象或者用户空间的文件系统（FUSE）挂载。

![filesystem-1](/images/2015-03-08-filesystem-1.png)

Ceph文件系统服务包括存储集群部署的Ceph元数据服务器（MDS）。MDS的目的是存储所有的文件系统元数据（目录，文件拥有者，访问模式等）到高可靠性的元数据服务器，其上的元数据都驻留在内存。MDS（称为 `ceph-mds` ）存在的原因是简单文件系统操作像列出目录或者改变目录（ls，cd）没有必要加重OSD守护进程的负担。所以从数据意义中分离元数据意味着Ceph文件系统可以提供高性能服务而不用涉及存储集群。

CephFS从数据中分离元数据，存储元数据到MDS，并存储文件数据到存储集群的一个或多个对象。Ceph文件系统致力于POSIX兼容性。Ceph-mds可以运行为单个进程。或者他可以分发到多个物理机器，同样是为了高可用性和扩展性。

* 高可用性：额外的 `ceph-mds` 实例可以作为后备，准备接管任何失效 `ceph-mds` 的职责。这很容易，因为所有的数据，包括日志卷，都保存在RADOS。这类转移会通过 `ceph-mon` 自动触发。
* 扩展性：多个 `ceph-mon` 实例可以处于活跃状态，他们会把目录树分割为子树（并为繁忙的目录分片），高效的平衡所有在线服务的负载。

后备和活跃状态的组合是可能的，例如运行3个活跃的 `ceph-mds` 实例用来扩展，一个后备实例保障高可用性。