---
layout:     post
title:      "SpringBoot源码系列之Bean实例化解析"
subtitle:   ""
date:       2020-11-120 14:00:00
author:     "YaPi"
header-img: ""
tags:
    - SpringBoot
---

Bean的实例化主要在refresh流程的finishBeanFactoryInitialization方法中


Bean的实力化操作主要是对BeanDefinition的操作

什么是BeanDefinition呢？

1. 一个对象在Spring中描述，RootDefinition是其常见实现
2. 通过操作BeanDefinition来完成bean实力化和属性注入



#### 总结

1. 介绍下refresh方法流程
2. 介绍下bean实例化流程
3. 说几个bean实例化的扩展点及其作用