---
title: "Ceph中EC的对齐"
date: 2015-11-23
tags: ceph
---

ErasureCode应用在存储中的是满足MDS性质的线性分组码，最常用的是Reed Solomon编码。 将一份数据切成K块数据块，然后编码成M块校验块，放到K+M个OSD上，可以容忍任意M块丢失。

Ceph的EC支持主要是Loïc Dachary实现的。具体的PG层逻辑，复用了ReplicatedPG，写数据前的准备工作，都是在主OSD完成。
编码的工作由对应的plugin完成，可以很灵活的实现自己的编码。Plugin的选用是在创建ecpool时的ecprofile中指定的，具体的设置可以参考[官网](http://docs.ceph.com/docs/master/rados/operations/erasure-code/)。

已有的Plugin是Jerasure，Intel ISA，LRC，SHEL，前两者是采用硬件加速的编码库，后两者是基于RS改进的编码方案，主要改进了修复的效率和占用的资源。

<!--more-->

目前Ceph不支持EC的偏移写，只能writefull，或者offset要满足是strip_width的倍数，且正好是原始对象的末尾，所以最终都会转换成一个AppendOp。

编码的流程

```C++
// ECTransaction::TransGenerator
void operator()(const ECTransaction::AppendOp &op)

// 每次编码的buf大小是stripe_width=K*chunk_size，循环编码
ECUtil::encode(sinfo, ecimpl, bl, want, &buffers);

// ECPlugin实现的编码过程
ecimpl->encode(want, buf, &encoded);
```

其中，chunk_size也是每个osd上要存的数据量，就是一次要对齐的大小，再来看chunk_size是怎么来的。

ECBackend中有这样一个变量，`ECUtil::stripe_info_t sinfo`，sinfo包含变量stripe_width。
sinfo是在ECBackend构造的时候实例化的。

实例化过程

OSDMonitor::prepare_new_pool -> prepare_pool_stripe_width

```c++
ErasureCodeInterfaceRef erasure_code;
// 实例化ecprofile定义的ECPlugin
get_erasure_code(erasure_code_profile, &erasure_code, ss);
// 用户期望的条带大小，读取配置文件的选项 osd_pool_erasure_code_stripe_width
uint32_t desired_stripe_width = g_conf->osd_pool_erasure_code_stripe_width;
// 调用ECPlugin的函数来获取条带大小
stripe_width = erasure_code->get_data_chunk_count() *
      erasure_code->get_chunk_size(desired_stripe_width);
```

不同的ECPlugin会有不同的条带大小，下面主要分析下Jerasure为例。get_chunk_size的对齐过程

* 调用`alignment = get_alignment()`返回ECPlugin实现的编码本身需要的对齐长度，就是一次矩阵运算需要的byte数

* 然后根据对齐长度，使object_size满足为alignment的整数倍

* 如果per_chunk_alignment为true，会使每个chunk大小都是alignment的整数倍。默认false

详细代码

```c++
ErasureCodeJerasure::get_chunk_size(int object_size)
{
  unsigned alignment = get_alignment();
  if (per_chunk_alignment) {
    unsigned chunk_size = object_size / k;
    if (object_size % k)
      chunk_size++;
    unsigned modulo = chunk_size % alignment;
    if (modulo) {
      chunk_size += alignment - modulo;
    }
    return chunk_size;
  } else {
    unsigned tail = object_size % alignment;
    unsigned padded_length = object_size + ( tail ?  ( alignment - tail ) : 0 );
    return padded_length / k;
  }
}
// Jerasure支持不同的编码方案，会有部分差别，这只是一个例子
unsigned ErasureCodeJerasureLiberation::get_alignment() const
{
  // k: data chunk数
  // w: 一次编码的字长，也是有限域运算的位长，默认为8。GF(2^8)上的运算可以查表
  // packetsize: 一次编码组的个数
  // sizeof(int): jerasure是把数据存在int中的，所以要对齐int
  unsigned alignment = k * w * packetsize * sizeof(int);
  // LARGEST_VECTOR_WORDSIZE 的作用未知
  if ( ((w*packetsize*sizeof(int))%LARGEST_VECTOR_WORDSIZE) )
    alignment = k*w*packetsize*LARGEST_VECTOR_WORDSIZE;
  return alignment;
}
```

总结，不同的编码本身会有不同的编码要求，跟踪get_chunk_size和get_alignment函数可以得到，了解对齐原理后合理设置参数，才能使编码性能提高。
每次编码都会prepare逻辑，object_size过小，会频繁prepare，导致性能下降。

