---
layout:     post
title:      "tinkerbell"
subtitle:   ""
date:       2022-02-16 16:01:00
author:     "YaPi"
header-img: ""
tags:
    - 云计算
---

### 概念
Tinkerbell是采用声明式配置和自动化方法管理基础架构和应用程序的工具，主要用来管理裸金属服务器。

#### 组件
Tinkerbell使用微服务的方式部署。内部可分为5个组件。

- Tink - 包含三个基础服务 tink-server（主要管控服务）、tink-worker(具体worker节点上的服务)、tink-cli(提供任务、工作流的命令行操作功能)
- Boots - DHCP服务。相应DHCP请求，分发IP，并且提供iPXE服务
- OSIE - 默认的裸金属内存安装环境，可安装操作系统
- Hegel - 元数据服务，从Tinkerbell和OSIE中收集数据，并转化为JSON数据
- Hook - OSIE（Operating System Installation Environment）的替代方案
- PBnj - 可选的服务，可与BMCs通信管理电源与启动配置

另还包含3个基础组件
- PostgreSQL - 存储硬件数据、模版、工作流等数据
- Image Repository - 镜像仓库
- NGINX - 在工作流执行过程中，提供引导文件和操作系统镜像

#### 架构

![avatar](https://blog-1257627424.cos.ap-chengdu.myqcloud.com/cloud/tinkerbell.png)

分为控制服务和worker服务。其中，控制服务包含Tinkerbell主要业务逻辑，如：DHCP和iPxe服务、元数组服务、工作流引擎。

```text
DHCP（动态主机配置协议）是一个局域网的网络协议。指的是由服务器控制一段IP地址范围，客户机登录服务器时就可以自动获得服务器分配的IP地址和子网掩码。
```

控制服务需满足的条件：

- CPU - 2vCPUs
- RAM - 4 GB
- Disk - 20 GB, this includes the OS
- 需要能运行DHCP服务

worker为裸金属节点即主要工作流执行节点。
worker节点需要满足的条件

- 支持iPXE启动
- RAM - 4 GB

```text
PXE(preboot execute environment，预启动执行环境)是由Intel公司开发的技术，工作于Client/Server的网络模式，支持工作站通过网络从远端服务器下载映像
并由此支持通过网络启动操作系统，在启动过程中，终端要求DHCP服务器分配IP地址，再用TFTP（trivial file transfer protocol）或MTFTP(multicast trivial file transfer protocol)
协议下载一个启动软件包到本机内存中执行，由这个启动软件包完成终端（客户端）基本软件设置，从而引导预先安装在服务器中的终端操作系统。

gPXE/iPXE是PXE的扩展版，支持HTTP协议，可以通过http、ISCSI SAN、network等方式启动。

iPXE由gPXE分支而来(fork)，功能更丰富。
```

#### 流程

控制节点操作：

1. 创建worker硬件数据，如：配置网络相关属性，ip地址、mac地址、pxe支持等
2. 配置任务，任务即在worker节点上的操作，比如：磁盘操作、操作系统安装等。这些任务被打包成docker镜像，在worker节点上pull过后执行。
3. 配置工作流，可以类比于ci过程中的pipeline，包含多个任务。依次执行。

worker节点

1. 当worker节点第一次启动时，它的硬盘驱动器中没有存储任何内容，引导加载程序将进入网络模式。此模式运行PXE，它会广播一个DHCP请求，请求“做某事”
   控制节点中的Boots是一个 DHCP 服务器，它处理DHCP请求并使用iPXE脚本回复包含操作系统安装环境的数据。
2. 在OSIE环境中，安装了tink-worker。tink-worker中会根据当前机器ip去tink-server查询配置的工作流，若查询到可执行的工作流，则执行相关任务完成初始化。

具体流程解析：

- 控制节点
控制节点是整个tinkerbell业务逻辑主要节点。在此节点运行的服务有：tink-server、tink-cli、Hegel、Boots服务。
其中tink-server管理创建工作流。tink-cli提供与用户交互的功能。如硬件信息录入、任务录入、工作流录入的基础功能。Hegel是元数据
服务，获取tinkerbell、OSIE的数据，并供其使用。Boots是DHCP服务及iPxe服务。

- worker节点
当worker启动的时候，引导进入ipxe模式，并广播DHCP消息，建立与Boots的联系，而后，通过iPxe提供给worker一个操作系统安装环境(OSIE)。
此环境默认是Apline Linux 3.7 包含一个kernel及ramdisk。并且，ramdisk包含了Docker环境，其中运行tink-worker服务，它会与tink-server建立联系并获取可以执行的工作流。

- 工作流

工作流及即worker节点需要执行的任务。每一个任务在tink-server被打包成一个docker镜像。在worker几点导入了OSIE过后，内部运行的tink-worker(docker服务)会
拉取需要执行的工作流。然后执行一些列任务。如：磁盘分区、实际操作系统安装等任务。在这个过程中，可以执行自定义的一些任务，达到可扩展操作的目的。工作流可以用户自定义，
实现用户自定义的一些基础设施创建的需求，比如安装一些监控任务、日志服务等。同时，整个OSIE环境也支持用户自定义，按照自己的需求去创建。


#### 参考

https://www.jianshu.com/p/7c8b44d03edc