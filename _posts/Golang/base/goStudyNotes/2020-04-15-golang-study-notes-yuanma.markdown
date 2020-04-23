---
layout:     post
title:      "Go学习笔记(二)源码-内存分配回收"
subtitle:   ""
date:       2020-04-15 02:40:00
author:     "YaPi"
header-img: ""
tags:
    - golang
---

#### 内存分配

Go的内存分配基于tcmalloc模型

核心目标

- 从 mmap 申请⼤大块内存，⾃主管理，减少系统调⽤用。
- 基于块的内存复⽤用体系，加快内存分配和回收操作。

分配器以页为单位向操作系统申请大块内存。这些大块内存由n个地址连续的页组成
并用名为span的对象进行管理

当需要时，span 所管理内存被切分成多个⼤大⼩小相等的⼩小块，每个⼩小块可存储⼀一个对象,故称作 object
分配器以32kb为界，将对象分为大小两种，

大对象直接找一个⼤小合适的span。小对象则以8的倍数分为不同⼤小等级(size class)。
⽐如 class1 为 8 字节，可存储 1 ~ 8 字节⼤大⼩小的对象


##### 三级部件

- heap: 全局根对象，负责向操作系统申请内存，管理由垃圾回收器收回的空闲span内存块
- central: 从heap获取空闲span,并按需要将其分成object块。heap管理多个central
对象，每个central负责处理一种等级的内存分配需求，对应Object的不同等级
- cache: 运行期，每个cache都与某个具体线程相绑定，实现无锁内存分配操作。其内部有个
以等级为序号的数组，持有多个切分好的span对象。缺少空间时，向对应的central获取新的span即可

分配流程：

- 通过 size class 反查表计算待分配对象等级
- 从 cache.alloc[sizeclass] 找到等级相同的 span
- 从 span 切分好的链表中提取可用 object
- 如 span 没剩余空间，则从 heap.central[sizeclass] 找到对应 central，获取 span
- 如 central 没可⽤用 span，则向 heap 申请，并切割成所需等级的 object 链表
- 如 heap 也没有多余 span，那么就向操作系统申请新的内存


回收流程：

- 垃圾回收器或其他⾏行为引发内存回收操作
- 将可回收 object 交还给所属 span
- 将 span 交给对应 central 管理，以便某个 cache 重新获取
- 如 span 内存全部收回，那么将其返还给 heap，以便被重新切分复⽤用
- 垃圾回收器定期扫描 heap 所管理的空闲 spans，释放超期不⽤用的物理内存

从 heap 申请和回收 span 的过程中，分配器会尝试合并地址相邻的 span 块，以形成更大内存块，减少碎⽚

cache从central获取的数据，实际上是一个完整的span,这个span被central分割
成了Object链表。

内存优化平衡策略

1. 一次平衡。cache将未使用的object会收到central。供其他cache使用
2. 二次平衡。central将闲置的span回收到heap。供其他central使用

