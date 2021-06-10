---
layout:     post
title:      "gRpc及Protobuf"
subtitle:   ""
date:       2021-02-05 14:01:00
author:     "YaPi"
header-img: ""
tags:
    - Go微服务
---

### 概念介绍

#### RPC

- RPC代指远程过程调用(Remote Procedure Call)
- 包含了传输协议和编码(对象序列号)协议
- 允许运行于一台计算机的程序调用另一台计算机的子程序(比如：程序代码调用一个Redis)
- 简单、高效

#### gRpc 和 ProtoBuff

- gRpc是一个高性能、开源、通用的RPC框架
- 基于HTTP2.0协议标准设计开发
- 支持多语言，默认采用Protocol Buffers数据序列化协议

什么是Protocol Buffers?

- 是一种轻便高效的序列化结构化数据的协议
- 通常用在存储数据和需要远程数据通信的程序上
- 跨语言、更小(相对于json、xml等)、更快也更简单

使用其好处

- 加速站点之间数据传输速度
- 解决数据传输不规范问题(比如定义json的时候，可以随便定义，但是ProtoBuff会有限制)

常用概念

- Message： 描述了一个请求或响应的消息格式
  > singular: 表示成员有0个或者一个，一般省略不写
  > repeated: 表示该字段可以包含0-N个元素，比如go语言中的数组、切片等
- 字段标识：消息定义中，每个字段都有一个唯一的数值
- 常用数据类型：double、float、int32/64、bool、string、bytes
- service服务定义：在service中可以定义个RPC服务接口


#### 实例

```text
// 版本号
syntax = "proto3"

// 包名
package go.test.product;

service Product{
    // 定义的服务
    rpc AddProduct(ProductInfo) returns (ResponseProduct){}
}

// 驼峰命名
// 数字 1、2是字段标识
message ProductInfo{
    int64 id=1;
    string product_name = 2;
}

message ResponseProduct {
    int64 product_id = 1;
    
}
```

安装protobuf后生成相关文件
```text
mac os :

brew install protoc

go get -u github.com/golang/protobuf/proto
go get -u github.com/golang/protobuf/protoc-gen-go
go get github.com/micro/micro/v2/cmd/protoc-gen-micro

若执行的时候报找不到文件：

找到protoc-gen-go 文件，执行以下命令，该文件通过go get安装时会安装在$GOPATH/go/bin目录下
cp protoc-gen-go /usr/local/bin/
然后vim ~/.bash_profile
export GOPATH=$HOME/go PATH=$PATH:$GOPATH/bin
```