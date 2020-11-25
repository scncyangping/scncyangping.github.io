---
layout:     post
title:      "SpringBoot源码系列(二)SpringBoot启动流程概览"
subtitle:   ""
date:       2019-01-02 12:00:00
author:     "YaPi"
header-img: ""
tags:
    - SpringBoot
---

#### 启动

SpringBoot通过启动类，运行main方法启动主要语句为

```
public static void main(String[] args) {
    SpringApplication.run(Sb2Application.class, args);
}
```
run方法内部，主要有两个重要的事项

1. 调用相关配置，生成SpringApplication对象
2. 调用SpringApplication中的run方法

### 主要流程步骤

- 框架初始化
- 框架启动
- 自动化装配

#### 框架初始化

- 配置资源加载器(文件、资源的配置读取)
- 配置primarySources(就是启动传递过去的类，一般都是启动类)
- 应用环境检测(springBoot2会检测环境是一个web环境还是reactnative环境)
- 配置系统初始化器
- 配置应用监听器
- 配置main方法所在类

#### 启动框架
![avatar](https://blog-1257627424.cos.ap-chengdu.myqcloud.com/springBoot%E6%BA%90%E7%A0%81/springboot%E5%90%AF%E5%8A%A8%E6%B5%81%E7%A8%8B.jpg)

#### 自动化装配

- 收集配置文件中的配置工厂类
- 加载组建工厂
- 注册组建内定义bean