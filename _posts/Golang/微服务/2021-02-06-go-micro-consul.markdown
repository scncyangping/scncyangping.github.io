---
layout:     post
title:      "服务发现及配置中心"
subtitle:   ""
date:       2021-06-010 14:01:00
author:     "YaPi"
header-img: ""
tags:
    - Go微服务
---

### 使用consul

#### 集群版consul架构

![avatar](https://blog-1257627424.cos.ap-chengdu.myqcloud.com/golang/consul%E9%9B%86%E7%BE%A4%E6%9E%B6%E6%9E%84.png)

多数据中心主要部分为两部分，server及client

Gossip: 流言协议

多server之间通讯(广域网 WAN Pool)使用 WAN GOSSIP协议进行通讯
多client之间通讯(局域网 LAN Pool)使用 LAN GOSSIP协议进行通讯
client到server端检测相关通讯也适用LAN Gossip协议

Gossip协议局域网：
- 让client自动发现Server节点
- 分布式故障检测在某几个Server机上执行
- 能够用来快速广播事件

Gossip协议广域网：
- WAN Pool全局唯一
- 不同数据中心的Server都会加入WAN Pool
- 允许服务器跨数据中心请求