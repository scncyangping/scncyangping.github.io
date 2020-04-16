---
layout:     post
title:      "Go学习笔记 - (三)源码-初始化"
subtitle:   ""
date:       2020-04-15 03:40:00
author:     "YaPi"
header-img: ""
tags:
    - golang
---
#### 启动流程

- 系统初始化 os_linux.go -> osinit
    > 获取系统cpu信息等
- 调度器初始化方法 proc.go -> schedinit
    > 内存相关初始化(如span信息的初始化)、命令行参数和环境变量、垃圾回收等
- 创建一个G
- 调用runtime.mstart 分配一个m
- 用创建的G调用runtime.main