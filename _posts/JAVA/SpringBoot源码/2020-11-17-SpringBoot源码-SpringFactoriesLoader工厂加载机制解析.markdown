---
layout:     post
title:      "SpringBoot源码系列之系统初始化器总结"
subtitle:   ""
date:       2020-11-17 14:00:00
author:     "YaPi"
header-img: ""
tags:
    - SpringBoot
---

#### 类型问题

1. 介绍下SpringFactoriesLoader ？
2. SpringFactoriesLoader如何加载工厂类 ？
通过加载指定路径下的文件，构建properties对象，并获取相应配置的对象的全类名，然后生成对应的对象。
然后根据配置的order设置执行的顺序。
3. 系统初始化器的作用？
它其实所以SpringBoot的一个回调接口，用来像SpringBoot容器中注册属性
4. 系统初始化器的调用时机？
是在SpringApplication的run方法中prepareContext()方法中执行的
5. 如何自定义实现？
三种方式
6. 自定义有哪些注意事项？
order大小的影响，即order失效的问题
